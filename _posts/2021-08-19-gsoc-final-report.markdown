---
layout: post
title:  "GSoC 21 Final Report: Concolic Tracing for LibAFL"
date:   2021-08-19 18:15:46 +0200
categories: 
---
This year I had the chance to add support for *concolic tracing* to the exciting and new [LibAFL fuzzing framework](https://github.com/AFLplusplus/LibAFL) as part of the great [Google Summer of Code](https://summerofcode.withgoogle.com) program.
This final report outlines my work from the last 10 weeks and gives an overview over the state of the project.

## Brief
I integrated [SymCC](https://github.com/eurecom-s3/symcc) and [SymQEMU](https://github.com/eurecom-s3/symqemu), two concolic tracing tools, which enables users of the LibAFL fuzzing framework to easily enhance their fuzzer with techniques based on concolic tracing.
Concolic tracing/execution is a rather specialized technique in the space of fuzzing that enables more directed and 'analytic' control over the target execution by applying ideas from [Symbolic Execution](https://en.wikipedia.org/wiki/Symbolic_execution).

The complete project was implemented across [several pull requests](https://github.com/AFLplusplus/LibAFL/pulls?q=is%3Apr+author%3Ajulihoh+) to the main LibAFL repository and a [separate fork](https://github.com/AFLplusplus/symcc) of SymCC under the AFLplusplus organization.
On a technical level this includes:

* [A new runtime](https://github.com/AFLplusplus/symcc/tree/main/runtime/rust_backend#readme) for SymCC and SymQEMU that facilitates the creation of new runtimes in languages other than C++ as part of the aforementioned fork of SymCC. Upstreaming of this new runtime is [in progress](https://github.com/eurecom-s3/symcc/pull/69).
* [Rust bindings](https://docs.rs/symcc_runtime) for the SymCC/SymQEMU [runtime interface](https://github.com/eurecom-s3/symcc/blob/master/runtime/RuntimeCommon.h) to facilitate the creation of SymCC/SymQEMU based concolic tracers in [Rust](https://www.rust-lang.org), the programming language of LibAFL.
* A library for building concolic tracers that are reusable and [compose well](https://docs.rs/symcc_runtime/0.1/symcc_runtime/macro.export_runtime.html), [including components](https://docs.rs/symcc_runtime/0.1/symcc_runtime/filter/index.html) that support the creation of entirely new runtimes.
* [A helper library](https://docs.rs/symcc_libafl) for using the SymCC instrumenting compiler from Cargo build scripts, as is common in LibAFL-based fuzzers.
* Support for [efficiently transferring](https://docs.rs/libafl/0.6.0/libafl/observers/concolic/serialization_format/index.html) concolic tracing expressions from the target program to a LibAFL-based fuzzer via shared memory [as part of the main LibAFL crate](https://docs.rs/libafl/0.6.0/libafl/observers/concolic/index.html).
* [A simple LibAFL Mutational Stage](https://docs.rs/libafl/0.6.0//libafl/stages/concolic/struct.SimpleConcolicMutationalStage.html) that uses concolic traces to increase coverage in the target program, similar to [SymCC's 'Simple' runtime](https://github.com/eurecom-s3/symcc/blob/master/docs/Backends.txt).
* [A very basic hybrid fuzzer](https://github.com/AFLplusplus/LibAFL/tree/main/fuzzers/libfuzzer_stb_image_concolic#readme) based on LibAFL that uses all of the components from this project.
* [A smoke test](https://github.com/AFLplusplus/LibAFL/tree/main/libafl_concolic/test#readme) that ensures that the integration between SymCC and LibAFL works properly which also [runs on CI](https://github.com/AFLplusplus/LibAFL/runs/3359607830?check_suite_focus=true#step:6:1).
* [A new chapter](https://aflplus.plus/libafl-book/advanced_features/concolic/concolic.html) in the [LibAFL book](https://aflplus.plus/libafl-book/) that introduces this project's work to LibAFL users.
* All code is thoroughly documented and documentation is available via [docs.rs](https://docs.rs).

## Goals

Since I managed to implement what I had planned, my original project proposal serves as a good summary of what I wanted to to achieve:

Building a real-world hybrid fuzzer into LibAFL is too ambitious for a GSoC project.
However, thanks to the nature of LibAFL, this doesn’t need to be a problem: LibAFL is a collection of components that can be used to build a concrete fuzzer.
Therefore, this project's goal should be to build the components that are necessary for hybrid fuzzing in a reusable manner.

More specifically, techniques in hybrid fuzzing can be divided roughly into two parts: 1. tracing the program’s execution, collecting path constraints, and 2. solving those path constraints in order to produce an input that takes a new, desired path.
From an architectural perspective, the first step is achieved by instrumentation and the second step usually involves an Satisfiability Modulo Theories (SMT) solver.

To be clear, it is not the goal to implement an instrumentation pass nor an SMT solver.
Those would be monumental tasks and there is plenty of work to build on ([SymCC](https://github.com/eurecom-s3/symcc)/[SymQEMU](https://github.com/eurecom-s3/symqemu)/[Kirenenko](https://chengyusong.github.io/fuzzing/2020/11/18/kirenenko.html)/[Triton](https://triton.quarkslab.com)/[Angr](https://angr.io) for instrumentation and [Z3](https://github.com/Z3Prover/z3) for solving).
Much rather, the goal is to integrate the existing tools into the framework.

This integration would involve the definition of a stable interface between the instrumented binary and the fuzzer.
This part of the project could be implemented as an Observer in LibAFL, which observes the path constraints that are generated by the instrumentation.
Additionally, a mutator needs to be built that takes these path constraints and solves them to generate new, mutated inputs.


#### The Observer

It may seem like integrating different instrumentation methods into a coherent interface is difficult, because they operate on different Intermediate Representations (IR’s) of the underlying code  (SymCC/QEMU on LLVM IR, Angr on VEX, etc.).
However, there is good precedence for this, as all of these tools feature integrations with Z3, which could be a great starting point.
In fact, SymCC and SymQEMU [share such an interface](https://github.com/eurecom-s3/symcc/blob/master/runtime/RuntimeCommon.h) and the constraints it generates can mostly be translated directly to Z3 queries.

On top of that, the authors of Kirenenko have [published a dataset](https://chengyusong.github.io/fuzzing/2021/03/08/constraints.html) that contains path constraints in a [format](https://github.com/chenju2k6/z3-test/blob/master/rgd.proto) that is close to what I am imagining.

On a technical level, the data could be written into shared memory, similar to how the coverage map is currently handled in AFL and LibAFL.

Note that in contrast to white-box testing (such as [KLEE](https://klee.github.io)), the communication between the instrumented binary and solver is unidirectional (ie. constraints are sent to the fuzzer without an expectation of a ‘response’).
Therefore, the communication does not require elaborate synchronization.

Concretely, a trace would be a linearization of the path constraints in the order in which they are generated.
The following should be a simplistic example:

```
[0]: Constant(value=0, bits=8)
[1]: Input(offset=13,value=1) // value contains the concrete value during the execution
[2]: Equals(left=0,right=1,taken=false) // referencing the previous two nodes
```

The corresponding C code would look something like: 
`if (inp[13] == 0) { /* … */ }`

This format can be efficiently generated inside the instrumented binary, keeping (comparatively high) instrumentation overheads as low as possible.

In addition to the pure expressions, the trace should contain location information at least for branches (ie. a branch would be tagged by the location in the binary).
This enables heuristics to avoid (re-)solving uninteresting constraints.

In fact, it would probably be wise to inject an AFL-like coverage map into the process as well and let the instrumentation filter out constraints based on whether a path was previously covered.
This could be part of the contract between the fuzzer and the instrumentor.

Since integrating many instrumentors is likely unrealistic given the timeframe, I would focus on integrating SymCC and SymQEMU (for source-based and binary-based targets respectively), accompanied by a written implementation proposal for other instrumentors.


#### The Mutator

The mutator would receive the path constraints as specified above and solve them.
An example implementation could simply translate the constraints to a Z3 query and solve it.
In case the query is satisfiable, a model is computed, which represents the mutations to apply to the input.
The goal should be to provide a simplistic implementation that can serve as an example to build more realistic approaches and to validate the correct functioning of the observer.


## Current State
I managed to reach all the goals I set in my proposal and in some aspects went beyond what I had set out to do:
The concolic tracing component is much more flexible than I had planned to make it, allowing for a broader range of applications.
Additionally, I also managed to provide an example of how concolic tracing can be used in LibAFL-based fuzzers and created continuous integration tests that should ensure the long-term quality of the implementation.

## Future Work
The project in its current state is already usable and it will be exciting to see how people will be able to make use of concolic tracing with LibAFL.
The following sections outline some ideas for extending the project in the future:

### Support for more Instrumentation Tools
From a user's perspective, however, there are still some barriers when it comes to actually instrumenting their particular target:
Instrumenting a target requires either using SymCC as an instrumenting compiler or SymQEMU for binary-only scenarios.
Both of these options are lacking in different ways:

* SymCC is a clang compiler plugin and is therefore inherently difficult to build (especially on non-linux targets).
* SymCC requires source and perhaps more importantly needs to be able to build the target. This usually tends to be more difficult than anticipated.
* Both SymCC and SymQEMU need to be built from source. Binary packages for popular operating systems are not available.
* While SymQEMU may in principle support many interesting targets (eg. non-x86, non-linux, embedded, etc.), it currently only supports x86 linux userspace tracing.

Therefore, it would be interesting to see more instrumentation tools supported, such as [frida](https://frida.re) or [Triton](https://triton.quarkslab.com), which may be easier to set up.

### Better Solving
Concolic tracing always involves some sort of solving of constraints to make use of the concolic trace.
This is an active field of research which ranges from effective filtering of constraints all the way to implementing specialized SMT-solvers.
The current state of the project already contains some filtering based on [QSym hybrid fuzzer](https://github.com/sslab-gatech/qsym), but it would be interesting to see whether more of the work in academia could be made usable by a wider range of users through LibAFL.
In general, the currently implemented solver is not very effective without being tuned to particular targets.

### Documentation of Learnings
During the project, I came across some issues that might be relevant for others.
I would like to document these findings for future reference.
Here is a brief overview of these findings:

* I managed to find a low-maintenance solution for re-exporting symbols from a C library in a Rust shared library (see [#2771](https://github.com/rust-lang/rfcs/issues/2771)).
It is [rather complicated](https://github.com/AFLplusplus/LibAFL/blob/main/libafl_concolic/symcc_runtime/build.rs), but works nicely for integrating SymCC's runtime.
Beware: it contains regular expressions to parse Rust code which is used to generate a C header on the fly to rename symbols and generates Rust macros that are used in the library to generate even more Rust code.
* The library can serialize expressions in a an LLVM IR-like language quite efficiently and the resulting organization of concolic tracers (especially regarding composability) is, in my opinion, quite profound and elegant.
I attempted to document the basic technical decisions and design in the [module documentation](https://docs.rs/libafl/0.6/libafl/observers/concolic/serialization_format/index.html), but I think a more long-form blog post would be in order.

## Personal Conclusion
I am very happy with how this GSoC project turned out.
The things I learned during the project are:

* Communicate early and clearly. Sometimes just formulating a problem for a discord message or github issue leads to the solution.
* Automated (integration) tests are key to solving difficult problems:
the ability to continuously verify that everything still works as expected enabled sweeping refactors that ultimately pushed this project from good to great (IMHO).
* Documentation is not nice-to-have, but a basic requirement. Documentation is also for the writer, not just the reader.
For me personally, it uncovered rough edges and code that I had simply forgotten to update or remove in a previous refactor. 
It makes it (sometimes painfully) obvious how (re-)usable an abstraction or piece of code _really_ is.

I am very grateful for the mentorship of [@domenukk](https://github.com/domenukk) and [@andreafioraldi](https://github.com/andreafioraldi).
Working with them was refreshingly productive and they were there to support me whenever I needed them, regardless of the problem or its difficulty.
Also: they are just [dufte](https://www.urbandictionary.com/define.php?term=dufte) dudes :)