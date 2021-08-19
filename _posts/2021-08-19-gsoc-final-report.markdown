---
layout: post
title:  "GSoC 21 Final Report: Concolic Tracing for LibAFL"
date:   2021-08-19 18:15:46 +0200
categories: 
---
This year (2021) I got the chance to add support for *concolic tracing* to the exciting and new [LibAFL fuzzing framework](https://github.com/AFLplusplus/LibAFL) as part of the great [Google Summer of Code](https://summerofcode.withgoogle.com) program.
This final report outlines the work I did during the last 10 weeks and gives an overview over the state of the project.

## Brief

I integrated [SymCC](https://github.com/eurecom-s3/symcc) and [SymQEMU](https://github.com/eurecom-s3/symqemu), two concolic tracing tools, which enables users of the LibAFL fuzzing framework to easily enhance their fuzzer with techniques based on concolic tracing.
Concolic tracing/execution is a rather specialized technique in the space of fuzzing that enables more directed and 'analytic' control over the target execution by applying ideas from Symbolic Execution.

The complete project was implemented in [several pull requests](https://github.com/AFLplusplus/LibAFL/pulls?q=is%3Apr+author%3Ajulihoh+) to the main LibAFL repository and a [separate fork](https://github.com/AFLplusplus/symcc) of SymCC under the AFLplusplus organisation.
The work includes, on a technical level:

* [A new runtime](https://github.com/AFLplusplus/symcc/tree/main/runtime/rust_backend#readme) for SymCC and SymQEMU that facilitates the creation of new runtimes in languages other than C++ as part of the aforementioned fork of SymCC. Upstreaming of this new runtime is [in progress](https://github.com/eurecom-s3/symcc/pull/69).
* [Rust bindings](https://docs.rs/symcc_runtime) for the SymCC/SymQEMU [runtime interface](https://github.com/eurecom-s3/symcc/blob/master/runtime/RuntimeCommon.h) to facilitate the creation of SymCC/SymQEMU based concolic tracers in [Rust](https://www.rust-lang.org), the programming language of LibAFL.
* A library for building concolic tracers that are reusable and [compose well](https://docs.rs/symcc_runtime/0.1.0/symcc_runtime/macro.export_runtime.html), [including components](https://docs.rs/symcc_runtime/0.1.0/symcc_runtime/filter/index.html) to aid in the creation of new runtimes.
* [A helper library](https://docs.rs/symcc_libafl) for using the SymCC instrumenting compiler from Cargo build scripts, as is common in LibAFL-based fuzzers.
* Support for [efficiently transferring](https://docs.rs/libafl/0.6.0/libafl/observers/concolic/serialization_format/index.html) concolic tracing expressions from the target program to a LibAFL-based fuzzer via shared memory [as part of the main LibAFL crate](https://docs.rs/libafl/0.6.0/libafl/observers/concolic/index.html).
* [A simple LibAFL Mutational Stage](https://docs.rs/libafl/0.6.0//libafl/stages/concolic/struct.SimpleConcolicMutationalStage.html) that uses concolic traces to increase coverage in the target program, similar to [SymCC's 'Simple' runtime](https://github.com/eurecom-s3/symcc/blob/master/docs/Backends.txt).
* [A very basic hybrid fuzzer](https://github.com/AFLplusplus/LibAFL/tree/main/fuzzers/libfuzzer_stb_image_concolic#readme) based on LibAFL that uses all of the components from this project.
* [A smoke test](https://github.com/AFLplusplus/LibAFL/tree/main/libafl_concolic/test#readme) that ensure that the integration between SymCC and LibAFL works properly that [runs on CI](https://github.com/AFLplusplus/LibAFL/runs/3359607830?check_suite_focus=true#step:6:1).
* A new chapter in the [LibAFL book](https://aflplus.plus/libafl-book/) that introduces this project's work to LibAFL users.
* All code is thoroughly documented and documentation is available via [docs.rs](https://docs.rs).

Perhaps surprisingly, all initial goals were met (and exceeded) and, so far, there is no outstanding/future work planned.

## Goals

Since I managed to implement what I had planned, my original project proposal serves as a good summary of what I wanted to to achieve:

> Hybrid fuzzing is a term used to describe techniques in fuzzing which involve concolic execution to drive the fuzzing campaign towards difficult-to-reach parts of the target program. In the academic world, this concept has received a lot of attention in the past few years and the results are promising. However, so far this technique does not seem to have escaped the lab yet, with many academic projects being difficult to use in practice.
> 
> AFL++ and LibAFL are projects that aim to make cutting edge fuzzing research usable in the real world and in the same vein, this project proposes to make hybrid fuzzing more approachable by providing the necessary components to do hybrid fuzzing with LibAFL.
> 
> 
> ### Concrete Goals
> 
> Building a real-world hybrid fuzzer into LibAFL is too ambitious for a GSoC project. However, thanks to the nature of LibAFL, this doesn’t need to be a problem: LibAFL is a collection of components that can be used to build a concrete fuzzer. Therefore, this project's goal should be to build the components that are necessary for hybrid fuzzing in a reusable manner.
> 
> More specifically, techniques in hybrid fuzzing can be divided roughly into two parts: 1. tracing the program’s execution, collecting path constraints, and 2. solving those path constraints in order to produce an input that takes a new, desired path. From an architectural perspective, the first step is achieved by instrumentation and the second step usually involves an Satisfiability Modulo Theories (SMT) solver.
> 
> To be clear, it is not the goal to implement an instrumentation pass nor an SMT solver. Those would be monumental tasks and there is plenty of work to build on ([SymCC](https://github.com/eurecom-s3/symcc)/[SymQEMU](https://github.com/eurecom-s3/symqemu)/[Kirenenko](https://chengyusong.github.io/fuzzing/2020/11/18/kirenenko.html)/[Triton](https://triton.quarkslab.com)/[Angr](https://angr.io) for instrumentation and [Z3](https://github.com/Z3Prover/z3) for solving). Much rather, the goal is to integrate the existing tools into the framework.
> 
> This integration would involve the definition of a stable interface between the instrumented binary and the fuzzer. This part of the project could be implemented as an Observer in LibAFL, which observes the path constraints that are generated by the instrumentation. Additionally, a mutator needs to be built that takes these path constraints and solves them to generate new, mutated inputs.
> 
> 
> #### The Observer
> 
> It may seem like integrating different instrumentation methods into a coherent interface is difficult, because they operate on different Intermediate Representations (IR’s) of the underlying code  (SymCC/QEMU on LLVM IR, Angr on VEX, etc.). However, there is good precedence for this, as all of these tools feature integrations with Z3, which could be a great starting point. In fact, SymCC and SymQEMU [share such an interface](https://github.com/eurecom-s3/symcc/blob/master/runtime/RuntimeCommon.h) and the constraints it generates can mostly be translated directly to Z3 queries.
> 
> On top of that, the authors of Kirenenko have [published a dataset](https://chengyusong.github.io/fuzzing/2021/03/08/constraints.html) that contains path constraints in a [format](https://github.com/chenju2k6/z3-test/blob/master/rgd.proto) that is close to what I am imagining.
> 
> On a technical level, the data could be written into shared memory, similar to how the coverage map is currently handled in AFL and LibAFL.
> 
> Note that in contrast to white-box testing (such as [KLEE](https://klee.github.io)), the communication between the instrumented binary and solver is unidirectional (ie. constraints are sent to the fuzzer without an expectation of a ‘response’). Therefore, the communication does not require elaborate synchronization.
> 
> Concretely, a trace would be a linearization of the path constraints in the order in which they are generated. The following should be a simplistic example:
> 
> 
> ```
> [0]: Constant(value=0, bits=8)
> [1]: Input(offset=13,value=1) // value contains the concrete value during the execution
> [2]: Equals(left=0,right=1,taken=false) // referencing the previous two nodes
> ```
> 
> 
> The corresponding C code would look something like: \
> `if (inp[13] == 0) { /* … */ }`
> 
> This format can be efficiently generated inside the instrumented binary, keeping (comparatively high) instrumentation overheads as low as possible.
> 
> In addition to the pure expressions, the trace should contain location information at least for branches (ie. a branch would be tagged by the location in the binary). This enables heuristics to avoid (re-)solving uninteresting constraints.
> 
> In fact, it would probably be wise to inject an AFL-like coverage map into the process as well and let the instrumentation filter out constraints based on whether a path was previously covered. This could be part of the contract between the fuzzer and the instrumentor.
> 
> Since integrating many instrumentors is likely unrealistic given the timeframe, I would focus on integrating SymCC and SymQEMU (for source-based and binary-based targets respectively), accompanied by a written implementation proposal for other instrumentors.
> 
> 
> #### The Mutator
> 
> The mutator would receive the path constraints as specified above and solve them. An example implementation could simply translate the constraints to a Z3 query and solve it. In case the query is satisfiable, a model is computed, which represents the mutations to apply to the input. The goal should be to provide a simplistic implementation that can serve as an example to build more realistic approaches and to validate the correct functioning of the observer.


