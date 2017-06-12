# Langsec and Rust and nom



   Langsec intro, needed or not?

we usually see our programs as some kind of engine or industrial machine,
that we set up and monitor, but largely runs by itself, except for the
occasional button push. That vision is flawed: our computers, operating systems
and programs are designed to modify their behaviour in complex ways
depending on their input.
The data you feed to your code, be it network packets, files, sensor data,
etc, drives your code, it's not the other way. That specific bit at that specific
address in the file decides whether your code goes into the if or the else
of that specific branch. Your application is in fact a virtual machine,
and its language is the input data. What can we do with this language?

Current practices: we can write safe, production ready code right now,
in safe languages, with sane parsing libraries, and maybe formal proofs

the problem: there's a huge body of code, already running in production,
that relies on unsafe practices.
libraries, services, operating systems, on servers, phones, microcontrollers, etc
it's written in C. Most of our basic libraries were written in the 90s
(like, libpng still has setjmp based error handling).
parsers are handwritten, with the parsing mixed within the state
machine. Some use huge switch statements, others use a lookup table for their
transitions.

we cannot rewrite those programs entirely: 10K to 10M lines of code.
Huge swats of unmaintained code, but also a large domain knowledge:
- lots of bugs were solved
- lots of experimentation with what's the right way to implement a spec
(because all specs are wrong)
- lots of developer knowledge (with the spec, and with the code itself)

rewriting means making a new project, while you still need to maintain the old
one. It also means converting developers to the new language and practice.
It's unfeasible in most cases.

what we propose: surgically replace a weak part of an application.
Parsers are a good target for this, since they can be well isolated
from the rest of the code.

we keep all the business domain code, that does things with real value.
Parsing has no value for the application, it should be done fast,
but that's all.
But it puts some constraints on how we must do it. It must be well
integrated with C code. The language we use must easily call C (FFI),
but also be called by C. A lot of languages assume they will be
the ones handling everything: the main event loop, sockets,
memory allocation, etc.
We cannot even have garbage collection, since that might affect
too much our host application. As an example, VLC media player
must synchronize its audio and video correctly.

We chose Rust for various reasons:
- safe memory handling
- no garbage collection
- great integration with C (it can even expose C compatible structures and functions)

We also chose nom, the parser combinators library, because
handwritten parsers in Rust can still make mistakes.
cf https://github.com/rust-fuzz/trophy-case

how to replace part of a C application:

The key is in defining the interface correctly. Rust has tools to automate
this: rust-bindgen to import C structures and functions from C,
rusty-cheddar to generate C headers from Rust code. Those tools
might not generate the best code, but they're a good start,
you can edit the headers afterwards.
When generating interfaces, think of how you will link the Rust
part to the C part. Do you make a static or dynamic library?
Do you generate an object file that you feed to autotools?
The rust compiler can generate any of those.
Integrating Rust libraries: Rust uses the cargo package manager
to download libraries (called crates). Unfortunately,
that requires an internet connection to download packages.
You can use cargo-vendor or cargo-local-registry to download
crates in advance and store them in an archive somewhere.

Once you have the buildsystem set up, you can start integrating
Rust code.
We would still recommend that you develop the nom parser in a separate
crate: that way, you can reuse it in other projects (Rust or other languages),
and you can employ Rusts's unit testing and fuzzing facilities.
Any fuzzing result can then be reused as test case for your parser.

nom parsers work well on byte slices, a Rust type that contains a pointer
and a length. You can easily transform any C buffer to this (show code example).
They never modify their input, they don't even need to own it.
This is important for integration in C applications: even if we know that
Rust code could be stronger than the rest of the application, it is still
a guest in someone else's house. If possible, let the host code handle
allocations, opening file, etc. This is a really good tip to apply,
because libraries with reentrant, deterministic functions without side
effects are easy to integrate, and IO is where most of the errors can happen.
This is also a part that (hopefully) has been stabilized long ago in the
host application.

The nom parser can return sub slices of the input, and will guarantee
that the data is within the bounds. But in some cases, it does not even
need to see the whole input. As an example, for media formats, you would
read a block's header, let nom decide which type of block it is, and the
parser would tell you how many bytes of the block you need to send to the
decoder.

Be wary of the high coupling that can appear between the parser and the rest
of the code in some C applications. This is where most of the work can happen.
This is usually the result of years of hacks upon hacks to add a feature
"quick and easy".
We usually recommend that the parser has a clear interface with the rest of the
code, in the form of a list small, deterministic parsers, and a reduced state
machine above it. Not a complete state machine intertwined with the parsing
(cf joyent's http parser), as those a re hard to debug and extend, nor a state
machine informally implemented via calls from other parts of the code.
The state machine is the main interface for the rest of the code: you feed it
data to parse, it decides which parser to apply depending on the current state,
changes its state depending on the data that was parsed (if successful), then
returns with info to drive the input consumption: how many bytes to consume,
or how many more bytes we need, or stop consuming if there was an error?
You can then query this state machine for the information you want, and for
data to write back to the network (in the case of a network protocol).


Talk about replacing a C library with an API compatible Rust one
