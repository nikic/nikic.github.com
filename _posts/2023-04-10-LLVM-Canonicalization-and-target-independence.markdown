---
layout: post
title: "LLVM: Canonicalization and target-independence"
excerpt: This documents LLVM's design philosophy when it comes to target-dependence in the middle-end optimizer.
---
Conceptually, the backend of an optimizing compiler is supposed to be the part that performs CPU architecture specific optimizations, while the middle-end is target-independent. In reality, the middle-end does need to take certain properties of the target into account. However, these need to be carefully managed to maintain the overall architecture of the compiler.

In this article, I want to discuss in which ways target-specific information can affect the LLVM middle-end, and more importantly, in which ways it is *not* allowed to affect it.

Data layout
-----------

Every IR module has an associated [data layout][data_layout], which specifies information about the size and alignment of primitive types, as well as other basic information. This determines whether the target is little-endian or big-endian, whether pointers are 32-bit or 64-bit, whether doubles have 4 byte or 8 byte alignment, etc.

Historically, LLVM used to allow modules without a data layout, which would in theory be completely target-independent. This turned out to be not particularly useful, as target-dependent aspects will make it into the IR anyway, for example due to ABI differences. Nowadays, the data layout is always required, and the memory layout of all structures is known.

Relying on the data layout in the middle-end is always fine. This is the one bit of target-dependence that is approved for use everywhere, and is automatically available in most places in the compiler.

Target library info
-------------------

TargetLibraryInfo (TLI) provided by the TargetLibraryAnalysis determines which library calls are assumed to be available on the target. This part is less about the target CPU architecture and more about the target OS and environment.

For example, if you are targeting `x86_64-unknown-linux-gnu` (and not using `-ffreestanding` or similar), LLVM will assume that certain symbols are provided by glibc.

When targeting `linux-gnu`, a call to `strrchr()` on a known-length string will be converted to `memrchr()`, while the same doesn't happen when targeting `windows-msvc`. That's because `strrchr()` is a standard C function, while `memrchr()` is a GNU extension.

Using TLI in the middle-end is generally also fine. This is something that needs to be explicitly requested from the pass manager, and many passes don't actually need this, but there are generally no concerns with adding additional TLI dependencies.

Target transform info
---------------------

TargetTransformInfo (TTI) provided by the TargetIRAnalysis provides a wide range of hooks that can be overridden by backends. This includes both generic cost modelling APIs (what is the reverse throughput of this instruction?) and specialized APIs for specific passes (what are the unrolling preferences for this target?)

Usage of TTI in the middle-end is tightly controlled. We make a distinction between target-independent canonicalization passes that are not allowed to use TTI, and cost model driven passes that can use TTI.

Examples of cost model driven passes are LoopVectorize, SLPVectorize or VectorCombine. This is pretty obvious, because vectorization is fundamentally based around a cost model. Vectorization is only profitable if we can show that the vector instructions are cheaper than the scalar instructions.

Canonicalization passes
-----------------------

Canonicalization passes like InstCombine are intended to produce IR in a "canonical form" that other passes can depend on. This canonical form is chosen based on how amenable it is to further analysis and transforms, not how beneficial it is for a particular target. The canonical form can be undesirable for some or even all targets, because what is convenient for analysis is not always what is convenient for CPUs.

In cases where the canonical form is undesirable for a target, a backend undo transform needs to be implemented. The undo transform will usually live either in CodeGenPrepare (CGP), DAGCombine, or some target-specific code. CGP is used for undo transforms that need to work across basic blocks, because this is not possible in DAGCombine.

A common question here is why we go through the trouble of making InstCombine convert IR into one form, and then make the backend undo it. Wouldn't it be simpler if we allowed backends to opt-out of the transform in the first place?

Consider two equivalent code patterns A and B. If we have an InstCombine transform from A to B, but the target prefers A, we could suppress the transform. However, this would do nothing about the case where the code is in form B in the first place. As such, we need a reverse transform from B to A.

We could implement both a transform in one direction and one in the other direction in InstCombine, and pick them depending on target. However, this means that there are now two different forms of the same pattern that other transforms need to deal with. We also need to consider more combinations of different patterns when it comes to ensuring that there are no infinite transform loops.

The better choice is to decide on one direction in InstCombine, so that there is a single canonical pattern the middle-end needs to deal with, and move the undo transform into the backend.

Another example of a canonicalization pass is LICM. LICM will move all loop-invariant code into the loop preheader. This is not always profitable, because it may increase register pressure, or unconditionally execute code that was previously conditional.

One could imagine an alternative LICM implementation that would take into account register pressure via TTI and code hotness using PGO information. However, that would mean that other transforms cannot rely on loop invariant operands actually being loop invariant, and this is not an easy property to determine once memory operations are involved.

LLVM makes the choice to make LICM a canonicalization transform, which may later be undone by other passes. LoopSink in the late middle-end optimization pipeline is PGO-based, while MachineSink is a backend pass.

Of course, our policies on TTI dependence of passes are just a means to an end, and it's sometimes necessary to make tradeoffs in favor of pragmatism. For example, we need to distinguish between using TTI for profitability queries and legality queries.

While legality queries would preferably be answered by the data layout, the truth is that adding a new TTI hook is much simpler than encoding additional information in the data layout. As such, it is usually the pragmatic choice to permit a TTI legality query in passes that are otherwise target-independent.

Even the InstCombine pass, which is likely the place with the hardest policy against target-dependent transforms, does use TTI to provide hooks for folding target-specific intrinsics. The thing we ultimately care about is that the canonical IR form remains target-independent to the degree that this is possible. Target-specific intrinsics naturally cannot have a target-independent canonical form.

Canonical form
--------------

The canonical IR form is not documented anywhere and essentially boils down to "whatever InstCombine produces, if it makes a choice one way or another". In some cases, the choice of canonical form is obvious. If it's possible to do the same thing with fewer instructions, that's almost always preferable (exceptions may be expensive instructions like integer division or certain intrinsics).

The question becomes more interesting when two sequences have the same number of instructions. For example, consider the following equivalent patterns:

```llvm
%tmp = add i32 %x, 2
%res = mul i32 %tmp, 3

%tmp = mul i32 %x, 3
%res = add i32 %tmp, 6
```

In principle, this could be canonicalized in either direction, and one might argue that the former is better because it leads to smaller constants.

However, we prefer the latter. There are a few reasons for why this is a better choice, but the primary one is that we can always convert the former into the latter, but the converse is not true. If you change the `add 6` to `add 5`, it's no longer possible to factor out the multiply.

This allows us to establish a canonical ordering of multiplies and adds, where we prefer "add of multiply" over "multiply of add". This canonical ordering makes patterns like the following fold in a two-step process:

```llvm
%tmp = add i32 %x, 2
%tmp2 = mul i32 %tmp, 3
%res = add i32 %tmp2, 5

; Step 1
%tmp = mul i32 %x, 3
%tmp2 = add i32 %tmp, 6
%res = add i32 %tmp2, 5

; Step 2
%tmp = mul i32 %x, 3
%res = add i32 %tmp, 11
```

As another example, let's consider these patterns:

```llvm
%res = add i32 %x, %x

%res = shl i32 %x, 1
```

These patterns are not equivalent: The latter is a refinement. This means we can convert the former into the latter, but the other way around requires the insertion of a `freeze` instruction. The reason for this is somewhat technical and I won't go into it here.

Even without this concern, we would canonicalize towards the latter pattern here, because it reduces the number of uses of `%x`, and this makes it more amenable to analysis. For example, the `shl` pattern makes it obvious that the low bit of the result is zero, while this would require special handling for the `add` pattern. Having only one use also makes it easier to fold the `shl` into whatever operation produced `%x`:

```llvm
%x = mul i32 %y, 3
%res = add i32 %x, %x

; Step 1
%x = mul i32 %y, 3
%res = shl i32 %x, 1

; Step 2
%res = mul i32 %y, 6
```

As a rule of thumb, for two patterns with the same number of instructions, we'll prefer the one with less value uses.

And here we can also see an undo transform in effect: The x86 backend will convert the `shl %x, 1` back into `add %x, %x` in order to form a `lea` instruction. The canonical IR preference goes against the target preference here. 

These are just some basic examples. InstCombine performs hundreds (thousands?) of different transforms, and the choice of canonical form is not always obvious. Sometimes it is entirely arbitrary, and we just chose something so that CSE/GVN sees a single pattern.

Sometimes the canonical form changes, because we later realize that a different form composes better with further transforms, or is less prone to infinite combine loops. Such changes may require adjusting other transforms (and/or backends) to work on the new canonical form. These changes are always somewhat risky, in that they can have unanticipated effects.

In any case, I hope this is enough to illustrate why we care about canonical IR form, and how it helps make transforms composable.

Summary
-------

The LLVM middle-end is target-dependent through three main sources: data layout (DL), target library info (TLI) and target transform info (TTI). The first two are considered unproblematic, but the use of the latter is restricted.

Canonicalization passes like InstCombine are not allowed to have target-dependent opt-outs or cost modelling. Instead, backend undo transforms should be implemented for undesirable canonicalizations.

[data_layout]: https://llvm.org/docs/LangRef.html#langref-datalayout
