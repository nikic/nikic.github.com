---
layout: post
title: "LLVM: Scalar evolution"
excerpt: A brief introduction to LLVM's scalar evolution analysis, which models how values change inside loops.
---
{% raw %}
Scalar evolution (SE or SCEV) models how values change inside loops. It can answer questions like "how many iterations does this loop perform?" and "What value does this variable have on the Nth iteration?" As such, it is the main building block for loop optimizations.

Scalar evolution models values using non-recursive expressions, which are called SCEVs. Yes, we use the same term for the whole analysis and individual expressions. The most important expression type is the add recurrence (addrec).

Add recurrences
---------------

Add recurrences model induction variables (IVs) which have a start value on the first iteration, and then increment by a fixed amount on each loop iteration. In LLVM IR, such an IV might look like this:

```llvm
loop:
  %iv = phi i32 [ 0, %preheader ], [ %iv.next, %loop ]
  call void @do_something(i32 %iv)
  %iv.next = add i32 %iv, 1
  %cmp = icmp ne i32 %iv.next, 100
  br i1 %cmp, label %loop, label %exit
```

The value `%iv` corresponds to the addrec `{0,+,1}<%loop>`, which says it is `0` on the first iteration, and then increments by one on each following iteration. We call this the pre-inc addrec.

The value `%iv.next` corresponds to the addrec `{1,+,1}<%loop>`, which is `1` on the first iteration, and also increments by one. This is the post-inc addrec.

This example uses the simplest possible canonical induction variable, but it generalizes to arbitrary start and step values:

```llvm
loop:
  %iv = phi i32 [ %start, %preheader ], [ %iv.next, %loop ]
  call void @do_something(i32 %iv)
  %iv.next = add i32 %iv, %step
  %cmp = icmp ne i32 %iv.next, 100
  br i1 %cmp, label %loop, label %exit
```

Here `%iv` is `{%start,+,%step}<%loop>` and `%iv.next` is `{(%start + %step),+,%step}<%loop>`.

However, `%step` still needs to be a loop-invariant value, which is the same for all iterations. SCEV cannot model recurrences with loop-varying steps, such as in this example:

```llvm
loop:
  %iv = phi i32 [ %start, %preheader ], [ %iv.next, %loop ]
  call void @do_something(i32 %iv)
  %step = call i32 @get_step()
  %iv.next = add i32 %iv, %step
  %cmp = icmp ne i32 %iv.next, 100
  br i1 %cmp, label %loop, label %exit
```

The exception to this is that the step might be an addrec itself, resulting in something like `{0,+,1,+,1}`, which should be interpreted as `{0,+,{1,+,1}}` and takes values 0, 1, 2, 4, 7, 11, 16, etc. These are called non-affine addrecs and have little practical relevance, so we can mostly ignore them.

Expressions
-----------

While add recurrences are what we primarily want to reason about, SCEV expressions also represent a number of other basic operations:

 * Addition: `(%a + %b + %c)`
 * Multiplication: `(%a * %b * %c)`
 * Min/max: `(%a umin %b umin %c)` (same for `umax`, `smin` and `smax`)
 * Sequential umin: `(%a umin_seq %b umin_seq %c)` (a non-commutative form of umin that has different poison-propagation semantics)
 * Unsigned division: `(%a /u %b)`
 * Zero-extend: `(zext i32 %a to i64)`
 * Sign-extend: `(sext i32 %a to i64)`
 * Truncate: `(trunc i64 %a to i32)`
 * Pointer-to-int cast: `(ptrtoint ptr %a to i64)`

SCEV expressions can be nested inside each other, while the leafs will always be either constants like 0 or "unknown" nodes that map to an IR value that cannot be analyzed further (like `%a` etc in these examples).

Unlike IR instructions, SCEV nodes for add/mul/min/max accept multiple operands. Rather than `(%a + (%b + %c))` or `((%a + %b) + %c)` we have `(%a + %b + %c)`. These operations are associative and commutative, so grouping or ordering of operands does not matter.

This nicely ties into the next property: SCEV expressions are canonicalized on construction. While something like `(%a + (%b + %c))` can technically be represented, SCEV will automatically fold it into `(%a + %b + %c)`. It also performs many other canonicalizations.

One of the more important ones is complexity sorting. As these operations are commutative, SCEV will bring them into a canonical order that is based on the type of SCEV expression first and domination relationships second. For example, constant operands will always come first, so that `(%a + 1)` always becomes `(1 + %a)` instead.

A notable absence from the expression list are subtractions: `(%a - %b)` is instead represented as `((-1 * %b) + %a)`. This ties in with the idea of having a single canonical form.

SCEV expressions are interned, which means that there is only one expression with a given set of operands/types. To determine whether two SCEV expressions are the same, you can just check if the pointers are equal.

Pointer expressions
-------------------

SCEV expressions have a type, which must be either an integer or pointer (the so called "SCEVable types"). However, only a subset of expressions support pointers:

 * addrecs can have a pointer start, while the step is always an integer.
 * add can have a single pointer operand, while the rest must be integers.
 * min/max are either all-integer or all-pointer.

For adds/addrecs the type of the integer operands must match the "index type" of the pointer. Typically, this will be the integer type with the same size as the pointer, but it can also be smaller for some exotic architectures like CHERI. This integer type is also known as the "effective type" of the SCEV.

An important consideration for pointer SCEVs is what happens if you try to subtract them: `((-1 * %b) + %a` is not a well-defined expression if `%a` and `%b` are pointers, because SCEV does not support pointer multiplication.

Instead, subtracting two pointer SCEVs requires that they have a common "pointer base": If you recursively follow the pointer operand of adds/addrecs, you must arrive at the same value. In that case, the common pointer base can be subtracted from both sides, and you are left with a well-defined integer subtraction.

This is the reason why, unlike most other SCEV construction methods, `getMinusSCEV()` is fallible.

Nowrap flags
------------

Add, mul and addrec nodes support the `nuw` (no unsigned wrap) and `nsw` (no signed wrap) flags familiar from IR. These specify that addition/multiplication cannot wrap in an unsigned/signed sense.

For addrecs, this property is limited to the number of iterations made by the loop. If the loop has 100 iterations, we don't care if the addrec would wrap on iteration 101.

Additionally, addrecs have a no-self-wrap `nw` flag, which determines whether the addrec may cross its initial value. This is a weaker property than either nsw or nuw.

We write the flags with the syntax `(%a + %b + %c)<nuw>` or `{0,+,1}<nsw>`. If the expression has more than two operands, then the nowrap flags must apply regardless of the order in which the additions/multiplications are performed.

While the nowrap flags in SCEV and IR look similar on the surface, they are actually fundamentally different. In IR, violating a nowrap flag returns poison, and it's possible for an `add` with and without nowrap flags to coexist.

This is not the case for SCEV expressions. SCEV expressions are interned without taking nowrap flags into account, which means that `(%a + %b)` and `(%a + %b)<nuw>` aren't considered to be distinct expressions. Once the `nuw` flag on `(%a + %b)` has been set, it must hold for *all* uses of `(%a + %b)`. A corollary of this is that nowrap flags cannot be removed.

More precisely, the nowrap property must hold within the defining scope of the SCEV expression, otherwise the behavior is undefined. The defining scope for addrecs would be their loop header, while for "unknown" nodes it is their defining IR instruction.

This is not a problem for nowrap flags that are inferred (e.g. from range information). However, it severely limits transfer of nowrap flags from IR instructions to SCEV expressions: Such transfer is only possible if we can show that violating the nowrap flags is guaranteed to result in undefined behavior once the defining scope has been entered.

To give a silly example, we can't transfer the `nuw` flag to SCEV in the first function, but can in the second:

```llvm
define i32 @cant_transfer(i32 %a, i32 %b) {
  %add = add nuw i32 %a, %b
  ret i32 %add
}

define noundef i32 @can_transfer(i32 %a, i32 %b) {
  %add = add nuw i32 %a, %b
  ret i32 %add
}
```

The defining scope here is the entire function. If `nuw` is violated, the first function will just return poison. The second function will cause undefined behavior due to the `noundef` attribute. It is guaranteed to do so whenever the function is entered, so we can transfer the flag to SCEV.

Exit count, backedge-taken count, trip count
--------------------------------------------

Next to modeling the evolution of values, SCEV can also determine how many iterations a loop performs. There are two different notions of "iterations" commonly used in this context:

The backedge-taken count (BE count) is how often the loop backedge is followed before the loop is exited. The trip count is how often the loop header is executed, which is probably what we'd usually call the "number of iterations". The trip count is the BE count plus one. A loop that exits on the first iteration has a BE count of 0 and a trip count of 1.

Part of the reason why we often work with BE counts is that (if it exists) it can always be represented in the same bit-width as the IV controlling the loop. The "plus one" addition of the trip count can overflow, so the trip count might require extending into a larger bit width.

Next, we need to distinguish between normal and abnormal exits. A loop can have multiple normal exits, in the form of conditional branches that either stay in the loop or leave it. Additionally, it can have abnormal exits in the form of potentially throwing operations. These are not explicitly modeled in the control-flow graph, but still exit the loop. It is also possible for calls to diverge (i.e. never return), which is not technically an exit, but for most practical purposes behaves like one.

For each normal exit, SCEV provides an exact, constant max and symbolic max exit count, which is a BE count in all cases. The exit count says that **if** this exit is taken, then it will be taken after this many iterations. It does not make any statements about whether it will actually be taken -- a different normal or abnormal exit could be taken instead.

The exact exit count is symbolic and specifies an exact number of backedges taken, such as `(-1 + %n)`. If we take this exit, it will be after this many iterations, not more, not less. The symbolic max exit count only guarantees that the exit will be taken after at most that many iterations. For example, if the exit condition is `%iv uge %n or %iv uge %m` the symbolic max exit count will be `(-1 + %n) umin (-1 + %m)`, and no exact exit count will be known.

The constant max exit count is a constant upper bound on the symbolic max. A common value for it is `-1` (or `UINT_MAX` in an unsigned interpretation). This means we don't know anything about the range of the exit count, but we do know that it is finite.

The same three options (exact, constant max, symbolic max) exist when we move from considering a single exit to considering the entire loop. These are once again under the assumption that the loop exits normally. The exact BE count will usually only be available for single exit loops.

SCEV cannot always determine an exit count (not even a constant/symbolic max). If the condition is not based on an induction variable (e.g. on a loop-varying load) there's no way to determine the exit count. Even if it's based on an IV, SCEV may not be able to exclude the possibility of an infinite loop.

If we have a known backedge taken count, we can determine which value an addrec (or expression based on one) will take once the loop has been exited. This is known as the "exit value", or the "SCEV at scope", where the "scope" is outside the loop of the addrec.

For example, if we have addrec `{2,+,3}` with BE count `(-1 + %n)`, then the exit value is `(2 + (3 * (-1 + %n)))` or `(-1 + (3 * %n))`.

Dispositions
------------

SCEV expressions have a lazily computed "disposition" towards any basic block and loop in the function. The possible block dispositions are "properly dominates", "dominates" and "does not dominate". The loop dispositions are "loop invariant", "loop computable" and "loop variant".

Most of these terms should be familiar from the IR layer, and have essentially the same meaning here. The only addition is "loop computable", which is the disposition of an addrec in its own loop: It's not invariant, but we can compute its value on any iteration.

However, there is one very important difference to IR-level queries: Because these are working on SCEV expressions, dominance and loop invariance is independent of the position of the original instruction in the IR, and only depends on the SCEV operands.

Consider the following example:

```llvm
loop:
  %iv = phi i32 [ 0, %preheader ], [ %iv.next, %loop.latch ]
  %iv.next = add i32 %iv, 1
  %cmp = icmp ne i32 %iv.next, %end
  br i1 %cmp, label %loop.latch, label %exit

loop.latch:
  %div = udiv i32 %a, %b    ; operands defined outside of loop
  br label %loop
```

Here, `udiv i32 %a, %b` cannot be hoisted out of the loop by LICM, because it could be that the loop exits on the first iteration, and the udiv is never reached. If `%b` were zero, hoisting the division would cause division by zero, which is undefined behavior. If you ask LoopInfo whether `%div` is loop invariant with respect to `%loop` the answer will be "no", as the instruction is inside the loop.

However, if we instead look at the SCEV expression `(%a /u %b)`, then it *is* loop invariant with respect to `%loop`. As such, domination / invariance on the SCEV level is stronger than on the IR level, as it behaves as-is expressions were fully hoisted, even for non-speculatable operations.


SCEV expansion
--------------

Sometimes, we want to go back from SCEV expressions to IR instructions, for example to materialize a backedge taken count or exit value in IR. This is handled by a separate SCEVExpander component, which will generate the necessary instruction sequence to compute the value of a SCEV expression at a given insertion point.

Before performing expansion, there are two things one usually wants to check: The first is whether the expression is "safe to expand". The most common reason for an expansion to be unsafe is that it contains a division where the denominator cannot be proven to be non-zero. This ties in directly with the previous section: Because SCEV expressions are effectively hoisted, expanding a division may move it outside a conditional branch.

The second is whether it is a "high cost expansion". SCEV expressions can become very large, and performing a transform that requires 50 instructions to compute a backedge taken count is likely not worthwhile.

The expander tries to generate more efficient code in two ways: By hoisting instructions as much as possible, and by reusing existing instructions. For the latter purpose (as well as invalidation), SCEV keeps track of which instructions map to a given SCEV expression. The expander can then use an existing instruction that dominates the insertion point.

However, there is a big caveat here: An instruction may be more poisonous than the corresponding SCEV expression. For example, consider two instructions `add i32 %a, %b` and `add nuw i32 %a, %b`. Both of these map to the same SCEV `(%a + %b)`, but if we want to expand it, reusing the instruction with the `nuw` flag may be incorrect, because it is more poisonous if the addition overflows.

For that reason, the expander has to prove that that the reused instruction either cannot be more poisonous, or it's possible to make it so by dropping poison-generating flags like `nuw`.

The last thing to be aware of, is that the expander has multiple modes of operation, which affect how add recurrences are expanded. By default, it uses "canonical mode", in which case addrecs will get expressed based on a canonical `{0,+,1}` induction variable. If you expand multiple different addrecs, there will only be a single canonical IV, and the other addrecs will be expressed as a multiply-add of that IV. In non-canonical mode, the recurrence is instead expanded literally.

Finally, there is also a special LSR mode, which is used by the loop strength reduction pass and allows precise control of how add recurrences are expanded. The details of how this works are outside the scope of this article.

Ranges, multiples and predicates
--------------------------------

SCEV supports calculating the possible value range of expressions: This is provided as a separate unsigned and signed range. Both ranges are always correct, but one of them may be more precise in an unsigned/signed context. Ranges are heavily used to infer nowrap flags and answer predicates like "is this SCEV known non-zero?"

Additionally, SCEV can determine whether an expression is always a multiple of some constant. For example `{0,+,3}` is always a multiple of three. Unsurprisingly, this is primarily used to determine loop trip multiples.

Finally, SCEV provides various functionality around answering predicates, such as "is A unsigned lower than B?" Answering this can involve range information, but also general symbolic reasoning.

These queries come in a number of different flavors:

 * Does a predicate always hold? (isKnownPredicate)
 * Does one predicate imply another predicate? (isImpliedCond)
 * Does a predicate hold at a specific program point? (isKnownPredicateAt, isLoopEntryGuardedByCond, etc)

The last one is heavily based on the second one: It checks whether any dominating conditions imply the predicate.

However, there is an alternative mechanism to achieve this: It is possible to "apply loop guards" to a SCEV expression, which will rewrite the expression itself based on information derived from dominating conditions. For example, if the loop is guarded by `%x != 0`, then the expression `%x` would be rewritten to `%x umin 1`.

As a result, range (and known multiple) queries on the rewritten expression become more precise. For certain types of predicates (like "is this expression non-zero?") this tends to both provide more accurate results and be more efficient.

Predicated scalar evolution
---------------------------

Sometimes, the incoming IR simply doesn't satisfy some pre-condition that is necessary for a transform. For example, to perform a vectorization we might need to know that an addrec does not wrap, but we don't actually know it.

In this case, "predicated" scalar evolution allows us to *assume* that a certain predicate holds, and compute SCEV expressions under that assumption.

To actually perform the transform, the loop needs to be versioned: Two copies of the loop are generated, one of them guarded by the assumed predicates. The guarded loop can then be transformed based on these predicates.

Problems
--------

Scalar evolution is an important and powerful analysis for loop optimizations, but it has quite a few and fairly fundamental problems. A few that come to mind are:

* The fact that nowrap flags are global causes two main issues: First, it is easy for non-experts to get wrong, because it differs from IR semantics. This has resulted in many miscompiles over the years. Second, SCEV can't represent cases where nowrap flags are only known to hold at a specific use, leading to missed optimizations.
* SCEV does not support subtraction and instead represents `sub i32 %a, %b` as `((-1 * %b) + %a)`. This means that nowrap flags from the subtraction cannot be preserved straightforwardly. It is generally impossible to represent nowrap for unsigned subtraction. This is also a problem in IR because sub by constant gets converted to add by constant. It's worse in SCEV because even non-constant cases are represented by adds. This is why loops that count down will usually get optimized worse than loops that count up.
* SCEV results are query-order dependent. If you fetch the SCEV for `%a` and then for `%b`, you might get different results than for `%b` then `%a`. SCEV queries can also affect nowrap flags on existing expressions. For example, computing a zext of an expression will often infer additional nowrap flags on it.
* SCEV has dozens of caches, making invalidation perilous and very expensive. Invalidation tends to scan large value graphs to make sure that all indirect dependencies are accounted for. We don't have a good understanding of what precisely needs to be invalidated and tend to go for the big hammer.
* SCEV in general, but especially anything involving context-sensitive reasoning, is very slow. Trying to use context-sensitive queries in a new place is pretty much a guaranteed major compile-time regression.

None of these are easy to resolve.
{% endraw %}
