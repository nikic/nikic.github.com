---
layout: post
title: This year in LLVM (2024)
excerpt: Summary of my work on LLVM in 2024.
---
Another year has passed, so it's once again time for my yearly summary blog post. As usual, this summary is mostly about my own work, and only covers the more significant / higher level items.

Previous years: [2023][year_2023], [2022][year_2022]

ptradd
------

I started working on the [ptradd migration][ptradd_rfc] last year, but the first significant steps towards it only landed this year. The goal is to move away from the type-based/structural `getelementptr` (GEP) instruction, to a `ptradd` instruction, which does no more and no less than adding an offset to a pointer. The value of that offset is computed using normal mul/add instructions, if necessary.

The general implementation approach is to gradually canonicalize all getelementptrs into `getelementptr i8` representation, which is equivalent to `ptradd`, at which point we can remove support for specifying other types.

The first step was to [canonicalize constant-offset GEPs][ptradd_const_offsets]. For example, `getelementptr i32, ptr %p, i64 1` becomes `getelementptr i8, ptr %p, i64 4`, removing the implicit multiplication. Just this step already fixed a number of optimization failures caused by the old representation. Of course, it also exposed new optimization weaknesses, like [insufficient select unfolding in SROA][sroa_select].

The second step was to [canonicalize constant-expression GEPs][ptradd_const_geps] as well. That is, `getelementptr (i32, ptr @g, i64 1)` is converted to `getelementptr (i8, ptr @g, i64 4)`. However, this first required a change to how the `inrange` attribute on GEP constant expressions is represented.

```llvm
; Old:
getelementptr inbounds ({ [4 x ptr], [4 x ptr] }, ptr @vt, i64 0, inrange i32 1, i64 2)
; New:
getelementptr inbounds inrange(-16, 16) (i8, ptr @vt, i64 48)
```

`inrange` is a niche feature that is only used when generating vtables. It specifies that the GEP result can only be accessed in a limited range, which enables global splitting. Previously, this range was specified by marking a GEP index as `inrange`, limiting any accesses to occur "below" that index only. The new representation explicitly specifies the range of valid offsets instead, which removes the dependence on the structural GEP type.

The next step for this project will be to canonicalize `getelementptr` instructions with variable offsets as well, in which case we will have to emit explicit offset arithmetic. I hope we can take this step next year.

getelementptr nuw
-----------------

I have worked on a number of new instruction flags, the most significant of which are probably the [`nusw` (no unsigned-signed wrap) and `nuw` (no unsigned wrap) flags for `getelementptr`][gep_nuw_rfc].

Getelementptr instructions already supported the `inbounds` flag, which indicates that the pointer arithmetic cannot go outside the underlying allocated object. This also implies that the addition of the pointer, interpreted as an unsigned number, and the offset, interpreted as a signed number, cannot wrap around the address space.

This implied property can now be specified using the `nusw` flag, without also requiring the stronger `inbounds` property. The intention of this change is to be more explicit about which property transforms actually need, and to allow frontends (e.g. Rust) to experiment with alternative semantics that don't require the hard to formalize `inbounds` concept.

However, the much more useful new flag is `nuw`, which specifies that the addition of the address and the offset does not wrap in an unsigned sense. Combined with `nusw` (or `inbounds`), it implies that the offset is non-negative. This flag fixes two key problems.

The first are bounds/overflow check elimination failures caused by not knowing that an offset is non-negative. For example, if we have a check like `ptr + offset >= ptr`, we want it to optimize to `true`, for the case where `offset` is unsigned and pointer arithmetic cannot wrap. Previously this was not possible, because the frontend had no way to convey that this is a valid optimization to LLVM.

The second is accesses to structures like `struct vec { size_t len; T elems[N]; }`. The `nuw` flag allows us to convey that `vec.elems[i]` cannot access the `len` field using a negative `i`.

The core work for the new GEP flags is complete. Both alias analysis and comparison simplification can take advantage of them. However, there is still a good bit of work that can be done to preserve the new flags in all parts of the compiler.

One interesting complication of the `nuw` flag is the interaction with incorrectly implemented overflow checks in C code. The `ptr + offset >= ptr` example above is something we want to be able to optimize, because this kind of pattern can, for example, appear when using a hardened STL implementation.

However, a C programmer might also write literally that check with the intention of detecting an overflowing pointer addition. This is incorrect, because `ptr + offset` will already trigger UB on overflow (or, in fact, just going out of bounds of the underlying object). These incorrect overflow checks will now be optimized away.

I have [extended][tautological_compare] the `-Wtautological-compare` warning to catch trivial cases of this problem, and the `-fsanitize=pointer-overflow` warning catches the issue reliably -- if you have test coverage.

trunc nuw/nsw
-------------

Another new set of instruction flags are the [`nuw` and `nsw` flags on `trunc`][trunc_flags]. These flags specify that only zero bits (nuw) or sign bits (nsw) are truncated. The important optimization property of these new flags is that `zext (trunc nuw (x))` is just `x` and `sext (trunc nsw (x))` is also just `x`.

These kinds of patterns commonly occur when booleans are converted between their i1 value and i8 memory representation, or during widening of induction variables.

Unfortunately, rolling out these flags did not go as smoothly as expected. While we do take advantage of the new flags to some degree, we don't actually perform the motivating `zext (trunc nuw (x)) -> x` fold yet, and instead keep folding this to `and x, 1` (for i1). The reason is that completely eliminating the zext/trunc loses the information that only the low bit may be set. We need to strengthen some other optimizations before we'll be able to take this step.

icmp samesign
-------------

Yet another new instruction flag is [`samesign` on `icmp` instructions][samesign_rfc]. As one might guess from the name, it indicates that both sides of the comparison have the same sign, i.e. are both non-negative or both negative.

If `samesign` is set, we can freely convert between signed and unsigned comparison predicates. The motivation is that LLVM generally tries to canonicalize signed to unsigned operations, including for comparisons. So something like `icmp slt` will become `icmp ult` if we can prove the operands have the same sign. However, after this has happened, later passes may have a hard time understanding how this unsigned comparison relates to another signed comparison that has related operands. The `samesign` flag allows easily converting back to a signed predicate when needed.

The bring-up work for `samesign` has only just started, so there's barely any visible optimization impact yet.

Improvements to capture tracking
--------------------------------

Capture tracking, also known as [escape analysis][escape_analysis], is critical for the quality of memory optimizations. Once a pointer has "escaped", we can no longer accurately track when the pointer may be accessed, disabling most optimizations.

LLVM currently provides a `nocapture` attribute, which indicates that a call does not capture a certain parameter. However, this is an all-or-nothing attribute, which does not allow us to distinguish several different *kinds* of captures.

I have [proposed][captures_rfc] to replace `nocapture` with a `captures(...)` attribute, which allows a finer-grained specification. In particular, it separates capturing the *address* of the pointer, which is information about its integral value, and capturing the *provenance* of the pointer, which is the permission to perform memory accesses through the pointer. Provenance captures are what we usually call an "escape", and only provenance captures are relevant for the purposes of alias analysis.

Address and provenance are often captured together, but not always. For example, a pointer comparison only captures the address, but not the provenance. Rust makes address-only capture particularly clear with its [strict provenance][strict_provenance] APIs, where `ptr.addr()` only returns the address of the pointer, without its provenance.

The new `captures` attribute additionally allows specifying whether the full provenance is captured, or only read-only provenance. For example, a `&` (Freeze) reference argument in Rust could use `readonly captures(address, read_provenance)` attributes to indicate that not only does the function not modify the argument directly, it also can't stash it somewhere and perform a modification through it after the function returns. As many optimizations are only inhibited by potential writes ("clobbers"), this is an important distinction.

I have implemented the [first step][captures_pr] for this proposal and plan to continue this in the new year. Furthermore, this is only the first step towards improving capture analysis in LLVM. Some followups I have in mind are:

 * Support `captures` attribute (or something similar) on `ptrtoint` instructions. This would allow us to accurately represent the semantics of `ptr.add()` in Rust.
 * Support `!captures` metadata on stores. This would allow us to indicate that values passed to `println!()` in Rust are read-only escapes.
 * Add a first-class operation for pointer subtraction, which is currently done through `ptrtoint`.

Three-way comparison intrinsics
-------------------------------

Three-way comparisons return the comparison result as one of -1 (smaller), 0 (equal) or 1 (greater). Historically, LLVM did not have a canonical representation for such comparisons, instead they could be represented using a large number of different instruction sequences, none of which produced ideal optimization outcomes and codegen for all targets.

I [proposed][3way_rfc] adding three-way comparison intrinsics `llvm.ucmp` and `llvm.scmp` to represent such comparisons. [Volodymyr Vasylkun][poseydon] implemented these as a GSoC project, including high-quality codegen, various middle-end optimizations and canonicalization to the new intrinsics.

There is a short [blog post][3way_blog] on the topic, which is worth reading, and has an example illustrating how codegen improves for the C++ spaceship operator `<=>`.

APInt assertions
----------------

When constructing an arbitrary-precision integer (APInt) from a `uint64_t`, we previously performed implicit truncation: If the passed value was larger than the specified bit width, we'd just drop the top bits. Sometimes, this is exactly what we want, but in other cases it hides a bug.

A particular common issue is to write something like `APInt(BW, -1)` without setting the `isSigned=true` flag. This would work correctly for bit widths <= 64, and then produce incorrect results for larger integers, which have much less coverage (both in terms of tests and in-the-wild usage).

The APInt constructor now [asserts][apint_assert] that the passed value is a valid N-bit signed/unsigned integer. It took a good while to get there, because a lot of existing code had to be adjusted.

This work is not fully complete yet, because `ConstantInt::get()` still enables implicit truncation. Changing this will require more work to adjust existing code.

Quality of life improvements
----------------------------

I have made a number of quality of life improvements when it comes to things I do often. Part of this are [improvements to the IR parser][ir_parser_improvements] in three areas:

 * Do not require declarations or type mangling for intrinsics. You can now write `call i32 @llvm.smax(i32 %x, i32 %y)` and the parser will automatically determine the correct type mangling and insert the intrinsic declaration. This removes a big annoyance when writing proofs.
 * Don't require unnamed values to be consecutive. This makes it much easier to reduce IR with unnamed values by hand.
 * Allow parsing of incomplete IR under the `-allow-incomplete-ir` option. This makes it easy to convert things like `-print-after-all` output into valid IR.

Additionally, I have added support for [pretty crash stacks][pretty_stack] for the new pass manager. This means that crashes now print which pass (with which options) crashed on which function.

I have also written an [InstCombine contributor guide][instcombine_contributor_guide]. This helps new contributors write good PRs for InstCombine (and the middle-end in general), which makes the review process a lot more efficient.

Another improvement to reviewer life was to [remove complexity-based canonicalization][complexity_canon]. While well-intentioned, in practice this transform just made it hard to write test coverage that does what you intended, which especially new contributors often struggled with. This removes the need for the "thwart" test pattern.

Compilation-time
----------------

<!--
https://llvm-compile-time-tracker.com/graphs.php?startDate=2024-01-01&interval=25&relative=on&bench=geomean&width=800
legend: "always",
left: 5.5em !important;
-->

I have not done much work on compile-time myself this year, but other people have picked up the slack, so I'll provide a summary of their work. Here is how compile-time developed since the start of 2024:

<img src="/images/llvm_geomean_2024.png" alt="LLVM geomean instruction count changes since January 2024" style="max-width: 100%" />

The largest wins this year are in the `ReleaseLTO-g` configuration (about 10%). A big reason for this is the [switch from debug intrinsics to debug records][dbg_records_rfc], which moves debug information out of the instruction list. This benefits all builds with debuginfo, but particularly optimized debuginfo builds.

The two commits switching to the new representation amount to about 4% improvement ([1][dbg_records_ct], [2][dbg_records_ct2]). There will be some additional improvements once support for debug intrinsics is removed.

Unoptimized `O0-g` configurations saw an improvement of about 4-6% this year, mostly thanks to a concerted effort by [aengelke][aengelke] to improve unoptimized build times for JIT use cases. 

One of the largest individual wins was [adding a fast-path][regalloc_pr] to RegAllocFast, the O0 register allocator, for a [0.8% improvement][regalloc_ct]. Caching the [register alias iterator][mc_reg_alias_pr] also produced a [0.6-0.8% improvement][mc_reg_alias_ct]. Not computing [unnecessary symbol names][mc_opt_unnamed_pr] gave a [0.4% improvement][mc_opt_unnamed_ct] and using a bump-pointer allocator [for MCFragments][mc_opt_bumpptr_pr] gave a [0.5% improvement][mc_opt_bumpptr_ct]. Though that one came at a cost of increased max-rss, so I'm not sure if it was a good tradeoff. There were also many other small improvements, that add up to a large overall improvement.

One regression for unoptimized builds was to [stop using `-mrelax-all`][mrelax_all_pr]. While this was a [0.6% compile-time regression][mrelax_all_ct], it also [decreased binary size by 4-5%][mrelax_all_pr], which is a pretty good deal.

Optimized builds also saw wins of about 5-7%. One of the key changes was the [introduction of block numbers][block_num_pr], which assign a unique number to each basic block in a function. These can then be used to map blocks to dominator tree nodes ([1][dt_num_pr], [2][ir_num_pr]) using a vector instead of a hashtable, significantly reducing the cost of accesses. This results in a ~1.2% improvement ([1][dt_num_ct], [2][ir_num_ct]), and will probably be useful for more things in the future.

Another interesting changes is to not [rerun the same pass][analysis_rerun_pr] if the IR did not change in the meantime, for a [0.5% improvement][analysis_rerun_ct]. I think this is currently less effective than it could be, because we have some passes that keep toggling the IR between two forms (e.g. adding and removing LCSSA phi nodes).

A somewhat different compile-time improvement was a change to clang's [bitfield codegen][bitfield_pr]. This resulted in a [0.3-0.6% improvement][bitfield_ct] on stage 2 builds (where clang is compiled with clang). It's an optimization improvement that has a large enough impact to affect compilation times.

There was also a change that improves the time to build clang, which was the introduction of [DynamicRecursiveASTVisitor][dyn_ast_visitor_pr], and migration of most visitors to use it. This switches many AST visitors away from huge CRTP template instantiations to using dynamic dispatch. The PR switching most visitors [improved][dyn_ast_visitor_ct] clang build times by 2% and the size of the resulting binary by nearly 6% (!). This does come at a small compile-time cost for small files, due to the increase in dynamic relocations.

To close with some optimizations I actually did myself, I implemented a series of improvements to the SmallPtrSet data structure ([1][smallptrset_pr1], [2][smallptrset_pr2], [3][smallptrset_pr3]), to make sure that the fast-path is as efficient as possible. These add up to a 0.7% improvement ([1][smallptrset_ct1], [2][smallptrset_ct2], [3][smallptrset_ct3]).

I also made a series of improvements to `ScalarEvolution::computeConstantDifference()` to make it usable in place of `ScalarEvolution::getMinusSCEV()` during SLP vectorization. This was a [10% improvement][const_diff_ct] on one specific benchmark.

Rust
----

I updated Rust to [LLVM 18][rust_llvm_18] and [LLVM 19][rust_llvm_19] this year. Both of these came with very nice perf results ([LLVM 18][rust_llvm_18_perf], [LLVM 19][rust_llvm_19_perf]).

The LLVM 18 upgrade had quite a few complications, but probably the most disproportionately annoying was the fact that the LLVM soname changed from `libLLVM-18.so` to `libLLVM.so.18.1`, with `libLLVM-18.so` being a symlink now. Threading the needle between all the constraints coming from Rust dylib linking, rustup and LLVM, this ultimately required converting `libLLVM-18.so` into a linker script.

The LLVM 19 upgrade went pretty smoothly. One change worth mentioning is that LLVM 19 enabled the `+multivalue` and `+extern-types` wasm features by default. We could have handled this in various ways, but the decision was that Rust's wasm32-unknown-unknown target should continue following Clang/LLVM defaults over time. There is a [blog post][wasm_features] with more information.

We have also reached the point where we align with upstream LLVM very closely: We only carry a single permanent patch, and most backports are now handled via upstream patch releases, instead of Rust-only backports.

I also implemented what has to be my lowest-effort compile-time win ever, which is to [disable LLVM IR verification][verify_ir_pr] in production builds more thoroughly. This is a [large win][verify_ir_perf] for debug builds.

Finally, I presented a keynote at this year's LLVM developer meeting with the title "Rust ❤️ LLVM" ([slides][keynote_slides], [recording][keynote_video]), discussing how Rust uses LLVM and some of the challenges involved. I had a lot of fruitful discussions on the topic as a result. The only downside is that people now think I know a lot more about Rust than I actually do...

I'd also like to give a shout-out to [DianQK][DianQK], who has helped a lot with tracking down and fixing LLVM bugs that affect Rust.

Packaging
---------

This year, the way LLVM is packaged for Fedora, CentOS Stream and RHEL underwent some major changes. This is something the entire LLVM team at Red Hat worked on to some degree.

The first, and most significant, is that we now build multiple LLVM subprojects (llvm, clang, lld, lldb, compiler-rt, libomp, and python-lit) as part of a single build, while keeping the separate binary RPMs. Previously, we used standalone builds for each subproject.

While standalone builds have their upsides (like faster builds if you're only changing one subproject), they are very much a second class citizen in upstream LLVM, and typically only used by distros. Using a monorepo build moves us to a better supported configuration.

Additionally, it simplifies the build process, because we previously had to do a coordinated rebuild of 14 different packages for each LLVM point release. For LLVM 19, we're down to a single build on RHEL and 8 builds on Fedora, which ships more subprojects. For LLVM 20 we should get down to 4 builds, as we merge more subprojects into the monorepo build.

The second change is related to the nightly [snapshot builds][snapshots_copr] we have been offering for a while already. These are now maintained directly on the Fedora rawhide branch of [rpms/llvm][llvm_rpm], which supports building both the current LLVM version (19) and snapshots for the upcoming one (20) from the same spec file. This avoids divergence between the two branches, and (in theory) reduces an LLVM major version upgrade to a change in the build configuration.

The final change is to consolidate the spec file for all operating systems we build for. The Fedora rawhide spec file can also be used to build for RHEL 8, 9 and 10 now. Again, this helps to prevent unintentional divergence between operating systems. Additionally, it means that LLVM snapshot builds are now available for RHEL as well.

Other
-----

I have been nominated as the new [lead maintainer][lead_maintainer] for LLVM, taking over from Chris Lattner. What does this mean? Even more code review, of course! According to graphite, I've reviewed more than 2300 pull requests last year. I have also spent some time getting LLVM's badly outdated [list of maintainers][maintainers] up to date (though this isn't entirely complete yet).

[N3322][n3322], the C proposal I worked on together with Aaron Ballman, has been accepted for C2y, [making memcpy(NULL, NULL, 0) well-defined][memcpy_blog].

To close this blog post, I'd like to thank [dtcxzyw][dtcxzyw] for his incredible work on LLVM, both in identifying and fixing numerous miscompilation issues, and implementing many optimizations with demonstrable usefulness. [llvm-opt-benchmark][llvm_opt_benchmark] has quickly become an indispensable tool for analyzing optimization impact.

[ptradd_rfc]: https://discourse.llvm.org/t/rfc-replacing-getelementptr-with-ptradd/68699
[ptradd_const_offsets]: https://github.com/llvm/llvm-project/pull/68882
[ptradd_const_geps]: https://github.com/llvm/llvm-project/pull/89872
[inrange]: https://github.com/llvm/llvm-project/pull/84341
[sroa_select]: https://github.com/llvm/llvm-project/pull/80983

[gep_nuw_rfc]: https://discourse.llvm.org/t/rfc-add-nusw-and-nuw-flags-for-getelementptr/78672
[gep_nuw_impl]: https://github.com/llvm/llvm-project/pull/90824
[tautological_compare]: https://github.com/llvm/llvm-project/pull/120222
[trunc_flags]: https://discourse.llvm.org/t/rfc-add-nowrap-flags-to-trunc/77453
[samesign_rfc]: https://discourse.llvm.org/t/rfc-signedness-independent-icmps/81423

[escape_analysis]: https://en.wikipedia.org/wiki/Escape_analysis
[captures_rfc]: https://discourse.llvm.org/t/rfc-improvements-to-capture-tracking/81420
[captures_pr]: https://github.com/llvm/llvm-project/pull/116990
[strict_provenance]: https://doc.rust-lang.org/beta/std/ptr/index.html#strict-provenance

[3way_rfc]: https://discourse.llvm.org/t/rfc-add-3-way-comparison-intrinsics/76685
[3way_blog]: https://blog.llvm.org/posts/2024-08-29-gsoc-three-way-comparison/
[poseydon]: https://github.com/Poseydon42

[dbg_records_ct]: https://llvm-compile-time-tracker.com/compare.php?from=8faefe36ed57c2dab2b50e76fd27045b908f8c1d&to=a93a4ec7dd205b965ee5597314bb376520cd736c&stat=instructions:u
[dbg_records_ct2]: https://llvm-compile-time-tracker.com/compare.php?from=b0d03ccc0855f2bff39160f25fcde06aae07cace&to=cbbbab349e9c412729c3969008cdcb677cc55790&stat=instructions:u
[dbg_records_rfc]: https://discourse.llvm.org/t/rfc-instruction-api-changes-needed-to-eliminate-debug-intrinsics-from-ir/68939

[aengelke]: https://github.com/aengelke
[regalloc_ct]: https://llvm-compile-time-tracker.com/compare.php?from=b1ec1a2dc81075eceddd2c6b34b52d2a741fd961&to=0ae6cfc5990b0b739166bd7db370125ca66494c2&stat=instructions:u
[regalloc_pr]: https://github.com/llvm/llvm-project/pull/96284
[mc_reg_alias_ct]: https://llvm-compile-time-tracker.com/compare.php?from=88e42c6779067c4b65624939be74db2d56ee017b&to=ab0d01a5f0f17f20b106b0f6cc6d1b7d13cf4d65&stat=instructions:u
[mc_reg_alias_pr]: https://github.com/llvm/llvm-project/pull/93510
[mc_opt_unnamed_ct]: https://llvm-compile-time-tracker.com/compare.php?from=23a33a72c620f7e37778fa26af46e88a0042cd15&to=43229977fe0f74bcdfeda20e478065a2072a1846&stat=instructions:u
[mc_opt_unnamed_pr]: https://github.com/llvm/llvm-project/pull/95021
[mc_opt_bumpptr_ct]: https://llvm-compile-time-tracker.com/compare.php?from=fc23564c44f3eff1847462253d43c08b85489148&to=8cb6e587fd40b97983e445ee3f17f4c6d7190df7&stat=instructions:u
[mc_opt_bumpptr_pr]: https://github.com/llvm/llvm-project/pull/96402
[mc_opt__ct]: https://llvm-compile-time-tracker.com/compare.php?from=a091bfe71fdde4358dbfc73926f875cb05121c00&to=75006466296ed4b0f845cbbec4bf77c21de43b40&stat=instructions:u
[block_num_pr]: https://github.com/llvm/llvm-project/pull/101052
[dt_num_pr]: https://github.com/llvm/llvm-project/pull/101706
[dt_num_ct]: https://llvm-compile-time-tracker.com/compare.php?from=f949b036610afe56fddde724ee01f64dd79814d3&to=1d2b6d9d4d1074bac4a6ec48dd0ff4253590e34a&stat=instructions:u
[ir_num_pr]: https://github.com/llvm/llvm-project/pull/102758
[ir_num_ct]: https://llvm-compile-time-tracker.com/compare.php?from=2b077ede083b4185f51a2fe648a27e4c85352a2f&to=8fc3a7974701f12f46f3f7c1f967001b0cb22847&stat=instructions:u
[analysis_rerun_pr]: https://github.com/llvm/llvm-project/pull/112092
[analysis_rerun_ct]: https://llvm-compile-time-tracker.com/compare.php?from=f6617d65e496823c748236cdbe8e42bf4c8d8a55&to=cacbe71af7b1075f8ad1f84e002d1fcc83e85713&stat=instructions:u
[bitfield_ct]: https://llvm-compile-time-tracker.com/compare.php?from=7df79ababee8d03b27bbaba1aabc2ec4ea14143e&to=49839f97d2951e0b95d33aee00f00022952dab78&stat=instructions:u
[bitfield_pr]: https://github.com/llvm/llvm-project/pull/65742
[dyn_ast_visitor_pr]: https://github.com/llvm/llvm-project/pull/110040
[dyn_ast_visitor_ct]: https://llvm-compile-time-tracker.com/compare.php?from=3d57c79728968e291df4929b377b3580d16af7b9&to=dde802b153d5cb41505bf4d377be753576991297&stat=instructions:u
[mrelax_all_ct]: https://llvm-compile-time-tracker.com/compare.php?from=ef2ca97f48f1aee1483f0c29de5ba52979bec454&to=18376810f359dbd39d2a0aa0ddfc0f7f50eac199&stat=instructions:u
[mrelax_all_size]: https://llvm-compile-time-tracker.com/compare.php?from=ef2ca97f48f1aee1483f0c29de5ba52979bec454&to=18376810f359dbd39d2a0aa0ddfc0f7f50eac199&stat=size-text
[mrelax_all_pr]: https://github.com/llvm/llvm-project/pull/90013
[smallptrset_ct1]: http://llvm-compile-time-tracker.com/compare.php?from=8a7730fb88445a019fe150d5db4f6642e43afd04&to=42c3edb4819ff2e9608f645fb5793dcb33b47f9c&stat=instructions%3Au
[smallptrset_ct2]: https://llvm-compile-time-tracker.com/compare.php?from=0ad6be1927f89cef09aa5d0fb244873f687997c9&to=6bfb6d4092a284e1fcd135625c0e713d019f0572&stat=instructions:u
[smallptrset_ct3]: https://llvm-compile-time-tracker.com/compare.php?from=a348f223cab54b21a7b1c38dec7bc6aa2f81c949&to=a8a494faab8af60754c4647dbb7b24bc86a80aab&stat=instructions:u
[smallptrset_pr1]: https://github.com/llvm/llvm-project/pull/96762
[smallptrset_pr2]: https://github.com/llvm/llvm-project/pull/118092
[smallptrset_pr3]: https://github.com/llvm/llvm-project/pull/118099
[const_diff_ct]: https://llvm-compile-time-tracker.com/compare.php?from=65390f9d6f0aa5f7bc5125d73337eb658e162d0a&to=6a84af704f57defd919a4ec2e34b70a48d548719&stat=instructions:u

[rust_llvm_18]: https://github.com/rust-lang/rust/pull/120055
[rust_llvm_18_perf]: https://perf.rust-lang.org/compare.html?start=bc1b9e0e9a813d27a09708b293dc2d41c472f0d0&end=eaff1af8fdd18ee3eb05167b2836042b7d4315f6&stat=instructions:u
[rust_llvm_19]: https://github.com/rust-lang/rust/pull/127513
[rust_llvm_19_perf]: https://perf.rust-lang.org/compare.html?start=e552c168c72c95dc28950a9aae8ed7030199aa0d&end=0b5eb7ba7bd796fb39c8bb6acd9ef6c140f28b65&stat=instructions:u
[wasm_features]: https://blog.rust-lang.org/2024/09/24/webassembly-targets-change-in-default-target-features.html
[verify_ir_pr]: https://github.com/rust-lang/rust/pull/133499
[verify_ir_perf]: https://perf.rust-lang.org/compare.html?start=4af7fa79a0e829c0edcc93434a8c788be8ec58c6&end=8ac313bdbede661669d7a7b4504b0f74d4ed9222&stat=instructions:u
[keynote_slides]: /pdf/slides_llvm_dev_meeting24_rust_heart_llvm.pdf
[keynote_video]: https://www.youtube.com/watch?v=Kqz-umsAnk8

[snapshots_copr]: https://copr.fedorainfracloud.org/coprs/g/fedora-llvm-team/llvm-snapshots
[llvm_rpm]: https://src.fedoraproject.org/rpms/llvm

[apint_assert]: https://github.com/llvm/llvm-project/pull/106524

[ir_parser_improvements]: https://discourse.llvm.org/t/recent-improvements-to-the-ir-parser/77366
[pretty_stack]: https://github.com/llvm/llvm-project/pull/96078
[instcombine_contributor_guide]: https://llvm.org/docs/InstCombineContributorGuide.html
[complexity_canon]: https://github.com/llvm/llvm-project/pull/91185

[year_2022]: https://www.npopov.com/2022/12/20/This-year-in-LLVM-2022.html
[year_2023]: https://www.npopov.com/2024/01/01/This-year-in-LLVM-2023.html

[lead_maintainer]: https://discourse.llvm.org/t/rfc-proposing-a-new-lead-maintainer-for-llvm/81290
[maintainers]: https://github.com/llvm/llvm-project/blob/main/llvm/Maintainers.md
[n3322]: https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3322.pdf
[memcpy_blog]: https://developers.redhat.com/articles/2024/12/11/making-memcpynull-null-0-well-defined
[DianQK]: https://github.com/DianQK
[dtcxzyw]: https://github.com/dtcxzyw
[llvm_opt_benchmark]: https://github.com/dtcxzyw/llvm-opt-benchmark

<!-- Ended up not using these -->
[bumpptr_ct]: https://llvm-compile-time-tracker.com/compare.php?from=d392520c645b653cd9c2ce944958fb115c4ba506&to=cd46c2c1ba0481e2194231f0f2c2ceeb0810bb79&stat=instructions:u
[sized_dealloc_ct]: https://llvm-compile-time-tracker.com/compare.php?from=b66779b5bf0f3c839114681bb4aca80a9dc144c3&to=130e93cc26ca9d3ac50ec5a92e3109577ca2e702&stat=instructions:u
[hash_seed_ct]: https://llvm-compile-time-tracker.com/compare.php?from=982c54719289c1d85d03be3ad9e95bbfd2862aee&to=ce80c80dca45c7b4636a3e143973e2c6cbdb2884&stat=instructions:u
[module_flag_ct]: https://llvm-compile-time-tracker.com/compare.php?from=1d2b6d9d4d1074bac4a6ec48dd0ff4253590e34a&to=b7cd564fa3ecc2a9ed0fded98c24f68e2dad63ad&stat=instructions:u
[smalldensemap_ct]: https://llvm-compile-time-tracker.com/compare.php?from=d9250061e10b82f82d9833009f6565775578ee58&to=056a3f4673a4f88d89e9bf00614355f671014ca5&stat=instructions:u
[string_tables_ct]: https://llvm-compile-time-tracker.com/compare.php?from=f6c51ea84ac914454142ee76f317c5f66a088434&to=be2df95e9281985b61270bb6420ea0eeeffbbe59&stat=instructions:u

