---
layout: post
title: "LLVM: The middle-end optimization pipeline"
excerpt: A high-level overview of LLVM's middle-end optimization pipeline and some of the design considerations behind it.
---

Compilers are classically split into three parts: The front end, the middle end and the back end. The front end is what deals with programming-language specific analysis, such as parsing, type checking, etc.

The middle end performs optimizations that are independent (to the degree that this is possible) from both the input language and the output target. Finally, the back end performs target-specific optimizations and emits machine code.

This separation is what saves everyone from reinventing the wheel. New programming languages only need to implement the frontend. New CPU architectures only need to implement the backend. And indeed, while there are many, many different language frontends, they nearly always rely on LLVM for the middle and backends:

```
 Frontend                    LLVM
/---------\                /-----------------------------------\
| clang   |                |                                   |
| rustc   | --(LLVM IR)--> | Middle end --(LLVM IR)--> Backend | --> Machine code
| ...     |                |                                   |
\---------/                \-----------------------------------/
```

In this article I want to discuss, at a very high level, how LLVM's middle end optimization pipeline looks like.

The three optimization pipelines
--------------------------------

There are three kinds of optimization pipelines: The default (non-LTO) pipeline, the ThinLTO pipeline and the FatLTO pipeline. The default pipeline works by separately optimizing each module, without any special knowledge about other modules (apart from function/global declarations). "Module" here corresponds to "one source file" in C/C++ and one "codegen unit" in Rust.

```
# Default

Module 1 --(Optimize)--> Module 1'
Module 2 --(Optimize)--> Module 2'
Module 3 --(Optimize)--> Module 3'
```

The LTO pipelines are split into a pre-link and a post-link optimization pipeline. After the pre-link pipeline, ThinLTO will perform some lightweight cross-module analysis, and in particular import certain functions from other modules to make them eligible for inlining. However, the post-link optimization again works on individual modules.

```
# ThinLTO
                              cross-import
Module 1 --(ThinLTO pre-link)------\-/-----(ThinLTO post-link)--> Module 1'
Module 2 --(ThinLTO pre-link)-------X------(ThinLTO post-link)--> Module 2'
Module 3 --(ThinLTO pre-link)------/-\-----(ThinLTO post-link)--> Module 3'
```

Finally, FatLTO will simply merge all modules together after the pre-link pipeline and then run the post-link pipeline on a single, huge module:

```
# FatLTO
                                 merge
Module 1 --(FatLTO pre-link)-------\
Module 2 --(FatLTO pre-link)---------------(FatLTO post-link)--> Module'
Module 3 --(FatLTO pre-link)-------/
```

While LTO stands for "link-time optimization", it is not necessarily performed by the linker. For example, rustc will usually perform it as part of the compiler. The important property of LTO is that is allows cross-module optimization.

LTO is also typically accompanied by a restricted list of symbols that need to be exported (i.e. externally accessible), while everything else can be internalized (i.e. become an internal implementation detail). This allows for whole program optimization (WPO).

The default optimization pipeline
---------------------------------

The optimization pipelines are defined in [PassBuilderPipelines][PassBuilderPipelines] and the default pipeline in particular is created by `buildPerModuleDefaultPipeline()`. I will mostly talk about the pipeline in terms of generalities, so take a look at that file if you want to inspect the precise pass ordering.

The pipeline consists of two parts: The module simplification and the module optimization pipelines.

The job of the simplification pipeline is to simplify and canonicalize the IR. The job of the optimization pipeline is to perform optimizations that may make the IR more complex or less canonical.

Examples of the latter are vectorization and runtime unrolling. These transforms will generally greatly increase IR size and make it harder to analyze, which is why it is important that they run in the late optimization pipeline.

Most other transforms are simplifications, including classics like SROA, CSE/GVN, LICM, instruction combining, etc. An interesting case is full unrolling, which, unlike runtime unrolling, is also part of the simplification pipeline, because it converts a loop into straight-line code, opening up additional simplification opportunities.

The LTO pipelines
-----------------

The ThinLTO pipeline is where the simplification/optimization split really comes into action: The pre-link pipeline basically just runs the simplification part, while the post-link pipeline runs both simplification and optimization.

The key property of LTO optimization is that additional inlining may be performed in the post-link step. As such, it is important that optimizations that increase IR size or decrease canonicality are not performed during the pre-link phase.

For example, we do not want functions to get vectorized pre-link, because this might prevent them from being inlined and then simplified in more profitable ways.

For the same reason, some other minor adjustments to the pre-link phase are performed. For example, loop rotation will be suppressed if it would duplicate a call: That call might become inlinable post-link, which might render the duplication much more expensive than it would otherwise be.

Running the simplification pipeline over everything again post-link is not strictly necessary: Ideally, it would only be applied to functions where additional inlining actually happened. This is a possible optimization for the future.

As for the fat LTO pipeline ... well, we don't talk about the fat LTO pipeline. It's design is somewhere between non-sensical and non-existent. Fat LTO runs the module optimization pipeline pre-link (which is not a good idea for the same reasons as for ThinLTO), and the post-link pipeline is entirely home-grown and not based on the standard simplification/optimization pipelines.

The module simplification pipeline
----------------------------------

Now that we have put things into context, let's return to the module simplification pipeline, the core of which is the CGSCC inliner. CGSCC is short for call-graph [strongly-connected components][scc].

The basic idea is that if you have a call-graph consisting of callers and callees (callee = called function), you want to simplify the callees first, before you try to inline them into the callers (and then simplify those too). Simplifying callees before inlining makes the cost-modelling more accurate and reduces the amount of simplification that needs to be performed post-inlining.

This is what CGSCC passes do: They try to visit callees before callers. For cyclic (recursive) call-graphs, this is not possible, so they do the next best thing: Visit strongly-connected components of mutually recursive functions. There is no well-defined visitation order within one component, but at least we preserve the "callee before callers" property between components.

The CGSCC pipeline consists essentially of the inliner and the function simplification pipeline. In addition to that it contains a few other CGSCC passes, e.g. for argument promotion (argument-level SROA) and inferral of function attributes.

The key bit is that inlining and simplification are interleaved, so that after calls into a function have been inlined, it will be simplified before becoming an inlining candidate itself. (This is another failure of the fat LTO pipeline, which does *not* interleave inlining and simplification.)

Before the CGSCC inliner, the module simplification pipeline performs some early cleanup. In particular this involes running a SimplifyCFG+SROA+EarlyCSE sequence on all functions, followed by some module-level passes, such as optimization of globals and IPSCCP, which performs inter-procedural constant and range propagation. I believe the purpose of the early function cleanup is to prepare for these module passes, and to make sure that non-trivial SCCs receive at least basic cleanup prior to inlining.

The function simplification pipeline
------------------------------------

The function simplification pipeline is the very core of the entire optimization pipeline. Unfortunately, this is also the point where it becomes hard to make high-level statements about its underlying design philosophy.

There is a saying that goes something like this (I couldn't find any source for it, but I'm also pretty sure I didn't invent it):

> Optimization is a science, phase ordering is an art.

There isn't really a way to determine the correct ordering of optimization passes from first principles. It's something that comes from experience, and incremental adjustments to handle this or that test case. In most cases, nobody will be able to tell you why a pass is scheduled in precisely the place it is.

The key constraint of phase ordering is compile-time. An easy solution to many phase ordering problems would be to just schedule a few extra pass runs, but this is rarely a worthwhile tradeoff. Fixing someone's pet use case by scheduling an extra GVN run and thus making everyone pay an extra 5-10% compile-time is not a good idea.

A key part of compile-time reduction is analysis sharing. Passes that depend on the same expensive analysis will be scheduled as a group. For example, the jump threading and correlated value propagation passes are always scheduled as a pair, because they both depend on the expensive lazy value information analysis, which provides information on value ranges.

The function simplification pipeline contains two primary loop pipelines. The loop pipelines also work in an interleaved manner, where a sequence of loop passes is first applied to an inner-most loop, and then outer loops.

The primary distinction between the two loop pipelines is that the former uses and preserves MemorySSA (primarily to drive loop invariant code motion), while the latter does not. Once again, this is driven by analysis-sharing concerns.

Summary
-------

In summary, this is roughly how LLVM's optimization pipeline looks like:

```
|-------------------------- default ----------------------------|

|------------- pre-link --------------|
|------------------------- post-link ---------------------------|

/-------------------------------------\
|       module simplification         |
|-------------------------------------|   /---------------------\
|                                     |   | module optimization |
|                    cgscc            |   |---------------------|   /---------\
| /-------\ /-----------------------\ |   |                     |   | backend |
| | early | |       inlining        | |   | vectorization       |   \---------/
| |cleanup| |function simplification| |   | runtime unrolling   |
| \-------/ \-----------------------/ |   \---------------------/
|                                     |
\-------------------------------------/
```

The cute "backend" box hides another optimization pipeline that I have ignored entirely here. Maybe I'll talk about it some other time.

Finally, it's worth mentioning that the optimization pipeline has a number of extension points where additional passes can be inserted either by the frontend, the backend or plugins. For example, frontends will usually add sanitizer passes using this mechanism.

It is also possible to have an entirely custom optimization pipeline. The standard pipeline is mostly tuned for Clang, but also works well for rustc, and I don't think rustc would benefit substantially from a custom pipeline. However, when it comes to more exotic languages (e.g. Java and other GC languages) a custom optimization pipeline may be necessary.


[PassBuilderPipelines]: https://github.com/llvm/llvm-project/blob/main/llvm/lib/Passes/PassBuilderPipelines.cpp
[scc]: https://en.wikipedia.org/wiki/Strongly_connected_component
