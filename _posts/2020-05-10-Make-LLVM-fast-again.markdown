---
layout: post
title: Make LLVM fast again
excerpt: Introduces a compile-time tracking service for LLVM, and some specific compile-time improvements I have implemented.
---
The front page of the [LLVM website](https://llvm.org/) proudly claims that:

> Clang is an "LLVM native" C/C++/Objective-C compiler, which aims to deliver amazingly fast compiles [...]

I'm not sure whether this has been true in the past, but it certainly isn't true now. Each LLVM release is a few percent slower than the last. LLVM 10 put some extra effort in this area, and somehow managed to make Rust compilation a whole [10% slower](https://perf.rust-lang.org/compare.html?start=9381e8178b49636d4604e4ec0f1263960691c958&end=c6d04150234d5cb973ec5213d27a38e2a7b67955), for as yet unknown reasons.

One might argue that this is expected, as the optimization pipeline is continuously being improved, and more aggressive optimizations have higher compile-time requirements. While that may be true, I don't think it is a desirable trend: For the most part, optimization is already "good enough", and additional optimizations have the unfortunate trend to trade large compile-time increases for very minor (and/or very rare) improvements to run-time performance.

The larger problem is that LLVM simply does not track compile-time regressions. While [LNT](https://lnt.llvm.org/) tracks run-time performance over time, the same is not being done for compile-time or memory usage. The end result is that patches introduce unintentional compile-time regressions that go unnoticed, and can no longer be easily identified by the time the next release rolls out.

## Tracking LLVM compile-time performance

The first priority then is to make sure that we can identify regressions accurately and in a timely manner. Rust does this by running a set of benchmarks on every merge, with the data available on [perf.rust-lang.org](https://perf.rust-lang.org/). Additionally, it is possible to run benchmarks against pull requests using the `@rust-timer` bot. This helps evaluate changes that are intended to improve compile-time performance, or are suspected of having non-trivial compile-time cost.

I have set up a similar service for LLVM, with the results viewable at [llvm-compile-time-tracker.com](http://llvm-compile-time-tracker.com/). Probably the most interesting part are the relative [instructions](http://llvm-compile-time-tracker.com/graphs.php?stat=instructions&relative=on) and [max-rss](http://llvm-compile-time-tracker.com/graphs.php?stat=max-rss&relative=on) graphs, which show the percentual change relative to a baseline. I want to briefly describe the setup here.

The measurements are based on [CTMark](https://github.com/llvm/llvm-test-suite/tree/master/CTMark), which is a collection of some larger programs that are part of the LLVM test suite. These were added as part of a previous attempt to track compile-time.

For every tested commit, the programs are compiled in three different [configurations](https://github.com/llvm/llvm-test-suite/tree/master/cmake/caches): `O3`, `ReleaseThinLTO` and `ReleaseLTO-g`. All of these use `-O3` in three different LTO configurations (none, thin and fat), with the last one also enabling debuginfo generation.

Compilation and linking statistics are gathered using `perf` (most of them), GNU `time` (max-rss and wall-time) and `size` (binary size). The following statistics are available:

    instructions  (stable and useful)
    max-rss       (stable and useful)
    task-clock    (way too noisy)
    cycles        (noisy)
    branches      (stable)
    branch-misses (noisy)
    wall-time     (way too noisy)
    size-total    (completely stable)
    size-text     (completely stable)
    size-data     (completely stable)
    size-bss      (completely stable)

The most useful statistics are instructions, max-rss and size-total/size-text, and these are the only ones I really look at. "instructions" is a stable proxy metric for compile-time. Instructions retired is not a perfect metric, because it discounts issues like cache/memory latency, branch misprediction and ILP, but most of the performance problems affecting LLVM tend to be simpler than that.

The actual time metrics task-clock and wall-time are too noisy to be useful and also undergo "seasonal variation". This could be mitigated by running benchmarks many times, but I don't have the compute capacity to do that. Instructions retired on the other hand is very stable, and allows us to confidently identify compile-time changes as small as 0.1%.

Max-rss is the maximum resident set size, which is one possible measure of memory usage (a surprisingly hard concept to pin down). In aggregate, this metric is also relatively stable, with the exception of the ThinLTO configuration.

The binary size metrics are not really useful to judge compile time, but they do help to identify whether a change has impact on codegen. A compile-time regression that results in a code-size change is at least doing *something*. And the amount and structure of IR during optimization can have a significant impact on compile-time.

Different benchmarks have different variance. The detailed comparison pages for individual commits, like [this max-rss comparison](http://llvm-compile-time-tracker.com/compare.php?from=3f7439b28063c284975b49ebdc9c5645cedae7a0&to=fe8abbf4425e20b5d3865924654f32b60f116d80&stat=max-rss), highlight changes in red/green that are likely to be significant. Highlighting starts at 3 sigma (no color) and ends at 4 sigma (a clear red/green). The highlighting has no relation to the size of the change, only to its significance. Sometimes a 0.1% change is significant, and sometimes a 10% change is not significant.

In addition to the three configurations, the comparison view also shows link-only data for the ThinLTO/LTO configurations, as these tend to be a build bottleneck. It's also possible to show data for all the individual files (the "per-file details" checkbox).

The benchmark server communicates exclusively over Git: Whenever it is idle, it fetches the `master` branch of LLVM upstream, as well as any branches starting with `perf/` from a number of additional LLVM forks on GitHub. These `perf/` branches can be used to run experiments without committing to upstream. If anyone is interested in doing LLVM compile-time work, I can easily add additional forks to listen to.

After measurements have been performed, the data is pushed to the [llvm-compile-time-data](https://github.com/nikic/llvm-compile-time-data) repository, which stores the raw data. The website displays data from that repository.

The server this runs on only has 2 cores, so a full LLVM build can take more than two hours. For smaller changes, building LLVM from ccache and compiling the benchmarks takes about 20 minutes. This is too slow to test every single commit, but we don't really need to do that, as long as we automatically bisect any ranges with significant changes.

## Compile-time improvements

Since I started tracking, the geomean compile-time on CTMark has been reduced by 8-9%, as the following graph shows.

<img src="/images/llvm_geomean_instructions.png" alt="LLVM geomean instruction count over time" style="max-width: 100%" />

Most compile-time improvements and regressions tend to be small, with only few large jumps. A change of 0.25% is already worth looking at. A change of 1% is large. In the following I'll describe some of the improvements that I have implemented over the last few weeks.

The largest one, and a complete outlier in terms of impact, was switching [string attributes to use a map](https://github.com/llvm/llvm-project/commit/8f4c78dcf8a454a2a8a0fa04fe34e2162efd4a5c), which gave a [3% improvement](http://llvm-compile-time-tracker.com/compare.php?from=b9de62c2b69e3b6342045ae82e1532d4f70c6d34&to=8f4c78dcf8a454a2a8a0fa04fe34e2162efd4a5c&stat=instructions).

Attributes in LLVM come in two forms: Enum attributes, which are predefined (e.g. `nonnull` or `dereferenceable`), and string attributes, which can be free-form (`"use-soft-float"="true"`). While enum attributes are stored in a bitset for efficient lookup, string attributes are only accessible by scanning through the whole attribute list and comparing attribute names one by one. As Clang tends to generate quite a few function attributes (20 enum and string attributes are normal) this has a large cost, especially when it comes to lookup of attributes which are not actually set. The patch introduces an additional map from name to attribute to make this lookup more efficient.

A [related change](https://reviews.llvm.org/D78862) that is still under review is to convert the `"null-pointer-is-valid"` string attribute into an enum attribute, which will give another 0.4% improvement. This is one of the most commonly queried string attributes, because it influences pointer semantics fundamentally.

One common source of performance issues in LLVM are [`computeKnownBits()`](https://github.com/llvm/llvm-project/blob/master/llvm/lib/Analysis/ValueTracking.cpp#L1866) queries. These are recursive queries that determine whether any bits of a value are known to be zero or one. While the query is depth-limited, it can still explore quite a few instructions.

There's two ways to optimize this: Make the query cheaper, or make less calls to it. To make the query cheaper, the most useful technique is to skip additional recursive queries if we already know that we cannot further improve the result.

[Short-circuiting GEP calculations](https://github.com/llvm/llvm-project/commit/8148b116474614c176d69de8c246fc21494faf5f) gives us a [0.65% improvement](http://llvm-compile-time-tracker.com/compare.php?from=b7e2358220f26ee82e0e958f2d691d2f00341a0a&to=8148b116474614c176d69de8c246fc21494faf5f&stat=instructions). Getelementptr essentially adds type-scaled offsets to a pointer. If we don't know any bits of the base pointer, we shouldn't bother computing bits of the offsets. Doing the same [for add/sub instructions](https://github.com/llvm/llvm-project/commit/7a62ea3889b94516f3886cec9e447f22b99856e3) gives a more modest [0.3% improvement](http://llvm-compile-time-tracker.com/compare.php?from=9ab0c9a64402359647660cf1e3eca6006d375cf5&to=7a62ea3889b94516f3886cec9e447f22b99856e3&stat=instructions).

Of course, it is best if we can avoid calling `computeKnownBits()` calls in the first place. InstCombine used to perform a known bits calculation for each instruction, in the hope that all bits are known and the instruction can be folded to a constant. Predictably, this happens only very rarely, but takes up a lot of compile-time.

[Removing this fold](https://github.com/llvm/llvm-project/commit/2b52e4e629e6793f832caef5e47f9d84607740f3) resulted in a [1% improvement](http://llvm-compile-time-tracker.com/compare.php?from=9b95929a26e133bc3cae9f29f91e8e351d233840&to=2b52e4e629e6793f832caef5e47f9d84607740f3&stat=instructions). This required a good bit of groundwork to ensure that all useful cases really are covered by other folds. More recently, I've [removed the same fold in InstSimplify](https://github.com/llvm/llvm-project/commit/5a2265647ed3f449e9e8e970e27f5e964db851af), for another [0.8% improvement](http://llvm-compile-time-tracker.com/compare.php?from=989ae9e848a079715c2d23e5d3622cac9b48e08e&to=5a2265647ed3f449e9e8e970e27f5e964db851af&stat=instructions).

[Using a SmallDenseMap](https://github.com/llvm/llvm-project/commit/dbf78ae12874c5ff2ecf4b6f49bfae616a40e11c) in hot LazyValueInfo code resulted in a [0.5% improvement](http://llvm-compile-time-tracker.com/compare.php?from=71f8b78d89798f6b4a4645ffcb3aa461ccb89111&to=dbf78ae12874c5ff2ecf4b6f49bfae616a40e11c&stat=instructions). This prevents an allocation for a map that usually only has one element.

While looking into sqlite3 memory profiles, I noticed that the [ReachingDefAnalysis](https://github.com/llvm/llvm-project/blob/master/llvm/lib/CodeGen/ReachingDefAnalysis.cpp) machine pass dominated peak memory usage, which I did not expect. The core problem is that it stores information for each register unit (about 170 on x86) for each machine basic block (about 3000 for this test case).

I applied a number of optimizations to this code, but two had the largest effect: First, avoiding [full reprocessing of loops](https://github.com/llvm/llvm-project/commit/259649a51982d0ea6fdbaa62a87e802c9a8a86d2), which was a [0.4% compile-time improvement](http://llvm-compile-time-tracker.com/compare.php?from=76e987b37220128929519c28bef5c566841d9aed&to=259649a51982d0ea6fdbaa62a87e802c9a8a86d2&stat=instructions) and [1% memory usage improvement](http://llvm-compile-time-tracker.com/compare.php?from=76e987b37220128929519c28bef5c566841d9aed&to=259649a51982d0ea6fdbaa62a87e802c9a8a86d2&stat=max-rss) on sqlite. This is based on the observation that we don't need to recompute instruction defs twice, it is enough to propagate the already computed information across blocks.

Second, storing reaching definitions [inside a TinyPtrVector](https://github.com/llvm/llvm-project/commit/952c2741599ed492cedd37da895d7e81bc175ab9) instead of a SmallVector, which was a [3.3% memory usage improvement](http://llvm-compile-time-tracker.com/compare.php?from=8abfd2c3bb0d66a123b6a6ae590a3d0200f7a688&to=952c2741599ed492cedd37da895d7e81bc175ab9&stat=max-rss) on sqlite. The TinyPtrVector represents zero or one reaching definitions (by far the most common) in 8 bytes, while the SmallVector used 24 bytes.

Changing [MCExpr to use subclass data](https://github.com/llvm/llvm-project/commit/8e7d771cf9b6197c723e1ea8739563d24aca2e3c) was a [2% memory usage improvement](http://llvm-compile-time-tracker.com/compare.php?from=a916e819275922ab9a350283a12647da6f4ad4b1&to=8e7d771cf9b6197c723e1ea8739563d24aca2e3c&stat=max-rss) for the LTO with debuginfo link step. This change made previously unused padding bytes in MCExpr available for use by subclasses.

Finally, [clearing value handles in BPI](https://github.com/llvm/llvm-project/commit/fe8abbf4425e20b5d3865924654f32b60f116d80) was a [2.5% memory usage improvement](http://llvm-compile-time-tracker.com/compare.php?from=3f7439b28063c284975b49ebdc9c5645cedae7a0&to=fe8abbf4425e20b5d3865924654f32b60f116d80&stat=max-rss) on the sqlite benchmark. This was a plain bug in analysis management.

One significant compile-time improvement that I did not work on is the [removal of waymarking](https://github.com/llvm/llvm-project/commit/ff9379f4b2d7ebcb8dee94df47dc43c3388f22bf) in LLVMs use-list implementation. This was a [1% improvement](http://llvm-compile-time-tracker.com/compare.php?from=54cfc6944e2669d7a41a164fc4f3d923a71e701d&to=ff9379f4b2d7ebcb8dee94df47dc43c3388f22bf&stat=instructions) to compile-time, but also a [2-3% memory usage regression](http://llvm-compile-time-tracker.com/compare.php?from=54cfc6944e2669d7a41a164fc4f3d923a71e701d&to=ff9379f4b2d7ebcb8dee94df47dc43c3388f22bf&stat=max-rss) on some benchmarks.

[Waymarking](http://heisenbug.blogspot.com/2008/04/llvm-data-structures-and-putting-use-on.html) was previously employed to avoid explicitly storing the user (or "parent") corresponding to a use. Instead, the position of the user was encoded in the alignment bits of the use-list pointers (across multiple pointers). This was a space-time tradeoff and reportedly resulted in major memory usage reduction when it was originally introduced. Nowadays, the memory usage saving appears to be much smaller, resulting in the removal of this mechanism. (The cynic in me thinks that the impact is lower now, because everything else uses much more memory.)

Of course, there were also other improvements during this time, but this is the main one that jumped out.

## Regressions prevented

Actively improving compile-times is only half of the equation, we also need to make sure that regressions are reverted or mitigated. Here are some regressions that did not happen:

A [change to the dominator tree implementation](https://github.com/llvm/llvm-project/commit/a90374988e4eb8c50d91e11f4e61cdbd5debb235) caused a [3% regression](http://llvm-compile-time-tracker.com/compare.php?from=37bcf2df01cfa47e4509a5d225a23e2ca95005e6&to=a90374988e4eb8c50d91e11f4e61cdbd5debb235&stat=instructions). This was reverted due to build failures, but I also reported the regression. To be honest I don't really understand what this change does.

A seemingly harmless [TargetLoweringInfo change](https://github.com/llvm/llvm-project/commit/60c642e74be6af86906d9f3d982728be7bd4329f) resulted in a [0.4% regression](http://llvm-compile-time-tracker.com/compare.php?from=5b18b6e9a84d985c0a907009fb71de7c1943bc88&to=60c642e74be6af86906d9f3d982728be7bd4329f&stat=instructions). It turned out that this was caused by querying a new `"veclib"` string attribute in hot code, and this was the original motivation for the attribute improvement mentioned previously. This change was also reverted for unrelated reasons, but should have much smaller performance impact when it relands.

A change to the [SmallVector implementation](https://github.com/llvm/llvm-project/commit/b8d08e961df1d229872c785ebdbc8367432e9752) caused a [1% regression](http://llvm-compile-time-tracker.com/compare.php?from=73b7dd1fb3c17a4ac4b1f1e603f26fa708009649&to=b8d08e961df1d229872c785ebdbc8367432e9752&stat=instructions) to compile-time and memory usage. This patch changed SmallVector to use `uintptr_t` size and capacity for small element types like `char`, while normally `uint32_t` is used to save space.

It turned out that this regression was caused by moving the vector grow implementation for POD types into the header, which (naturallyresulted in excessive inlining. The memory usage did not increase because SmallVectors take up more space, but because the clang binary size increased so much. An improved version of the change was later [reapplied](https://github.com/llvm/llvm-project/commit/dda3c19a3618dce9492687f8e880e7a73486ee98) with essentially no impact.

The emission of [alignment attributes for sret parameters](https://github.com/llvm/llvm-project/commit/de98cf92e301ab559a7417f1eca5cfa53624c9e1) in clang caused a [1.4% regression on a single benchmark](http://llvm-compile-time-tracker.com/compare.php?from=43a6d285bfead762ac472a6e62beedc9f88bce89&to=de98cf92e301ab559a7417f1eca5cfa53624c9e1&stat=instructions).

It turned out that this was caused by the emission of alignment-preserving assumptions during inlining. These assumptions provide little practical benefit, while both increasing compile-time and pessimizing optimizations (this is a general LLVM issue in the process of being addressed with a new operand-bundle based assumption system).

We previously ran into this issue with Rust and disabled the functionality there, because rustc emits alignment information absolutely everywhere. This is contrary to Clang, which only emits alignment for exception cases. The referenced sret change is the first deviation from that approach, and thus also the first time alignment assumptions became a problem. This regression was mitigated by [disabling alignment assumptions](https://github.com/llvm/llvm-project/commit/b74c6d2c9d8e57db96742094cc4daf98a258b412) by default.

One regression that I failed to prevent is a steady increase in max-rss. This increase is caused primarily by an increase in clang binary size. I have only started tracking this recently (see the [clang binary size graph](http://llvm-compile-time-tracker.com/graphs.php?stat=size-total&relative=on&bench=clang)) and in that time binary size increased by nearly 2%. This has been caused by the addition of builtins for ARM SVE, such as [this commit](https://github.com/llvm/llvm-project/commit/b32d14c30e45dd60df435456b4e6747fd83590bb). I'm not familiar with the builtins tablegen system and don't know if they can be represented more compactly.

## Conclusion

I can't say a 10% improvement is making LLVM fast again, we would need a 10x improvement for it to deserve that label. But it's a start...

One key problem here is the choice of benchmarks. These are C/C++ programs compiled with Clang, which generates very different IR from rustc. Improvements for one may not translate to improvements for the other. Changes that are neutral for one may be large regressions for the other. It might make sense to include some rustc bitcode outputs in CTMark to give non-Clang frontends better representation.

I think there is still quite a lot of low-hanging fruit when it comes to LLVM compile-time improvements. There's also some larger ongoing efforts that have mostly stalled, such as the migration to the new pass manager, the migration towards opaque pointers (which will eliminate many bitcast instructions), or the NewGVN pass.

Conversely, there are some ongoing efforts that might make for large compile-time regressions when they do get enabled, such as the attributor framework, the knowledge retention framework and MemorySSA-based DSE.

We'll see how things look by the time of the LLVM 11 release.
