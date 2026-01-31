---
layout: post
title: This year in LLVM (2025)
excerpt: Summary of my work on LLVM in 2025.
---
It's 2026, so it's time for my yearly summary blog post. I'm a bit late, but at least it's still January! As usual, this summary is about my own work, and only covers the more significant / higher-level items.

Previous years: [2024][year_2024], [2023][year_2023], [2022][year_2022]

ptradd
------

I have been making slow progress on the [ptradd migration][ptradd_rfc] over the last three years. The goal of this change is to move away from the type-based `getelementptr` (GEP) representation, towards a `ptradd` instruction, which just adds an integer offset to a pointer.

The state at the start of the year was that constant-offset GEP instructions were canonicalized to the form `getelementptr i8, ptr %p, i64 OFFSET`, which is equivalent to a `ptradd`.

The progress this year was to canonicalize all GEP instructions to have a single offset. For example, `getelementptr [10 x i32], ptr %p, i64 %a, i64 %b` gets split into two instructions now. This moves us closer to `ptradd`, which only accepts a single offset argument. However, the change is also independently useful, because it allows CSE of common GEP prefixes.

This work happened in multiple phases, first splitting [multiple variable indices][gep_split_var], then splitting off [constant indices as well][gep_split_all] and finally removing [leading zero indices][gep_leading_zeros].

As usual, the bulk of the work was not in the changes themselves, but in mitigating resulting regressions. Many transforms were extended to work on chains of GEPs rather than only a single one. Once again, this is also useful independently of the ptradd migration, as chained GEPs were already very common beforehand.

There are still some major remaining pieces of work to complete this migration. The first one is to decide whether we want `ptradd` to support a constant scaling factor, or require it to be represented using a separate multiplication. There are good arguments in favor of both options.

The second one is to move from mere canonicalization towards requiring the new form. This would probably involve first making IRBuilder emit it, and then actually preventing construction of the type-based form. That would be the point where we'd actually introduce the `ptradd` instruction.

ptrtoaddr
---------

LLVM 22 [introduces][ptrtoaddr_pr] a new `ptrtoaddr` instruction. This is the outcome of a [long discussion][ptrtoaddr_rfc] on the semantics of `ptrtoint` and pointer comparisons for [CHERI architectures][cheri_wiki].

The semantics of `ptrtoaddr` are similar to `ptrtoint`, but differ in two respects:

 * It does not expose the provenance of the pointer. In Rust terms, it corresponds to `addr()` instead of `expose_provenance()`.
 * It returns only the address portion of the pointer. This matters for CHERI, where pointers also carry additional metadata bits.

A non-exposing way to convert a pointer into an integer is an important step towards figuring out LLVM's provenance story. LLVM currently ignores the fact that `ptrtoint` has an (exposure) side-effect, and having a side-effect-free alternative is one of the prerequisites to actually taking this seriously. (The other is the [byte type][byte_type_rfc].)

The downside of having two instructions that do something similar but not quite the same is that it requires careful adjustment of existing optimizations to work on both forms, where possible. This is something I have been working on, and `ptrtoaddr` should now be supported in most of the important optimizations.

Lifetime intrinsics
-------------------

LLVM represents stack allocations using `alloca` instructions. These are generally always placed inside the entry block, while the actual lifetime of the allocation is marked using [`lifetime.start`][lifetime_start] and [`lifetime.end`][lifetime_end] intrinsics. The primary purpose of these intrinsics is to enable stack coloring, which can place stack allocations that are not live at the same time at the same address, greatly reducing stack usage.

I have made two major changes to lifetime intrinsics: The first is to enforce that they are [only used with allocas][lifetime_alloca_only]. Previously, it was possible to use them on arbitrary pointers, such as function arguments. This is incompatible with stack coloring, which requires that all lifetime markers for an allocation are visible -- they can't be hidden behind a function call.

Making this an IR validity requirement was helpful in uncovering quite a few cases where we ended up using lifetimes on non-allocas by mistake, as a result of optimization passes. Most commonly, the alloca was accidentally obscured by phi nodes.

The second change was to [remove the size argument][lifetime_alloca_no_size] from lifetime intrinsics. In theory, this argument allowed you to control the lifetime of a subset of the allocation. In practice, this was never used, and stack coloring just ignored the argument. This was a smaller change in terms of IR semantics, but significantly larger in impact because it required updates to all code and tests involving lifetime intrinsics.

While these changes have resolved some issues with our handling of lifetimes, more problems (with [store speculation][lifetime_problem_licm] and [comparisons][lifetime_problem_icmp]) remain. A core issue is that in the current representation, it's not possible to efficiently determine whether an alloca is live at a given point, or whether the lifetime of two allocas can overlap. Fixing this requires more intrusive changes.

Capture tracking
----------------

Another piece of work that carried over from the previous year are improvements to capture tracking. I [proposed][captures_rfc] this last year, but the majority of the implementation work happened this year.

The most important part of this proposal is that we now distinguish between capturing the address of a pointer, and its provenance. Many optimizations only care about the latter, because only provenance capture may result in non-analyzable memory effects.

The most significant changes to enable this were [inference support][captures_inference], and updating alias analysis to only check for [provenance captures][captures_aa_prov] and make use of [read-only captures][captures_aa_read_only].

I've also extended this feature by adding [`!captures` metadata][captures_metadata] on stores. This is intended to allow encoding that stores of non-mut references in Rust only capture read provenance, which is helpful to optimize around constructs like `println!()`, which capture via memory rather than function arguments. Whether we can actually do this depends on an [open question][mir_retags] in Rust's aliasing model.

ABI
---

One of the biggest failures of LLVM as an abstraction across different target architectures is its handling of platform ABIs, in the sense of calling conventions (CC). A large part of the ABI handling has to be performed in the frontend, which currently means that every frontend with C FFI support has to reimplement complex and subtle ABI rules for all targets it supports.

To ameliorate this, I have proposed an [ABI lowering library][abi_library_rfc], which tells frontends how to correctly lower a given function signature that is provided using a separate ABI type system, which is richer than LLVM IR types, but much simpler than Clang QualTypes.

As part of GSoC, [vortex73][vortex73] has implemented a [prototype][abi_prototype] for such a library. It demonstrates that the general approach works for the x86-64 SystemV ABI (one of the more complex ones), without significant overhead. For more information, see the accompanying [blog post][abi_library_blog]. Work to upstream this library is underway.

Another pain point is that information on type alignment is duplicated between Clang and LLVM (for layering reasons), and this information can get out of sync. This causes issues for frontends like Rust, which use the LLVM information. I've implemented some [consistency checks][datalayout_consistency] to prevent more of these issues in the future. I've also [removed][datalayout_deduplication] the duplicate data layout definitions between Clang and LLVM.

I've also done some work to improve the backend side of things, by exposing the [original, unlegalized argument type][cc_orig_ir_type] to CC lowering. This allowed cleaning up lots of target-specific hacks, like MIPS' hardcoded list of [fp128 libcalls][mips_f128_libcalls].

ConstantInt assertions
----------------------

The previous year, I introduced an assertion when constructing arbitrary-precision integers (APInts) from `uint64_t`, which ensures that the value actually is an N-bit signed/unsigned integer. The purpose of this assertion is to avoid miscompiles due to incorrectly specified signedness, which only manifests for large integers (with more than 64 bits).

Back then, I excluded the `ConstantInt::get()` constructor from this assertion to reduce the (already very large) scope of the work. I ended up regretting that when I hit a [SelectOptimize miscompile][selectopt_bug], which is caused by precisely the problem this assertion is supposed to prevent.

That was enough motivation to [extend][constantint_assert] the assertion to `ConstantInt::get()`. Once again, this required substantial work to fix existing issues (most of which were harmless, but I've caught at least two more miscompiles along the way).

Compilation-time
----------------

<!--
http://localhost:8000/graphs.php?startDate=2025-01-01&endDate=2026-01-01&interval=25&relative=on&bench=geomean&width=800&configs=stage2-O3,stage2-O0-g
legend: "always",
top: 16em !important;
left: 5.5em !important;
-->

I have done little compile-time work this year, and there hasn't been much activity from other people either. Here's how compile-time developed over the course of 2025:

<img src="/images/llvm_geomean_2025.png" alt="LLVM geomean instruction count changes since January 2025" style="max-width: 100%" />

I have only included two configurations in the graph, because it is getting quite cluttered otherwise. Historically, I have only been tracking compile-times on x86, but have added two AArch64 configurations this year. An interesting takeaway from this is that compilation for AArch64 is around 10-20% slower, depending on configuration. For unoptimized builds, this is due to use of GlobalISel instead of FastISel. For optimized builds, use of alias analysis during codegen is a significant factor.

In terms of optimizations, I've implemented an improvement to [SCCP worklist management][sccp_worklist] ([~0.25% improvement][sccp_worklist_ct]), which reduces the number of times instructions are visited during sparse conditional constant propagation. I've introduced a [getBaseObjectSize()][base_obj_size] function ([~0.35% improvement][base_obj_size]) to avoid use of expensive `__builtin_object_size` machinery where it is not needed. I've also specialized calculation of [type allocation sizes][type_alloc_size] ([~0.25% improvement][type_alloc_size_ct]) to reduce redundant operations.

I'd also like to highlight two changes from other contributors. One was to optimize [debug linetable emission][debug_linetable], by avoiding the creation of unnecessary fragments. This improved debug builds by [~1%][debug_linetable_ct]. Another was to change the representation of [nested name specifiers][name_specifier] in the Clang AST. I have no idea what this is doing, but it improved Clang build time by [~2.6%][name_specifier_ct], so this has a big impact on C++ heavy projects.

I'm especially happy about the Clang change, as Clang is our main source of unmitigated compile-time regressions.

Optimizations
-------------

I don't tend to do much direct optimization work: If the optimization does not require significant IR or infrastructure changes, we have plenty of other people who can work on it. But sometimes I can't resist, so here are a couple of the more interesting optimizations I worked on.

I've implemented a [store merge][store_merge] optimization, which combines multiple stores into a single larger store. LLVM already had some support for this in the backend, but it was rather hit and miss. The reason I worked on this is that someone [on Reddit][reddit_codegen_gcc] shared an example where Rust's GCC backend actually produced better code than the LLVM backend, which is an injustice I just could not let stand.

I've enabled the [use of PredicateInfo][sccp_predicateinfo][^predicateinfo] in non-inter-procedural SCCP (sparse conditional constant propagation). This enables reliable optimization based on constant ranges implied by branches and assumptions. Previously we only handled this during inter-procedural SCCP, which runs very early, and CVP (correlated value propagation), which is based on LVI (lazy value info) and has problems dealing with loops[^lvi_loops]. The main work here went into speeding up PredicateInfo, but we still had to eat a ~0.1% compile-time regression in the end.

Finally, I've implemented a pass to [drop assumes][drop_assumes] that are unlikely to be useful anymore. This partially addresses a recurring problem where adding more assumes degrades optimization quality. This is just a starting point, we should likely be dropping assumes with various degrees of aggressiveness at multiple pipeline positions.

Rust
----

As usual, I've updated Rust to use [LLVM 20][rust_llvm20] and then [LLVM 21][rust_llvm21]. Similar to all recent updates, this came with compile-time improvements ([LLVM 20][rust_llvm20_perf], [LLVM 21][rust_llvm21_perf]).

Both updates went relatively smoothly. LLVM 21 ran into a BOLT instrumentation miscompile that defied local reproduction, but luckily an update of the host toolchain fixed it.

With these updates we were able to use a number of new LLVM features, some of which were added specifically for use by Rust.

The most significant is the use of [read-only captures][rust_read_only_captures] for non-mutable references. This lets LLVM know that not only can't the function modify the memory, but it also can't be modified through captured pointers after the call. This further increases the reliability of memory optimizations in Rust relative to C++.

Another is the use of the [`alloc-variant-zeroed`][rust_alloc_variant_zeroed] attribute, which enables optimization of `__rust_alloc` + memset zero to `__rust_alloc_zeroed`. This ended up running into some LTO issues that required follow-up changes to fix attribute emission for allocator definitions.

We're also marking by value arguments as [`dead_on_return`][rust_dead_on_return] now, and using [getelementptr nuw][rust_gep_inbounds_nuw] for pointer arithmetic.

Packaging
---------

The LLVM team at Red Hat has shared responsibility for packaging LLVM on Fedora, CentOS Stream, and RHEL. Most of the work happens as part of round-robin maintenance of daily[^daily_snapshots] [snapshot builds][snapshots]. In theory, snapshot builds ensure that shipping a new major version is as simple as incrementing a version number. In practice, it never works out quite that easily.

In the previous year, we had already started using a monolithic build for the core llvm packages. This year, the mlir, polly, bolt, libcxx and flang builds were also merged into the monolithic build, which means that we only build libclc separately now. Additionally, the builds now use PGO. These improvements were made by my colleague [kwk][kwk].

One change that I worked on, and which ended up as a big failure, was to increase the consistency between our main llvm package, and the llvmNN compatibility packages we provide for older versions. The compatibility packages install LLVM inside a prefixed path like `/usr/lib64/llvmNN`, while the main package is installed to the usual system paths. The idea was that we should always install to the prefixed path, and symlink from the system path. That way, the main package could be used the same as a compatibility package, avoiding the need for adjustments when switching between them.

The first issue this ran into is that RPM does not support replacing a directory with a symlink during upgrades. There are [documented workarounds][rpm_symlink] using pretrans scriptlets, but those don't fully work.[^pretrans_scriptlets] In the end we had to symlink individual files instead of symlinking entire directories.

The second issue only became apparent much later, after this change had already shipped: It was no longer possible to install the 32-bit and 64-bit packages of LLVM at the same time (known as the "multilib" configuration). While the prefixed path for both packages is different, they both install symlinks in the same system paths, and once again, RPM can't deal with that. RPM has special "file color" support that lets 64-bit files win over 32-bit ones, but *of course* it only works for ELF files, not for symlinks.

I explored lots of options for fixing this, but everything ended up running into one missing RPM feature or another. In the end, I ended up inverting the symlink direction (making the version-prefixed paths point to the system path). The lesson learned here is that if you use RPM, you should avoid symlinks like the plague.

### LLVM area team and project council

This year, LLVM adopted a new [governance process][llvm_governance], which includes elected area teams. Together with [fhahn][fhahn] and [arsenm][arsenm], I have been [elected][llvm_election] to the LLVM area team. The LLVM area team holds a [meeting][llvm_area_team_meetings] every two weeks to discuss pending RFCs. (The meetings are public, but usually it's just us three.)

Our approach has generally been hands-off. We have explicitly approved some RFCs where people expressed uncertainty, but usually our only involvement has been to provide additional comments on RFCs with insufficient engagement.

Unlike some other areas, I believe we had very few controversial proposals/discussions. One of them is an extensive discussion on [floating-point min/max semantics][minmax_semantics], which has only been resolved recently. The other one was on [delinearization challenges][detypeification], but that was more a generic complaint than a specific proposal.

As chair of the LLVM area team, I also participate in the project council. This is kind of the opposite of the area team, in that nearly all topics that reach the project council are controversial -- things like the AI policy, the mandatory pull request proposal, and the sframe upstreaming. While progress has been made, we haven't reached final resolutions on many of these topics yet. The [AI policy][ai_policy] is now live though.

### Other

Towards the end of the year, we formed the [formal specification working group][formal_spec_wg], which aims to close long standing correctness gaps in LLVM, especially relating to the provenance model. I've participated in various early discussions for this group and wrote up a draft [provenance model][provenance_model] for LLVM. The current focus of the group is the [byte type][byte_type_rfc].

I've deprecated the [global context][global_context_deprecation] in the LLVM C API, a common footgun. I've changed the representation of alignment in [masked memory intrinsics][masked_mem_align]. I've simplified the in-memory [blockaddress representation][blockaddress_repr] and proposed [more significant changes][blockaddress_rfc] for the future.

Last but not least, I reviewed approximately 2500 pull requests last year. Unfortunately, this is nowhere near enough to keep up with my review queue.

[^daily_snapshots]: In practice, we produce successful builds across all architectures and operating systems much less often than daily. Something always breaks.

[^pretrans_scriptlets]: They solve the upgrade problem, but not the downgrade problem, so still fail rpmdeplint. And handling downgrades requires a change to *previous* versions of the package.

[^lvi_loops]: LVI is a shared analysis between JumpThreading and CVP. Because JumpThreading performs pervasive control-flow changes, it cannot preserve the dominator tree. As such, LVI also can't use the dominator tree. This results in a purely recursive analysis, which conservatively aborts on cycles, even if there is a common dominating condition.

[^predicateinfo]: PredicateInfo performs SSA-renaming based on branch conditions and assumes. This results in something akin to SSI (static single information) form, and allows standard sparse dataflow propagation to make use of them.

[year_2022]: https://www.npopov.com/2022/12/20/This-year-in-LLVM-2022.html
[year_2023]: https://www.npopov.com/2024/01/01/This-year-in-LLVM-2023.html
[year_2024]: https://www.npopov.com/2025/01/05/This-year-in-LLVM-2024.html

[lifetime_start]: https://llvm.org/docs/LangRef.html#llvm-lifetime-start-intrinsic
[lifetime_end]: https://llvm.org/docs/LangRef.html#llvm-lifetime-end-intrinsic
[lifetime_alloca_only]: https://github.com/llvm/llvm-project/pull/149310
[lifetime_alloca_no_size]: https://github.com/llvm/llvm-project/pull/150248
[lifetime_problem_licm]: https://github.com/llvm/llvm-project/issues/51838
[lifetime_problem_icmp]: https://github.com/llvm/llvm-project/issues/45725

[ptradd_rfc]: https://discourse.llvm.org/t/rfc-replacing-getelementptr-with-ptradd/68699?u=nikic
[gep_split_var]: https://github.com/llvm/llvm-project/pull/137297
[gep_split_all]: https://github.com/llvm/llvm-project/pull/151333
[gep_leading_zeros]: https://github.com/llvm/llvm-project/pull/155415

[ptrtoaddr_rfc]: https://discourse.llvm.org/t/clarifiying-the-semantics-of-ptrtoint/83987?u=nikic
[ptrtoaddr_pr]: https://github.com/llvm/llvm-project/pull/139357
[cheri_rfc]: https://discourse.llvm.org/t/rfc-upstream-target-support-for-cheri-enabled-architectures/87623?u=nikic
[cheri_wiki]: https://en.wikipedia.org/wiki/Capability_Hardware_Enhanced_RISC_Instructions
[byte_type_rfc]: https://discourse.llvm.org/t/rfc-add-a-new-byte-type-to-llvm-ir/89522?u=nikic

[rust_llvm20]: https://github.com/rust-lang/rust/pull/135763
[rust_llvm21]: https://github.com/rust-lang/rust/pull/143684
[rust_llvm20_perf]: https://perf.rust-lang.org/compare.html?start=2162e9d4b18525e4eb542fed9985921276512d7c&end=ce36a966c79e109dabeef7a47fe68e5294c6d71e&stat=instructions:u
[rust_llvm21_perf]: https://perf.rust-lang.org/compare.html?start=ec7c02612527d185c379900b613311bc1dcbf7dc&end=dc0bae1db725fbba8524f195f74f680995fd549e&stat=instructions:u

[rust_alloc_variant_zeroed]: https://github.com/rust-lang/rust/pull/144086
[rust_read_only_captures]: https://github.com/rust-lang/rust/pull/145259
[rust_dead_on_return]: https://github.com/rust-lang/rust/pull/145093
[rust_gep_inbounds_nuw]: https://github.com/rust-lang/rust/pull/137271

[llvm_governance]: https://github.com/llvm/llvm-www/blob/main/proposals/LP0004-project-governance.md
[llvm_election]: https://discourse.llvm.org/t/llvm-area-team-election-results/84601?u=nikic
[llvm_area_team_meetings]: https://discourse.llvm.org/t/llvm-area-team-meetings/85368?u=nikic
[fhahn]: https://github.com/fhahn
[arsenm]: https://github.com/arsenm
[minmax_semantics]: https://discourse.llvm.org/t/rfc-a-consistent-set-of-semantics-for-the-floating-point-minimum-and-maximum-operations/89006?u=nikic
[detypeification]: https://discourse.llvm.org/t/rfc-de-type-ification-of-llvm-ir-why/88257?u=nikic
[ai_policy]: https://llvm.org/docs/AIToolPolicy.html

[vortex73]: https://github.com/vortex73
[abi_library_rfc]: https://discourse.llvm.org/t/rfc-an-abi-lowering-library-for-llvm/84495?u=nikic
[abi_library_blog]: https://blog.llvm.org/posts/2025-08-25-abi-library/
[abi_prototype]: https://github.com/llvm/llvm-project/pull/140112
[datalayout_consistency]: https://github.com/llvm/llvm-project/pull/144720
[datalayout_deduplication]: https://github.com/llvm/llvm-project/pull/171135
[cc_orig_ir_type]: https://github.com/llvm/llvm-project/pull/152709
[mips_f128_libcalls]: https://github.com/llvm/llvm-project/pull/153798

[snapshots]: https://copr.fedorainfracloud.org/coprs/g/fedora-llvm-team/llvm-snapshots/
[kwk]: https://github.com/kwk
[rpm_symlink]: https://docs.fedoraproject.org/en-US/packaging-guidelines/Directory_Replacement/

[sccp_worklist]: https://github.com/llvm/llvm-project/commit/545cdca4883552b147a0f1adfac713f76fc22305
[sccp_worklist_ct]: https://llvm-compile-time-tracker.com/compare.php?from=6f7370ced630ec1994456a979ca10ac26e3dc0a7&to=545cdca4883552b147a0f1adfac713f76fc22305&stat=instructions:u
[base_obj_size]: https://github.com/llvm/llvm-project/commit/a987022f33a27610732544b0c5f4475ce818c982
[base_obj_size_ct]: https://llvm-compile-time-tracker.com/compare.php?from=af41d0d7057d8365c7b48ce9f88d80b669057993&to=a987022f33a27610732544b0c5f4475ce818c982&stat=instructions:u
[type_alloc_size]: https://github.com/llvm/llvm-project/commit/4d927a5faf42d025410586f0cdc3bf60ef198a86
[type_alloc_size_ct]: https://llvm-compile-time-tracker.com/compare.php?from=34d4f0c13666ea25b4d27dcb96dfc70da005f286&to=4d927a5faf42d025410586f0cdc3bf60ef198a86&stat=instructions:u

[debug_linetable]: https://github.com/llvm/llvm-project/commit/26d9cb17a6e655993c991b66b21d5c378256d79b
[debug_linetable_ct]: https://llvm-compile-time-tracker.com/compare.php?from=705e27c23474f3177670a791b5b54eefedee0cd8&to=26d9cb17a6e655993c991b66b21d5c378256d79b&stat=instructions:u
[name_specifier]: https://github.com/llvm/llvm-project/commit/91cdd35008e9ab32dffb7e401cdd7313b3461892
[name_specifier_ct]: https://llvm-compile-time-tracker.com/compare.php?from=fc44a4fcd3c54be927c15ddd9211aca1501633e7&to=91cdd35008e9ab32dffb7e401cdd7313b3461892&stat=instructions:u

[constantint_assert]: https://github.com/llvm/llvm-project/pull/171456
[selectopt_bug]: https://github.com/llvm/llvm-project/pull/170860

[sccp_predicateinfo]: https://github.com/llvm/llvm-project/pull/153003
[store_merge]: https://github.com/llvm/llvm-project/pull/147540
[drop_assumes]: https://github.com/llvm/llvm-project/pull/159403
[reddit_codegen_gcc]: https://www.reddit.com/r/rust/comments/1lhgld6/why_do_these_bit_munging_functions_produce_bad_asm/

[captures_rfc]: https://discourse.llvm.org/t/rfc-improvements-to-capture-tracking/81420?u=nikic
[captures_inference]: https://github.com/llvm/llvm-project/pull/125880
[captures_aa_prov]: https://github.com/llvm/llvm-project/pull/130777
[captures_aa_read_only]: https://github.com/llvm/llvm-project/pull/143097
[captures_read_only_rust]: https://github.com/rust-lang/rust/pull/145259
[captures_metadata]: https://github.com/llvm/llvm-project/pull/160913
[mir_retags]: https://github.com/rust-lang/unsafe-code-guidelines/issues/371

[formal_spec_wg]: https://discourse.llvm.org/t/rfc-forming-a-working-group-on-formal-specification-for-llvm/89056?u=nikic
[provenance_model]: https://hackmd.io/@nikic/SJBt4mFCll

[global_context_deprecation]: https://github.com/llvm/llvm-project/pull/163979
[masked_mem_align]: https://github.com/llvm/llvm-project/pull/163802
[blockaddress_repr]: https://github.com/llvm/llvm-project/pull/137958
[blockaddress_rfc]: https://discourse.llvm.org/t/rfc-changing-the-indirectbr-blockaddress-representation/88677?u=nikic
