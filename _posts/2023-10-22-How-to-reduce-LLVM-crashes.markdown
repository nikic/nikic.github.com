---
layout: post
title: How to reduce LLVM crashes
excerpt: Step-by-step guide on how to reduce LLVM crashes.
---
Unlike miscompilations, reducing compiler crashes to a minimal test case is a straightforward process, which always follows the same general approach, and is supported by high-quality reduction tooling.

However, I don't think the entire process (and some of the edge cases) is documented anywhere, so I am trying to remedy that.

## Prerequisites

To reduce LLVM crashes, it is strongly recommended to use an assertion-enabled build of LLVM (using `-DLLVM_ENABLE_ASSERTIONS=true`). You do not need (or want) a debug build.

While something may also crash without assertions, enabling assertions can make the crash more stable and move it to an earlier point.

Usually, the only way to get an assertion-enabled build is to build LLVM yourself -- as this guide is primarily targeted at LLVM developers, you should have one anyway. (People in the Rust ecosystem on x86-64 Linux can also download an "alt" build of rustc with enabled LLVM tools.)

All of the following examples use plain binary names like `opt` -- don't forget to point these to the correct binary! I regularly mess up and use the system binary instead.

For LLVM crashes, it is usually *not* worthwhile to do any pre-reduction in the source language. It is best to go directly to LLVM IR. Of course, if the crash is in the frontend, then you'll want to use something like `cvise` to reduce that -- but this blog post is not about frontend crashes.

## Reproducing in opt/llc

The first step is to reproduce the issue using standard LLVM tooling, which is `opt` for the middle-end and `llc` for the backend. When debugging `clang` crashes in particular, this is often not necessary, and one can jump directly to the new step. However, this is usually necessary for other compilers using LLVM, such as `rustc`.

For `clang`, it is possible to obtain the *optimized* bitcode using `-emit-llvm` and the optimized IR using `-emit-llvm -S`. You can then feed the result into `llc` to confirm the crash.

```sh
clang -O3 -emit-llvm -S test.c
llc < test.ll > /dev/null
```

The unoptimized IR can be obtained by adding `-Xclang -disable-llvm-optzns`. Do **not** use `-O0` instead, which will annotate all functions with `optnone`. In this case, `opt` is used to confirm the crash.

```sh
clang -O3 -Xclang -disable-llvm-optzns -emit-llvm -S test.c
opt -disable-output -O3 < test.ll
```

If the crash happens during LTO, you can pass `-plugin-opt=save-temps` to the linker instead. This is done by passing `-Wl,-plugin-opt=save-temps` to the compiler driver, or by dumping the linker command using `-v` and a manual invocation.

This will produce a number of `*.bc` files at different stages with names like these:

```
foo.0.preopt.bc
foo.1.promote.bc     # ThinLTO only
foo.2.internalize.bc
foo.3.import.bc      # ThinLTO only
foo.4.opt.bc
foo.5.precodegen.bc
```

Depending on whether you use full or thin LTO, there will be one set of these files, or many. For a middle-end crash, the `opt.bc` and `precodegen.bc` files will not be present. We carry on either in `opt` or `llc`:

```sh
# Full LTO middle-end crash.
opt -disable-output -passes='lto<O3>' < foo.2.internalize.bc
# Thin LTO middle-end crash.
opt -disable-output -passes='thinlto<O3>' < foo.3.import.bc
# Backend crash.
llc < foo.5.precodegen.bc > /dev/null
```

When using ThinLTO, there will be many bitcode files. For middle-end crashes, you can find the one missing opt/precodegen. For backend crashes, just try all of them:

```sh
for f in *.precodegen.bc; do echo $f; llc < $f > /dev/null; done
```

For `rustc`, the first thing you want to do is reduce it to an actual `rustc` call, rather than an invocation of `cargo`, `x.py` or similar. Usually the crashing command gets dumped automatically, but otherwise `-v` will work.

Once you have a `rustc` command, there are two ways to obtain bitcode/IR. The first one is to use `--emit=llvm-ir` or `--emit=llvm-bc`, optionally combined with `-C no-prepopulate-passes` for unoptimized IR (depending on whether it's a middle-end or backend crash).

However, this has the big disadvantage that `--emit=llvm-ir` significantly impacts the optimization pipeline. In particular, it will force `-C codegen-units=1` and disable crate-local ThinLTO, which will often result in the crash no longer reproducing.

Instead, it is better to use `-C save-temps`, which essentially works the same as the LTO case above: It will produce a whole bunch of `.bc` files, just with different naming. For rustc commands produced by cargo, these will be in the `target/release/deps` directory. The easiest way to find the right one is once again to loop over all of them and check which one crashes.

Sometimes, passing bitcode/IR to opt/llc fails to reproduce the issue. One common cause for this is a mismatch in options between what llc uses and what the frontend specifies.

The first thing to be aware of is that `llc` default to `-O2`, so you need to explicitly specify `-O0` for crashes in unoptimized builds. The second is that `llc` produces assembly by default and you need `-filetype=obj` to produce object code. This matters for crashes in the MC layer. Sometimes `-relocation-model=pic` is also relevant.

Unfortunately, this is something that requires domain knowledge -- if a straightforward opt/llc invocation doesn't work, it's often hard to figure out what the relevant difference is.

## Obtaining IR directly before the crash

The second step is to obtain the IR directly before the crash. This section is only relevant for middle-end crashes.

LLVM has a convenient `-print-on-crash` option, which dumps the IR before the crashing pass. By default, this will only dump a single function, which is going to reference lots of symbols and metadata that did not get dumped. It can be combined with `-print-module-scope` to dump the full module instead.

```sh
opt -disable-output -O3 -print-on-crash -print-module-scope input.bc 2>out.ll
```

This will produce a file that looks something like this:
```
opt: [ASSERTION FAILURE MESSAGE]
[STACK TRACE]
*** Dump of Module IR Before Last Pass GVNPass Started ***
[MODULE DUMP]
```

Drop everything before the actual module dump, and then continue running `opt` with just the indicated pass (GVNPass here):

```sh
opt -disable-output -passes=gvn < out.ll
```

`-print-on-crash` works by dumping the function/module before each pass, and then only printing it once a crash happens. Especially if `-print-module-scope` is used, this is extremely inefficient, and is only feasible for small inputs.

For larger test cases, there is an alternative approach. First, run `opt` with the `-print-pass-numbers` option:

```sh
opt -disable-output -O3 -print-pass-numbers < input.bc
```

This is going to produce output that looks something like this:

```
 Running pass 1 Annotation2MetadataPass on [module]
[PASS LOG]
 Running pass 12345 MergedLoadStoreMotionPass on function_abc
 Running pass 12346 GVNPass on function_abc
opt: [ASSERTION FAILURE MESSAGE]
[STACK TRACE]
```

Then run `opt` again, this time using `-print-before-pass-number` with the last pass number, here `12346`:

```sh
opt -disable-output -O3 -print-before-pass-number=12346 -print-module-scope < input.bc 2>out.ll
```

Once again, it's necessary to manually delete the backtrace from the produced file, after which you can carry on using `opt` with a single pass.

I mentioned before that the first step (reproducing in opt/llc) is optional. The reason is that you can also pass these options directly to the compiler frontend. Examples for `clang` and `rustc` would be:

```sh
clang [ARGS] -mllvm -print-before-pass-number=12346 -mllvm -print-module-scope 2>out.ll
rustc [ARGS] -Cllvm-args=-print-before-pass-number=12346 -Cllvm-args=-print-module-scope 2>out.ll
```

However, this only works if the frontend compiles a single module. For example, rustc will usually build multiple codegen units instead. Even if you disable parallelism (`-Z no-parallel-llvm`), you'll still be left with pass numbers for many separate optimization pipelines. As such, this is only really applicable when the issue reproduces under `-C codegen-units=1`. The same goes for LTO crashes with clang.

Sometimes, dumping the IR before the crashing pass and then running that pass will fail to reproduce the problem. We'll get back to what to do in that case later, in the "special cases" section. For now, let's assume we have a single-pass reproducer.

## IR reduction

The next (and final) step is to reduce the issue to a minimal IR reproducer, which is done using the `llvm-reduce` tool. Do **not** use `bugpoint`, which is an old implementation of the same basic idea.

`llvm-reduce` accepts an interestingness test, which should exit with status 0 for interesting inputs. A typical interestingness test would be:

```sh
#!/bin/bash
! opt -disable-output -passes=gvn < $1
# Or for a backend crash:
! llc < $1 > /dev/null
```

Save this to `crash.sh` and don't forget to `chmod +x crash.sh`. Then run `llvm-reduce`:

```sh
llvm-reduce --test crash.sh out.ll
```

The reduction will be saved to `reduced.ll`.

One somewhat common gotcha is that `llvm-reduce` may run some IR passes during the reduction. If one of those passes is the crashing one, `llvm-reduce` itself is going to crash. The passes used by default (as of this writing) are `function(sroa,instcombine,gvn,simplifycfg,infer-address-spaces)`. If you're trying to reduce any of these, you're probably going to have a bad time.

It's possible to avoid this in two ways. The first is to disable the IR passes entirely by passing `--skip-delta-passes=ir-passes`. The second is to exclude just the problematic one. For example, if we want to exclude just GVN:

```sh
llvm-reduce --test crash.sh out.ll \
    --ir-passes='function(sroa,instcombine,simplifycfg,infer-address-spaces)'
```

The interestingness test used above considers any crashing input "interesting". Usually, this is what I want. However, it can sometimes reduce to a different error condition. To avoid that, it's possible to make it more specific, e.g. by checking for a specific assertion failure:

```sh
#!/bin/bash
opt -disable-output -passes=gvn < $1 2>&1 | grep "[FAILURE_MESSAGE_OMITTED]"
```

Once the reduction is done, it is often worthwhile to run the MetaRenamer pass on it:

```sh
opt -S -passes=metarenamer < reduced.ll > reduced2.ll
```

This pass will rename all symbols (both functions and local variables). This serves the dual purpose of getting rid of unnamed symbols, and avoiding long and confusing symbol names like `%.sroa.0.0.copyload.i.i.i.i49.i`, where you have to carefully count the number of `.i`s to distinguish one variable from another.

Afterwards, it is advisable to perform some final manual reduction. While `llvm-reduce` is quite good, it is not perfect, and it is often possible to further simplify the test.

At this point, we are essentially done. Once you have a minimal test case, tracking down and fixing the actual bug tends to be easy. It may help to pass `-debug` to get some more information on what transforms the pass in question is performing.

## Special cases

While the above workflow covers most LLVM crashes, there are some special cases to keep in mind. The first one is that extracting the IR before the crash and running it through the pass may fail to reproduce the issue.

This usually happens for one of two reasons. The first is that the pass runs with some non-default options, which need to be explicitly passed to reproduce the crash. Unfortunately, all of the IR printing methods only print the class name of the pass, rather than its pass pipeline representation (this seems like a nice opportunity for improvement!)

You can see the pass parameters used by the different passes in the pipeline using the `-print-pipeline-passes` option:

```sh
opt -disable-output -O3 -print-pipeline-passes input.bc
```

This is going to print something like the following:

```
annotation2metadata,forceattrs,inferattrs,coro-early,function<eager-inv>(lower-expect,simplifycfg<bonus-inst-threshold=1;no-forward-switch-cond;no-switch-range-to-icmp;no-switch-to-lookup;keep-loops;no-hoist-common-insts;no-sink-common-insts;speculate-blocks;simplify-cond-branch>,sroa<modify-cfg>,early-cse<>,callsite-splitting),openmp-opt,ipsccp,called-value-propagation,globalopt,function<eager-inv>(mem2reg,instcombine<max-iterations=1;no-use-loop-info;no-verify-fixpoint>,simplifycfg<bonus-inst-threshold=1;no-forward-switch-cond;switch-range-to-icmp;no-switch-to-lookup;keep-loops;no-hoist-common-insts;no-sink-common-insts;speculate-blocks;simplify-cond-branch>),require<globals-aa>,function(invalidate<aa>),require<profile-summary>,cgscc(devirt<4>(inline<only-mandatory>,inline,function-attrs<skip-non-recursive-function-attrs>,argpromotion,openmp-opt-cgscc,function<eager-inv;no-rerun>(sroa<modify-cfg>,early-cse<memssa>,speculative-execution,jump-threading,correlated-propagation,simplifycfg<bonus-inst-threshold=1;no-forward-switch-cond;switch-range-to-icmp;no-switch-to-lookup;keep-loops;no-hoist-common-insts;no-sink-common-insts;speculate-blocks;simplify-cond-branch>,instcombine<max-iterations=1;no-use-loop-info;no-verify-fixpoint>,aggressive-instcombine,libcalls-shrinkwrap,tailcallelim,simplifycfg<bonus-inst-threshold=1;no-forward-switch-cond;switch-range-to-icmp;no-switch-to-lookup;keep-loops;no-hoist-common-insts;no-sink-common-insts;speculate-blocks;simplify-cond-branch>,reassociate,constraint-elimination,loop-mssa(loop-instsimplify,loop-simplifycfg,licm<no-allowspeculation>,loop-rotate<header-duplication;no-prepare-for-lto>,licm<allowspeculation>,simple-loop-unswitch<nontrivial;trivial>),simplifycfg<bonus-inst-threshold=1;no-forward-switch-cond;switch-range-to-icmp;no-switch-to-lookup;keep-loops;no-hoist-common-insts;no-sink-common-insts;speculate-blocks;simplify-cond-branch>,instcombine<max-iterations=1;no-use-loop-info;no-verify-fixpoint>,loop(loop-idiom,indvars,loop-deletion,loop-unroll-full),sroa<modify-cfg>,vector-combine,mldst-motion<no-split-footer-bb>,gvn<>,sccp,bdce,instcombine<max-iterations=1;no-use-loop-info;no-verify-fixpoint>,jump-threading,correlated-propagation,adce,memcpyopt,dse,move-auto-init,loop-mssa(licm<allowspeculation>),coro-elide,simplifycfg<bonus-inst-threshold=1;no-forward-switch-cond;switch-range-to-icmp;no-switch-to-lookup;keep-loops;hoist-common-insts;sink-common-insts;speculate-blocks;simplify-cond-branch>,instcombine<max-iterations=1;no-use-loop-info;no-verify-fixpoint>),function-attrs,function(require<should-not-run-function-passes>),coro-split)),deadargelim,coro-cleanup,globalopt,globaldce,elim-avail-extern,rpo-function-attrs,recompute-globalsaa,function<eager-inv>(float2int,lower-constant-intrinsics,chr,loop(loop-rotate<header-duplication;no-prepare-for-lto>,loop-deletion),loop-distribute,inject-tli-mappings,loop-vectorize<no-interleave-forced-only;no-vectorize-forced-only;>,infer-alignment,loop-load-elim,instcombine<max-iterations=1;no-use-loop-info;no-verify-fixpoint>,simplifycfg<bonus-inst-threshold=1;forward-switch-cond;switch-range-to-icmp;switch-to-lookup;no-keep-loops;hoist-common-insts;sink-common-insts;speculate-blocks;simplify-cond-branch>,slp-vectorizer,vector-combine,instcombine<max-iterations=1;no-use-loop-info;no-verify-fixpoint>,loop-unroll<O3>,transform-warning,sroa<preserve-cfg>,infer-alignment,instcombine<max-iterations=1;no-use-loop-info;no-verify-fixpoint>,loop-mssa(licm<allowspeculation>),alignment-from-assumptions,loop-sink,instsimplify,div-rem-pairs,tailcallelim,simplifycfg<bonus-inst-threshold=1;no-forward-switch-cond;switch-range-to-icmp;no-switch-to-lookup;keep-loops;no-hoist-common-insts;no-sink-common-insts;speculate-blocks;simplify-cond-branch>),globaldce,constmerge,cg-profile,rel-lookup-table-converter,function(annotation-remarks),verify
```

Finding the invocations of the relevant pass and copying their parameters may help.

The second reason why a single-pass reproduction may fail is that the issue *cannot* be reproduced using a single pass: This mainly happens when the issue is caused by missing invalidation of cached analyses. You need one pass to compute the analysis first, and another to use it later.

In this case, it is possible to perform a pipeline reduction using the `llvm/utils/reduce_pipeline.py` script:

```sh
llvm/utils/reduce_pipeline.py --opt-binary opt --passes "default<O3>" \
    --input input.bc --output out.ll 
```

This will expand the passed pipeline to a string like the one above, and then try to remove passes from it. At the end, you will be left with an output `out.ll` and a new pass pipeline, that might look something like this:

```
-passes="cgscc(devirt<4>(function<eager-inv;no-rerun>(loop-mssa(loop-rotate<header-duplication;no-prepare-for-lto>,licm<allowspeculation>))))"
```

You should then be able to reproduce the issue by running:

```sh
opt -passes="cgscc(devirt<4>(function<eager-inv;no-rerun>(loop-mssa(loop-rotate<header-duplication;no-prepare-for-lto>,licm<allowspeculation>))))" < out.ll
```

The output will often contain a bunch of unnecessary "pass adaptors", which can usually be omitted:

```sh
# This will probably also work
opt -passes="loop-mssa(loop-rotate<header-duplication;no-prepare-for-lto>,licm<allowspeculation>)" < out.ll
# And maybe this will as well
opt -passes="loop-mssa(loop-rotate,licm)" < out.ll
```

Once you have a reduced pipeline, you can reduce the input IR using `llvm-reduce` once again.

If you suspect an analysis invalidation issue, it can be helpful to pass the corresponding verification flag, such as `-verify-scev` for scalar evolution. This may expose the issue at an earlier point in the pipeline.

In some cases, it is possible to replace one of the "real" passes with something like `print<scalar-evolution>`, which just computes (and prints) an analysis. This can make it clearer that the only purpose of the pass is to cache an analysis, and that the transforms it does are not relevant.

Another interesting special case are IR verifier failures. The verifier will usually only run at the end of the pass pipeline, while the issue it detects will be introduced by an earlier pass. It is possible to verify after each pass instead, by passing the `-verify-each` option.

Finally, a note on compiler hangs. The general approach for these is pretty similar to crashes. For example, you can use `-print-pass-numbers` to find out at which pass it starts hanging, and then use `-print-at-pass-number` to print the IR before the hang and get a single-pass reproducer.

You can then use `llvm-reduce` with a timeout-based reproducer:

```sh
#!/bin/bash
! timeout 10 opt -disable-output -passes=gvn < $1
```

This will consider any input that runs for 10 seconds as interesting. Of course, this means that reduction is quite slow, and you may have to keep this running in the background for a few hours.

Additionally, this reduction approach has the tendency to give you a reduction that takes exactly 10 seconds to run. This is somewhat unavoidable if the original issue is not an actual infinite loop, but rather "just" strongly super-linear complexity. This may or may not be helpful.

## Cheatsheet of useful options and tools

Tools:

 * `opt`: Run middle-end optimizer.
 * `llc`: Run backend.
 * `llvm-reduce`: Reduce IR using interestingness test.
 * `llvm/util/reduce_pipeline.py`: Reduce pass pipeline.

`opt` options:

 * `-O3`, `-passes='default<O3>'`: Run standard optimization pipeline.
 * `-passes='lto<O3>'`: Run full LTO pipeline.
 * `-passes='thinlto<O3>'`: Run thin LTO pipeline.
 * `-passes=metarenamer`: Rename all symbols.
 * `-print-on-crash`: Print IR before crashing pass.
 * `-print-module-scope`: Print whole module instead of single function.
 * `-print-pass-numbers`: Print log of executed passes, for use with next option.
 * `-print-before-pass-number`: Print IR before pass with number printed by previous option.
 * `-print-pipeline-passes`: Print string representation of optimization pipeline.
 * `-verify-each`: Run IR verifier after each pass.

`llc` options:

 * `-O0`: Disable optimization. `-O2` is the default.
 * `-filetype=obj`: Produce machine code instead of assembly.
 * `-verify-machineinstrs`: Enable machine instruction verification.

`llvm-reduce` options:

 * `--test`: Specify test script that exits with 0 for interesting inputs.
 * `--skip-delta-passes=ir-passes`: Disable reduction using IR passes.
 * `--ir-passes="function(...)"`: Specify custom IR pass reduction pipeline.

`clang` options:

 * `-emit-llvm`: Dump optimized bitcode. Combine with `-S` for IR.
 * `-Xclang -disable-llvm-optzns`: Disable optimization (for dumping unoptimized bitcode).
 * `-Wl,-plugin-opt=save-temps`: Save temporary bitcode files during LTO.
 * `-mllvm -foo`: Pass option through to LLVM.

`rustc` options:

 * `--emit=llvm-ir` / `--emit=llvm-bc`: Dump optimized bitcode / IR. Warning: Implicitly uses one codegen unit and disables crate-local ThinLTO.
 * `-C no-prepopulate-passes`: Disables optimizations.
 * `-C save-temps`: Save temporary bitcode files.
 * `-Z no-parallel-llvm`: Disable backend parallelism. Needed when using any LLVM options that print text, to avoid interleaving.
 * `-C llvm-args=-foo`: Pass option through to LLVM.
