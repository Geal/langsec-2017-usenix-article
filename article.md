# Langsec and Rust and nom

A large part of our infrastructure is built on a sand castle. We have
been reusing the same code for decades, the same libraries written in the 90s,
the same applications, the same operating systems. We tried, and are still
trying, to maintain them in shape, patching bit by bit, mostly in reaction
to published vulnerabilities, sometimes as a proactive effort.
But all that old code is slowing us down.

And if that was not enough, to connect those pieces of code to each other,
we have pages and pages of unclear, ambiguous specifications for file formats
and network protocols. How can you be sure your implementation is correct
when some remove features, some add features, others implement them
incorrectly, and there are parts that are completely open to interpretation.
Also, let's mention that one broken application that generates incorrect
files everyone else must support.

Additionally, most of that software has been written in C (sometimes still
in the ancient C89), with unsafe practices and insufficient testing.

One could say it is a miracle all of this has worked until now, but
there is no luck in that. It is the patient work of thousands of developers
patiently fixing bugs, and system administrators monitoring failing services.
But we are losing the race now.

Attackers only get smarter, as what was difficult yesterday gets simple now.
And the tools only get smarter themselves. More vulnerabilities are published
everyday, while we keep the same old code and the same development practices.

## We cannot rewrite everything

Whatever the quality of all that code, we cannot replace it. Software gets
reused over and over, each generation of developers building upon what the
previous one built. There's much more churn in hardware than software <INSERT
PERRY METZGER QUOTE HERE>. We can write new software with better solutions,
but it would not fix the millions of devices currently in place, or the billions
of applications actually running.
Our only option is to strengthen the sand castle, bit by bit, until it can weather
the storm.

How could we achieve that? Even rewriting application by application or library
by library is a sisyphean task. Most of those projects are written in C,
from 10k to 10m lines of code. Large parts of that are unmaintained, but there's
also a huge domain knowledge embedded in the code. Thousands of bug fixes,
improvements, experimentations with the specifications were done over the
years. And the developers themselves carry most of this knowledge.
Rewriting a project completely means losing that knowledge, and hitting most
of those bugs the old project solved.
Not to mention, that rewriting the project entirely will create political issues,
require teaching the new ways to developers, all this while still maintaining
the old version.
This is impossible to do in most cases.

Here is what we propose: there are specific parts of applications and libraries,
weaker than the rest, that could be rewritten, while keeping all of the domain
knowledge present in the rest of the code. As part of the LangSec movement,
we concentrate on the parsers and state machines, since file formats and protocols
are the point of entry in most applications, and an often overlooked and vulnerable
part of the code.

The LangSec approach is in changing the way we view software:
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
By modeling correctly that input language, or restricting it to a manageable subset,
we can greatly reduce the attack surface of our applications, in their most
vulnerable elements.

If we replace the parsers and protocols in an existing application, we can
better protect it from the attacker's point of entry, while keeping the more useful
parts of the code running. To that end, we need languages and tools that can easily
integrate themselves inside a C application.

## Choosing the tools

We decided on using Rust for various reasons: the language is designed to avoid
memory vulnerabilities and development issues frequent in other languages.
There is no garbage collection: the compiler is smart enough to know when to
allocate and deallocate memory. Or it will not compile if the code cannot be safe.
With this, the compiler can protect your code from common flaws like double free,
use after free, add bounds check to buffers, etc. It is even able to know which
part of the code owns which part of the memory, and warn you when your code manipulates
data from multiple threads.
As you learn more Rust, you tend to rely more and more on the compiler to verify
the code, instead of keeping track of dozens of pointers in your head, thus freeing
you to think about the more valuable parts of the application.

Along with those features, Rust can work at the same level as C applications.
There's no runtime. There is no garbage collector (important in time critical software).
It can even work without an allocator. As an example, it can be used for embedded
development, from microcontrollers to larger CPUs.
To that end, Rust code can easily import C functions and structures and use them
natively, but the opposite works as well: you can expose functions and structures
to be used by C (or other languages) applications. This is a crucial part when
rewriting C code: sometimes, we have to expose and manipulate the exact same types
the target application is using.

Writing parsers manually in Rust is not enough





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
If the code is not highly coupled, you could even rewrite function by function,
since the Rust code can expose C compatible functions. Beware, though:
take the time to write a correct internal API for Rust code, since at some point,
you might stop exporting those functions and call the underlying functionality
directly from Rust.

fun thing to test when fuzzing: write code that uses both parsers, and that panics
when they don't give the same output.

Talk about replacing a C library with an API compatible Rust one
