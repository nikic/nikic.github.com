---
layout: post
title: Design issues in LLVM IR
excerpt: "Discusses a number of design issues related to IR canonicality: Pointer element types, type-based GEP, and constant expressions."
---
On the whole, LLVM has a well-designed intermediate representation (IR), which is specified in the [language reference](https://llvm.org/docs/LangRef.html). However, there are a number of areas where design mistakes have been made. And while the LLVM project is generally open to addressing such issues, mistakes in core IR design tend to be firmly embedded in the code base, making them hard to fix in practice. This blog post discusses some of the problems.

## Canonicalization

In the middle-end, we commonly view transformations not as optimizations, but as canonicalizations. There are many different ways to write the same program, and the purpose of target-independent middle-end transforms is to reduce them to a single form. Of course, the chosen form will often (but not always) coincide with the more efficient form. (This does not apply to *all* transforms, for example runtime unrolling and vectorization are certainly not canonicalization transforms.)

Why do we care about canonicalization? The main reason is that it reduces the number of permutations that other passes need to deal with. If `1 + a` is canonicalized to `a + 1`, everything else only needs to handle the latter form. Another reason is that it improves the effectiveness of redundancy-elimination transforms (like common subexpression elimination and global value numbering). Let's look at an example:

```llvm
define i1 @test(i32 %x) {
  %y = sub i32 %x, 1
  %z = add i32 %x, -1
  %c = icmp eq i32 %y, %z
  ret i1 %c
}
```

Instruction combining will canonicalize `x - 1` to `x + (-1)`:

```llvm
define i1 @test(i32 %x) {
  %y = add i32 %x, -1
  %z = add i32 %x, -1
  %c = icmp eq i32 %y, %z
  ret i1 %c
}
```

At this point it becomes trivial to detect that `%y` and `%z` are in fact the same, so we can replace `%z` with `%y`:

```llvm
define i1 @test(i32 %x) {
  %y = add i32 %x, -1
  %c = icmp eq i32 %y, %y
  ret i1 %c
}
```

Once that is done, we can also see that the equality comparison is trivially true:

```llvm
define i1 @test(i32 %x) {
  ret i1 true
}
```

So, what does this have to do with IR design? A good IR design can make certain redundancies non-existent by design. Primarily, this is "just" a matter of not encoding any unnecessary information.

For example, early LLVM versions distinguished between signed and unsigned integers, while later versions only have a single integer type (for a given bitwidth) and signedness is only encoded in the few places it matters: comparisons, extensions and overflow flags. This means that there is only one representation of `x + 1` regardless of whether `x` is signed or not. This is an outcome of the core IR design, and does not require additional canonicalization.

You will notice that the problems discussed in the following are quire similar in flavor.

## Pointer element types

Currently, LLVM pointer types look a lot like C pointer types: `i8*` is a pointer to an 8-bit integer. Let's copy around an `i8*` pointer:

```llvm
define void @copy.i8ptr(i8** %dst, i8** %src) {
  %val = load i8*, i8** %src
  store i8* %val, i8** %dest
  ret void
}
```

Now let's copy an `i32*` pointer instead:

```llvm
define void @copy.i32ptr(i32** %dst, i32** %src) {
  %val = load i32*, i32** %src
  store i32* %val, i32** %dest
  ret void
}
```

The astute reader will notice that these two pieces of code do exactly the same thing. They copy a pointer-sized value from one location to another, and the fact that this happens to be a pointer to `i8` or `i32` is wholly irrelevant. If we detect this fact and want to merge these functions by implementing one in terms of the other, we have to insert `bitcast`s:

```llvm
define void @copy.i32ptr(i32** %dst, i32** %src) {
  %dst.i8ptr = bitcast i32** %dst to i8*
  %src.i8ptr = bitcast i32** %src to i8*
  call void @copy.i8ptr(i8** %dst.i8ptr, i8** %src.i8ptr)
  ret void
}
```

Bitcasts are LLVMs way of representing a cast that reinterprets a value in different type, without changing its binary representation. These pointer bitcasts are only needed because LLVM encodes an unnecessary pointer element type.

Pointer type conversions are pervasive and often occur when data goes through different layers. A pointer may start out as `[8 x i32]*` when representing a specific object of known size, then become `[0 x i32]*` when interpreted as a slice, then `i8*` when passed to a `memcpy` as a generic pointer.

Thankfully, LLVM is in the process of removing pointer element types and switching to [opaque pointers](https://llvm.org/docs/OpaquePointers.html). With opaque pointers, the above code would like this:

```llvm
define void @copy.ptr(ptr %dst, ptr %src) {
  %val = load ptr, ptr %src
  store ptr %val, ptr %dest
  ret void
}
```

This renders both functions equivalent by-design, and removes any need for pointer bitcasts. An interesting question to ask here is: Can't we go one step further, and just treat a pointer as an integer? Say `i64` on a platform with 64-bit address space:

```llvm
define void @copy.i64(i64 %dst, i64 %src) {
  %val = load i64, i64 %src
  store i64 %val, i64 %dest
  ret void
}
```

This is not possible for two main reasons. The first is that pointers carry provenance, which is important for [alias analysis](https://llvm.org/docs/LangRef.html#pointer-aliasing-rules). A pointer has an "underlying object" it is based on, and dereferencing it is only valid if it points into that object. Integers do not track provenance, and as such conversions between integers and pointers are not considered no-ops (and are not represented as bitcasts).

The second is that there are different kinds of pointers: Even with opaque pointers, LLVM still distinguishes pointers in different address spaces. While these would be the same on a von Neumann architecture, some targets may have separate address spaces for data and instructions, which might be represented as `ptr` and `ptr addrspace(1)`. Pointers may even be [non-integral](https://llvm.org/docs/LangRef.html#non-integral-pointer-type), in which case they don't have an integer representation at all.

The migration to opaque pointers is already "in progress" for many years, though that progress has been rather slow and has only recently [picked up again](https://groups.google.com/g/llvm-dev/c/OFH6AStSaWQ/m/CD5r_28YBAAJ). The state of opaque pointers in LLVM itself is actually fairly okay, because there is a general awareness of the issue, and any optimization that tries to make use of pointer element types will not pass review. However, the state in clang is, diplomatically speaking, less than stellar. I especially like this [cute comment](https://github.com/llvm/llvm-project/blob/13a8aa3ee15a67048deeb193fe8b86005fdf9d10/clang/lib/CodeGen/Address.h#L49-L50) -- "simply" indeed.

## Type-based GetElementPtr

LLVM performs address calculations using the [`getelementptr` instruction](https://llvm.org/docs/LangRef.html#i-getelementptr), which accepts a base pointer and a number of indices:

```llvm
%struct = type { i32, { i32, { [2 x i32] } } }

define i32* @test(%struct* %base) {
  %ptr = getelementptr %struct, %struct* %base, i64 1, i32 1, i32 1, i32 0, i64 1
  ret i32* %ptr
}
```

The first `i64` means that we take the second element from the pointer, akin to `&base[1]` in C. The following indices drill down to a specific element in the nested struct and array structure. Assuming that `i32` is 4-byte aligned, if you add up all the offsets, it turns out that this `getelementptr` is equivalent to the following one:

```llvm
%struct = type { i32, { i32, { [2 x i32] } } }

define i32* @test(%struct* %base) {
  %base.i8 = bitcast %struct* %base to i8*
  %ptr = getelementptr i8, i8* %base.i8, i64 11
  ret i32* %ptr
}
```

These two GEPs compute the same address in two different ways, which adds up to another canonicalization problem. It also means that analyses usually need to decompose type-based GEPs into offset-based calculations. This is surprisingly expensive, because it requires computing the store size of all involved types from data layout information, including potential alignment constraints. It is not atypical for a couple percent of total compile-time to be spent in nothing but GEP offset calculations.

The better alternative is to make GEPs directly offset based, which works best in conjunction with opaque pointers:

```llvm
define ptr @test(ptr %base) {
  %ptr = getelementptr ptr %base.i8, i64 11
  ret ptr %ptr
}
```

What is not entirely clear is how variable indices should be encoded. Either the necessary scale could be part of the GEP instruction:

```llvm
define ptr @test(ptr %base, i64 %index) {
  %ptr = getelementptr ptr %base.i8, 4 * %index
  ret ptr %ptr
}
```

Or else the index calculation could be explicit in IR:

```llvm
define ptr @test(ptr %base, i64 %index) {
  %offset = shl i64 %offset, 2
  %ptr = getelementptr ptr %base.i8, i64 %offset
  ret ptr %ptr
}
```

The latter variant is once again beneficial from a canonicalization perspective, as there is no choice between an explicit offset calculation and an offset calculation embedded in the GEP. However, `inbounds` GEPs carry additional constraints on the offset calculation, which would be lost with such an encoding.

GEP instructions are not the only kind of instruction afflicted by too much type information. For example, the [`alloca` instruction](https://llvm.org/docs/LangRef.html#alloca-instruction), which is used to reserve stack space, accepts a type, while it really only needs to know the number of bytes to reserve:

```llvm
%ptr = alloca [8 x i32], align 4
; should be
%ptr = alloca i64 32, align 4
```

However, canonicality is not particularly important for allocas in practice, as they are typically not subject to redundancy elimination or structural equality checks.

## Constant Expressions

Next to instructions, LLVM also supports [constant expressions](https://llvm.org/docs/LangRef.html#constant-expressions). The `add` instruction in this function

```llvm
define i64 @test() {
  %res = add i64 1, 2
  ret i64 %res
}
```

can be equivalently written using a constant expression:

```llvm
define i64 @test() {
  ret i64 add (i64 1, i64 2)
}
```

A nice thing about constant expressions is that they are maximally folded and uniqued at time of construction. So this example will actually be converted into

```llvm
define i64 @test() {
  ret i64 3
}
```

when the file is parsed. Isn't this great for canonicalization? By construction, we can't have different representations of the same constant expression, because it is always fully folded!

Unfortunately, things aren't that simple. Constant expressions are unique per LLVM context, which is not [data layout](https://llvm.org/docs/LangRef.html#data-layout) aware. This means that at time of construction, only folds that do not depend on data layout can be performed.

For example, it's not possible to fold an expression like this:

```llvm
define i64 @sizeof_int64() {
  ret i64 ptrtoint (i64* getelementptr (i64, i64* null, i64 1) to i64)
}
```

This expression is the encoding for the store size of `i64`, and requires knowing the ABI alignment of `i64`, which is provided by the data layout, and not known by the constant expression itself.

This means that there are two levels of constant folding: One that is target-independent and happens on construction, and one that is target-dependent and performed by certain passes like instruction combining. This second folding needs to recursively walk all referenced constant expressions and attempt to fold them at all levels, essentially defeating the benefits of the "always folded" representation.

It goes beyond that, however. Instructions and constant expressions typically have the same constraints to permit uniform treatment, which means that we can't enforce IR validity constraints that depend on data layout. For example, the `ptrtoint` instruction/expression allows the integer type to have a different size than the pointer type and performs an implicit truncation or zero extension if they don't match. Such IR does not occur in practice, because it will be canonicalized to an explicit `trunc` or `zext`, but all passes still need to deal with the possibility. We can't actually enforce that the sizes are the same, because constant expressions don't have data layout.

However, data layout is only part of the problem. The other issue is the simple fact that operations can be represented either using an instruction or a constant expression. This means that code generally needs to be able to deal with both representations, to avoid pessimizing optimizations if constant folding (into an expression) has occurred. LLVM is generally fairly good about doing this, thanks to many generic pattern matchers that treat instructions and constant expressions transparently.

One significant complication is that a lot of code assumes that performing an operation on a constant is "free". For example, an optimization that pushes a negation through an instruction sequence might assume that pushing a negation into a constant operand is free, because it will just turn `1` into `-1`, or similar. Of course, this is not true for constant expressions, where the negation may not be folded, and you'll be left with a `sub (i64 0, i64 ...)`.

This is one of the most common sources of infinite loops in the optimizer. A transform pushes an operation into a constant, on the assumption that it will get folded. Thanks to constant expressions, it doesn't. A different transform may see the new constant expression and perform the reverse change, resulting in an infinite combine loop.

Constant expressions also have pathological cases in IR printing:

```llvm
define i64 @test(i64 %x1) {
  %x2 = mul i64 %x1, %x1
  %x3 = mul i64 %x2, %x2
  %x4 = mul i64 %x3, %x3
  %x5 = mul i64 %x4, %x4
  ret i64 %x5
}
```

Written in instructions, this is a nice linear chain, where each instruction is used twice. Using constant expressions, the same also holds for the in-memory representation. However, printed out as textual IR, there is no way to reuse a constant sub-expression that appears multiple times, so the root value `%x1` will end up being repeated `2^4` times. It is easy to construct IR that cannot be printed in any reasonable amount of time.

Finally, as divisions can occur inside constant expressions, it's possible for a constant materialization to trigger undefined behavior. This has odd implications: For example, an instruction may be rendered non-speculatable due to a "constant" operand. This will essentially never occur in practice, but still needs to be considered and accounted for.

So what is the alternative here? Having data layout available during construction (even if the resulting constant is data layout independent) would resolve a part of the problem. The remainder would require removing the concept of constant expressions, or rather reducing their scope.

Constant expressions do exist for a reason: global initializers, which only accept constants. However, while these initializers can hold arbitrary constant expressions on the LLVM IR level, only limited "relocatable" expressions will be accepted during lowering. The constant expression mechanism is only really needed for such relocatable expressions. LLVM's mistake was to combine the representation of these relocatable expressions together with the constant folding machinery.

## Closing thoughts

The design issues discussed here are evident with the benefit of hindsight. Of course, there tend to be sensible historical reasons why these choices were made in the first place. For example, having pointer element types and type-based GEPs makes it convenient to implement a front-end that lowers directly to LLVM IR -- in other words, clang. It means that the task of keeping track of types is delegated to LLVM. Similarly, I believe that the constant expression design dates back to a time where LLVM still supported a concept of target-independent modules without an associated data layout.

The issues covered in this post are related to IR canonicality. There is a separate class of issues related to correctness. The biggest of these is the concept of [undef values](https://llvm.org/docs/LangRef.html#undefined-values), which represent a quantum superposition state of all possible bit patterns. Yes, this is exactly as bad as it sounds. Thankfully, undef values are in the process of being replaced by [poison values](https://llvm.org/docs/LangRef.html#poison-values). But all this is a discussion for another time.