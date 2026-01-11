---
layout: post
title: "LLVM: The bad parts"
excerpt: A collection of issues with LLVM, ranging from social and infrastructure problems to specific technical challenges.
---
A few years ago, I wrote a blog post on [design issues in LLVM IR][design_issues]. Since then, one of these issues has been fixed fully (opaque pointers migration), one has been mostly fixed (constant expression removal), and one is well on the way towards being fixed (ptradd migration).

This time I'm going to be more ambitious and not stop at three issues. Of course, not all of these issues are of equal importance, and how important they are depends on who you ask. In the interest of brevity, I will mostly just explain what the problem is, and not discuss what possible solutions would be.

Finally, I should probably point out that this is written from my perspective as the lead maintainer of the LLVM project: This is not a list of reasons to not use LLVM, it's a list of opportunities to improve LLVM.

## High level issues

### Review capacity

Unlike many other open-source projects, LLVM certainly [does not suffer][lfx_insights] from a lack of contributors. There are *thousands* of contributors and the distribution is relatively flat (that is, it's not the case that a small handful of people is responsible for the majority of contributions.)

What LLVM does suffer from is insufficient review capacity. There are a lot more people writing code than reviewing it. This is somewhat unsurprising, as code review requires more expertise than writing code, and may not provide immediate value[^review_value] to the person reviewing (or their employer).

Lack of review capacity makes for a bad contributor experience, and can also result in bad changes making their way into the codebase. The way this usually works out is that someone puts up a PR, then fails to get a qualified review for a long period of time, and then one of their coworkers (who is not a qualified reviewer for that area) ends up rubberstamping the PR.

A related problem is that LLVM has a somewhat peculiar contribution model where it's the responsibility of the PR author to request reviewers. This is especially problematic for new contributors, who don't know whom to request. Often relevant reviewers will become aware of the PR thanks to a label-based notification system, but this is not apparent from the UI, and it's easy for PRs to fall through the cracks.

A potential improvement here would be a Rust-style [PR assignment system][pr_assignment].

### Churn

Both the LLVM C++ API and LLVM IR are not stable and undergo frequent changes. This is simultaneously a great strength and weakness of LLVM. It's a strength because LLVM does not stagnate and is willing to address past mistakes even at significant cost. It's a weakness because churn imposes costs on users of LLVM.

Frontends are *somewhat* insulated from this because they can use the largely stable C API. However, it does not cover everything, and most major frontends will have additional bindings that use the unstable C++ API.

Users that integrate with LLVM more tightly (for example downstream backends) don't have that option, and have to keep up with all API changes.

This is part of LLVM's general development philosophy, which I'll express somewhat pointedly as "upstream or GTFO". LLVM is liberally licensed and does not require you to contribute changes upstream. However, if you do not upstream your code, then it will also not factor into upstream decision-making.

This point is somewhat unlike the rest, in that I'm not sure it's possible to make things "strictly better" here. It's possible that LLVM's current point on the stability scale is not optimal, but moving it somewhere else would come with significant externalities. Making major changes in LLVM is already extremely hard due to the sheer scale of the project, without adding additional stability constraints on top.

### Build time

LLVM is a huge project. LLVM itself is >2.5 million lines of C++ and the entire monorepo is something like 9 million. C++ is not exactly known for fast build times, and compiling all that code takes time. This is bearable if you either have fast hardware or access to a build farm, but trying to build LLVM on a low-spec laptop is not going to be fun.

An additional complication is building with debug info (which I always recommend against), in which case you'll add the extra gotchas of slow link times, high risk of OOM and massive disk usage. There are ways to avoid that (using shared libs or dylib build, using split dwarf, using lld), but it takes some expertise.

Promising changes in this area are the [use of pre-compiled headers][pch] (which significantly improves build time), and changing to use a [dylib build by default][dylib_default] (which reduces disk usage and link time, esp. for debuginfo builds). Another is to [reduce test performance][test_time_improvement] using daemonization (not strictly part of the "build time", but relevant for the development cycle).

### CI stability

LLVM CI consists of over 200 post-commit buildbots that test LLVM in lots of different configurations on lots of different hardware. Commits that turn a buildbot from green to red result in an email to the commit author.

Unfortunately, this CI is never fully green, and flaky on top. This is in part due to flaky tests (typically in lldb or openmp), but can also be due to buildbot-specific issues. The end result is that it's "normal" to get buildbot failure notifications for any given commit, even if it is perfectly harmless. This dilutes the signal, and makes it easier to miss the real failures.

The introduction of pre-merge testing on PRs did significantly improve the overall CI situation, but not the buildbot problem as such. I think we need to start taking flaky tests/buildbots more seriously before we can really make progress here.

Because someone is definitely going to mention how this [is not rocket science][not_rocket_science], and we just need to start using bors / merge queues to guarantee an always-green build: It's a problem of scale. There are >150 commits on a typical workday, which would be more than one commit every 10 minutes even if they were uniformly distributed. Many buildbots have multi-hour runs. This is hard to reconcile.[^bors]

### End-to-end testing

In some respects, LLVM has very thorough test coverage. We're quite pedantic about making sure that new optimizations have good coverage of both positive and negative tests. However, these tests are essentially unit tests for a single optimization pass or analysis.

We have only a small amount of coverage for the entire optimization pipeline (phase ordering tests), so optimizations sometimes regress due to pass interactions. Tests for the combination of the middle-end and backend pipelines are essentially nonexistent. There is likely room for improvement here, though it comes with tradeoffs.

However, what actually concerns me are end-to-end executable tests. LLVM's test suite proper does not feature these at all. Executable tests are located in a separate [llvm-test-suite][llvm-test-suite] repo, which is typically not used during routine development, but run by buildbots. It contains a lot of different code ranging from benchmarks to unit tests.

However, llvm-test-suite has quite few tests (compared to LLVM lit tests) and does not comprehensively cover basic operations. Things like testing operations on different float formats, on integers of different sizes, vectors of different sizes and element types, etc.

In part this is because of limitations of testing through C/C++, which is very heterogeneous in type support (C compilers don't like exposing types that don't have a defined psABI for the target). But that's no excuse to delegate this testing to Zig instead (which exposes everything, everywhere, and has the corresponding test coverage).

### Backend divergence

While LLVM's middle-end is very unified, backend implementations are very heterogeneous, and there is a tendency to fix issues (usually performance, but sometimes even correctness) only for the backend you're interested in.

This takes many forms, like implementing target-specific DAG combines instead of generic ones. Though my definite favorite is to introduce lots of target hooks for optimizations -- not because the optimization is actually only beneficial for one target, but because the person introducing it just doesn't want to deal with the fallout on other targets.

This is understandable -- after all, they may lack the knowledge to evaluate a change for other targets, so it may require working with many other maintainers, which can slow progress a lot. But the end result is still increasing divergence and duplication.

Lack of end-to-end testing compounds this issue, because that would act as something of a forcing function that at least all operations compile without crashing and produce correct results for all tested targets.

### Compilation time

Because I've [complained][llvm_fast] about this enough in the past, I'll keep it short: LLVM is slow, which is an issue both for JIT use cases, and anything that tends to produce huge amounts of IR (like Rust or C++).

Since I've started [tracking compile-times][llvm_compile_time_tracker], the situation has significantly improved, both through targeted improvements and avoidance of regressions. However, there is still a lot of room for improvement: LLVM still isn't fast, it's just less slow.

One thing that LLVM is particularly bad at are `-O0` compile-times. The architecture is optimized for optimization, and lots of costs remain even if no optimization takes place. The [LLVM TPDE][llvm_tpde] alternative backend shows that it's possible to do better by an order of magnitude.

### Performance tracking

The flip side of the compile-time coin is runtime performance. This is something that LLVM obviously cares a lot about. Which is why I find it rather surprising that LLVM does not have any "official" performance tracking infrastructure.

Of course, there are lots of organizations which track performance of LLVM downstream, on their own workloads. In some ways this is good, because it means there is more focus on real-world workloads than on synthetic benchmarks like SPEC. However, not having readily accessible, public performance tracking also makes it hard for contributors to evaluate changes.

To be fair, LLVM does have an [LNT][lnt] instance, but a) it's currently broken, b) LNT is one of the worst UX crimes ever committed, c) little data gets submitted there, and d) it's not possible to request a test run for a PR, or something like that.

This point is frankly just baffling to me. I don't personally care about SPEC scores, but I know plenty of people do, so why there is no first-class tracking for this is a mystery to me.

## IR design

### Undef values

[Undef values][undef] take an arbitrary value from a certain set. They are used to model uninitialized values, and have historically been used to model deferred undefined behavior. The latter role has been replaced by poison values, which have much simpler propagation rules and are more amenable to optimization. However, undef is still used for uninitialized memory to this day.

There are two main problems with undef values. The first is the multi-use problem: An undef value can take a different value at each use. This means that transforms that increase the use count are generally invalid, and care has to be taken when optimizing based on value equality. The mere existence of undef values prevents us from performing optimizations we want to do, or greatly increases their complexity.

The second issue is that undef is very hard to reason about. Humans have trouble understanding it, and for proof-checkers it is computationally expensive.

Most likely, uninitialized memory will be represented using poison values instead in the future, but this runs into the problem that LLVM currently is not capable of correctly treating poison in memory. Proper support for poison in memory requires additional IR features, like the [byte type][byte_type].

### Unsoundness and specification incompleteness

While most miscompilations (that is, correctness bugs) in LLVM are resolved quickly, there are quite a few that remain unfixed despite having been known for a long time. These issues usually combine the qualities of being largely theoretical (that is, appearing only in artificially constructed examples rather than real-world code) and running up against issues in LLVM's IR design.

Some of them are cases where we have a good idea of how the IR design needs to change to address the issue, but these changes are complex and often require a lot of work to recover optimization parity. There is often a complexity cliff where you can do something that's simple and *nearly* correct, or you can do something very complex that is fully correct.

Then there are other cases, where just deciding on how things *should* work is a hard problem. The provenance model is a prime example of this. The interaction of provenance with integer casts and type punning is a difficult problem with complex tradeoffs.

However, at some point these issues do need to be resolved. The recently formed [formal specification working group][formal_spec] aims to tackle these problems.

### Constraint encoding

A key challenge for optimizing compilers is encoding of constraints (like "this value is non-negative" or "this add will not overflow"). This includes both frontend-provided constraints (based on language undefined behavior rules), but also compiler-generated ones.

In particular, there are many different analyses that can infer facts about the program, but keeping these up-to-date throughout optimization is challenging. One good way to handle this is to encode facts directly in the IR. Correctly updating or discarding these annotations then becomes part of transform correctness.

LLVM has many different ways to encode additional constraints (poison flags, metadata, attributes, assumes), and these all come with tradeoffs in terms of how much information can be encoded, how reliably it is retained during optimization and to what degree it can *negatively* affect optimization. Information from metadata is lost too often, while information from assumes is not lost often enough.

### Floating-point semantics

There are various issues with floating-point (FP) semantics once we move outside the nice world of "strictly conforming IEEE 754 floats in the default environment". A few that come to mind are:

* Handling of signaling NaN and FP exceptions, and non-default FP environment in general. LLVM represents this using constrained FP intrinsics. This is not ideal, as all the FP handling is split into two parallel universes.
* Handling of denormals. LLVM has a function attribute to not assume IEEE denormal behavior, but this is only suitable for cases where flush to zero (FTZ) is used globally. It does not help with modeling cases like ARM, where scalar ops are IEEE, while vector ops use FTZ.
* Handling of excess precision, in particular when using the x87 FPU.

<!--### Type-based representation

This is something I've written about enough in the past, so I'll keep it brief: For historical reasons, LLVM IR encodes unnecessary type information that no longer carries any semantic meaning in various places. The biggest offender here were pointer element types, which were removed by the [opaque pointers migration][opaque_pointers].-->

## Other technical issues

### Partial migrations

LLVM is a very large project, and making any significant changes to it is hard and time consuming. Migrations often span years, where two different implementations of something coexist, until all code has been migrated. The two prime examples of this are:

**New pass manager:** The "new" pass manager was first introduced more than a decade ago. Then about five years ago, we started using it for the middle-end optimization pipeline by default, and support for the legacy PM was dropped.

However, the back-end is still using the legacy pass manager. There is ongoing work to support the new pass manager in codegen, and we're pretty close to the point where it can be used end-to-end for a single target. However, I expect it will still take quite a while for all targets to be ported and the legacy pass manager to be completely retired.

**GlobalISel:** This is an even more extreme case. GlobalISel is the "new" instruction selector that is intended to replace SelectionDAG (and FastISel). It was introduced approximately one decade ago, and to this day, none of the targets that originally used SelectionDAG have been fully migrated to GlobalISel. There is one new target that's GlobalISel-only, and there is one that uses GlobalISel by default for unoptimized builds. But otherwise, SelectionDAG is still the default everywhere.

There are two backends (AMDGPU and AArch64) that have somewhat complete GlobalISel support, but it's not clear when/if they'll be able to switch to using it by default. A big problem here is that new optimizations are continually being implemented on the SDAG side, so it's hard to keep parity.

### ABI / calling convention handling

Essentially everything about the handling of calling conventions in LLVM is a mess.

The responsibility for handling calling conventions is split between the frontend and the backend. There are good reasons why LLVM can't do this by itself (LLVM IR sits at a too low level of abstraction to satisfy the extremely arcane ABI rules).

This is not a problem in itself -- however, there is zero documentation of what the calling convention contract between the frontend and LLVM is, and the proper way to implement C FFI is essentially to look at what Clang does and copy that (invariably with errors, because the rules can be very subtle).

I've proposed to fix this by introducing an [ABI lowering library][abi_library_rfc] and vortex73 has [implemented a prototype][abi_library_blog] for it as part of GSoC. So we're well on the way to resolving this side of the problem.

There are more problems though. One that Rust has struggled with a lot is the interaction of target features with the calling convention. Enabling additional target features can change the call ABI, because additional float/vector registers start getting used for argument/return passing. This means that calls between functions with a feature enabled and disabled may be incompatible, because they assume different ABIs.

Ideally, ABI and target features would be orthogonal, and only coupled in that some ABIs require certain target features (e.g. you can't have a hard float ABI without enabling FP registers). Target features are a per-function choice, while the ABI should be per-module.

Some of the newer architectures like Loongarch and RISC-V actually have proper ABI design, but most of the older ones don't. For example, it's currently not possible to target AArch64 with a soft float ABI but hard float implementation.

### Builtins / libcalls

Somewhat related to this is the handling of compiler builtins/libcalls, which are auxiliary functions that the compiler may emit for operations that are not natively supported by the target. This covers both libcalls provided by libc (or libm), and builtins provided by compiler runtime libraries like libgcc, compiler-rt or compiler-builtins.

There are two sources of truth for this, TargetLibraryInfo (TLI) and RuntimeLibcalls. The former is used by the middle-end, primarily to recognize and optimize C library calls (this mostly covers only libc, but not libgcc). The latter is used by the backend, primarily to determine which libcalls may be emitted by the compiler and how they are spelled (this covers libgcc, and the subset of libc covered by LLVM intrinsics).

A problem with RuntimeLibcalls is that it currently largely works off only the target triple, which means that we have to make "lowest common denominator" assumptions about which libcalls are available, where the lowest common denominator is usually libgcc. If `--rtlib=compiler-rt` is used, LLVM does not actually know about that, and cannot make use of functions that are in compiler-rt but not libgcc.

This also means that we're missing a customization point for other runtime libraries. For example, there is no way for Rust to say that it provides f128 suffix libcalls via compiler-builtins, overriding target-specific naming and availability assumptions based on which type `long double` in C maps to.

There is a lot of ongoing work in this area (by arsenm), so the situation here will hopefully improve in the near-ish future.

### Context / module dichotomy

LLVM has two high-level data holders. A module corresponds to a compilation unit (e.g. pre-LTO, a single file in C/C++). The LLVM context holds various "global" data. There's usually one context per thread, and multiple modules can (in principle) use a single context.

Things like functions and globals go into the module, while constants and types go into the context. The module also contains a data layout, which provides important type layout information like "how wide is a pointer".

The fact that constants and types do not have access to the data layout is a constant source of friction. If you have a type, you cannot reliably tell its size without threading an extra parameter through everything. We have subsystems (like ConstantFold vs. ConstantFolding) that are separated entirely by whether data layout is available or not.

At the same time, I feel like this split is not actually buying us a lot. Having shared types and constants is somewhat convenient when it comes to module linking, because they can be directly shared, but I think performing explicit remapping in that one place would be better than having complexity everywhere else. Additionally, this would also allow cross-context linking, which is currently only possible by going through a bitcode roundtrip. In theory, the context could also allow some memory reuse when compiling multiple modules, but I think in practice there is usually a one-to-one correspondence between those.

### LICM register pressure

This is getting a bit down in the weeds, but I'll mention it anyway due to how often I've run across this in recent times.

LLVM considers loop invariant code motion (LICM) to be a canonicalization transform. This means that we always hoist instructions out of loops, without any target specific cost modelling. However, LICM can increase the live ranges of values, which can increase register pressure, which can lead to a large amount of spills and reloads.

The general philosophy behind this is that LICM hoists everything, all middle-end transforms can work with nicely loop invariant instructions, and then instructions will get sunk back into the loop by the backend, which can precisely model register pressure.

Except... that second part doesn't actually happen. I believe that (for non-PGO builds) instructions only get sunk back into loops either through rematerialization in the register allocator, or specialized sinking (typically of addressing modes), but for anything not falling into those buckets, no attempt to sink into loops in order to reduce register pressure is made.

## Other

This list is not exhaustive. There's more I could mention, but we'd get into increasingly narrow territory. I hope I covered most of the more important things -- please do let me know what I missed!




[^bors]: The way Rust reconciles this is via a combination of "rollups" (where multiple PRs are merged as a batch, using human curation), and a substantially different contribution model. Where LLVM favors sequences of small PRs that do only one thing (and get squash merged), Rust favors large PRs with many commits (which do not get squashed). As getting an approved Rust PR merged usually takes multiple days due to bors, having large PRs is pretty much required to get anything done. This is not necessarily bad, just very different from what LLVM does right now.

[^review_value]: If you're not concerned with overall project health, the primary value of reviews is reciprocity. People are more likely to review your PR, if you reviewed theirs.

[pch]: https://discourse.llvm.org/t/rfc-use-pre-compiled-headers-to-speed-up-llvm-build-by-1-5-2x/89345?u=nikic
[dylib_default]: https://discourse.llvm.org/t/rfc-llvm-link-llvm-dylib-should-default-to-on-on-posix-platforms/85908?u=nikic
[test_time_improvement]: https://discourse.llvm.org/t/rfc-reducing-process-creation-overhead-in-llvm-regression-tests/88612?u=nikic
[not_rocket_science]: https://graydon2.dreamwidth.org/1597.html
[lfx_insights]: https://insights.linuxfoundation.org/project/llvm-llvm-project
[llvm-test-suite]: https://github.com/llvm/llvm-test-suite
[llvm_fast]: https://www.npopov.com/2020/05/10/Make-LLVM-fast-again.html
[llvm_compile_time_tracker]: https://llvm-compile-time-tracker.com/
[llvm_tpde]: https://discourse.llvm.org/t/tpde-llvm-10-20x-faster-llvm-o0-back-end/86664?u=nikic
[lnt]: https://lnt.llvm.org
[undef]: https://llvm.org/docs/UndefinedBehavior.html#undef-values
[abi_library_rfc]: https://discourse.llvm.org/t/rfc-an-abi-lowering-library-for-llvm/84495?u=nikic
[abi_library_blog]: https://blog.llvm.org/posts/2025-08-25-abi-library/
[byte_type]: https://blog.llvm.org/posts/2025-08-29-gsoc-byte-type/
[langref]: https://llvm.org/docs/LangRef.html
[formal_spec]: https://discourse.llvm.org/t/rfc-forming-a-working-group-on-formal-specification-for-llvm/89056?u=nikic
[design_issues]: https://www.npopov.com/2021/06/02/Design-issues-in-LLVM-IR.html
[ptradd]: https://discourse.llvm.org/t/rfc-replacing-getelementptr-with-ptradd/68699
[opaque_pointers]: https://llvm.org/docs/OpaquePointers.html
[pr_assignment]: https://forge.rust-lang.org/triagebot/pr-assignment.html
