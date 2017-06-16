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
Rust has been available for years now (first stable release in May 2015), and has been
steadily improving. An interesting part of that approach focused on the compiler
is that, if someone actually finds a memory safety issue, instead of fixing it
in their code, they can improve the compiler so that nobody will ever get that issue
again. Do not fix bugs, fix bug classes.
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

Writing parsers manually in Rust is not enough. We can still find bugs, although
they are often less critical than the ones you would find in C applications
( https://github.com/rust-fuzz/trophy-case ). Parsing software correctly is hard,
and anybody can make mistakes.

So we use nom, a parser combinators library written in Rust. Parser combinators
are an interesting way to handle data. You assemble small functions, like one
that recognizes "hello", or one that recognizes alphanumerical characters,
and you combine them to make more complex parsers, through the use of combinators.
There are combinators for lots of cases, like "terminated", that would apply
two parsers in a row, then return the result of the first if both are successful,
or branching combinators that apply different parsers depending on the result of
a first one.

Those parsers are always functions with the same signature, which means even
complex parsers can be reused in other ones easily. You end up writing a lot of
small parsers, than you can test in isolation, and reapply them in larger parsers
as you see fit.
On the contrary, an approach based on parsers generated from a grammar often tend
to lack flexibility, and are harder to test. They are also quite restrictive in
what you can allow from the format you are trying to pass. On the other hand,
since nom parsers are just functions, you can perform whatever complex, ambiguous,
dangerous tasks you need to, and as long as the interface is the same, you can plug
that parser with other ones. This is an important property, since most formats are
badly designed and can require unsafe manipulations.

nom has been available for some time now, and has been used extensively for various
formats and protocols in production software.

Armed with a safe, low level language, and a parser library, we can now start rewriting
core parts of our infrastructure.

# How to replace part of a C application

Not all existing applications will support easily a rewrite of their parser. If that
part of the code is highly coupled with the rest, it will be problematic. Thankfully,
as said earlier, we do not need to rewrite everything. Find a restricted subset of the
parser, isolate it, rewrite it, then expand to other parts of the application.

The key is in defining the interface correctly. Deterministic functions are the easiest
to replace, and structures are usually the hardest, since multiple parts of the code
might use directly internal members of that structure (accessors are not a common
practice in C). But there are a lot of tricks one can use to help in the task.
As an example, commenting out a member of a structure and launching a build can
expose all of the uses of that member, which makes it easier to measure how much
work is needed.

When performing a rewrite, you will often need to import C code, and expose your Rust
code to C. You can write the Rust definitions and the C headers by hand, but Rust has
tools to automate this. Rust-bindgen can import C structures and functions from C,
and generate Rust bindings. While the generated code might be a bit complex at times,
it is a great way to start a project and generate code that you can edit later.
The opposite way works as well: you can employ rusty-cheddar to generate C headers from

The missing part for the integration is the linking phase: think of how you will link the Rust
part to the C part. Do you make a static or dynamic library? Do you generate an object
file that you feed to autotools? The rust compiler can generate any of those, and they
can then be handled by the buildsystem, be it autotools and makefiles, CMake, scons, etc.

On the build system side of things, Rust uses the cargo package manager
to download libraries (called crates), build and link them, publish new libraries and
applications. That tool greatly increases the productivity of Rust developers.
Unfortunately, the package management part requires an internet connection to download
packages, which might not be an option (do you expect your Makefile to make network calls?).
Fortunately, cargo is easy to extend with separate tooling. You can use cargo-vendor or
cargo-local-registry to download crates in advance and store them in an archive somewhere.
That way, you can freeze the dependency list of an application and make its compilation
reproducible, while keeping a simple way to update dependencies when needed.

# Start integrating some Rust

Once you have the buildsystem set up, you can start actually writing Rust code.
We would recommend that you develop the nom parser in a separate
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

The nom parser can return sub slices of the input without copying them, and will
guarantee
that the data is within the bounds. But in some cases, it does not even
need to see the whole input. As an example, for media formats, you would
read a block's header, let nom decide which type of block it is, and the
parser would tell you how many bytes of the block you need to send to the
decoder.

Here is the code of the TLS 1.3 ServerHello structure definition and message
parsing:
```rust
pub struct TlsServerHelloV13Contents<'a> {
    pub version: u16,
    pub random: &'a[u8],
    pub cipher: u16,

    pub ext: Option<&'a[u8]>,
}

pub fn parse_tls_server_hello_tlsv13draft18(i:&[u8])
    -> IResult<&[u8],TlsMessageHandshake>
{
    do_parse!(i,
        hv:     be_u16 >>
        random: take!(32) >>
        cipher: be_u16 >>
        ext:    opt!(length_bytes!(be_u16)) >>
        (
            TlsMessageHandshake::ServerHelloV13(
                TlsServerHelloV13Contents::new(hv,random,cipher,ext)
            )
        )
    )
}
```

This code generates a parser reading some simple fields, and an optional
length-value field for the TLS extensions (not parsed in that example), and
returns a structure. All error cases are properly handled, especially incomplete
data.


One characteristic of TLS is that the parsing of messages is context-specific: the
content of some messages cannot be decoded without having information about the
previous messages. For example, the type of the Diffie-Hellman parameters, in
the ServerKeyExchange message, depends on the ciphersuite from the ServerHello
message. Because of that, some variables are extracted and stored in a parser
context, associated to every connection, and passed to higher-level parser
functions.

Finally, the combinator features of nom are especially useful for protocols like
TLS: TLS certificates are based on X.509, which uses the DER encoding format. This
makes writing independent parser easier, for example as in the following code:

```rust
use x509::parse_x509_certificate;

/// Read several certificates from the input buffer
/// and return them as a list.
pub fn parse_tls_certificate_list(i:&[u8])
    -> IResult<&[u8],Vec<X509Certificate>>
{
    many1!(i,parse_x509_certificate)
}
```

Parsing an X.509 certificate is done by combining the DER parsing functions:
```rust
pub fn x509_parser(i:&[u8]) -> IResult<&[u8],X509Certificate> {
    map!(i,
         parse_der_defined!(
             0x10,
             parse_tbs_certificate,
             parse_algorithm_identifier,
             parse_der_bitstring
         ),
         |(_hdr,o)| X509Certificate::new(o)
    )
}
```



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

You could spend a large part of the work making the new parser bug compatible with
the old one. This is often a bad approach, since both parsers will probably not
recognize the exact same set of files. You only need to worry about recognizing
the same representative set of samples. Most C parsers are not even really tested
regularly anyway. If you still want to get close results to the original parser,
you could employ a smart fuzzer to do the work of testing the difference. Write
a program that wraps both the C parser and the new nom one, and that panics if
both parsers do not return the same result.

Once the parser is written and in the source, be happy, the "interesting" part
of the work will begin: getting it accepted in the tree, and deciding how you
will handle the software suddenly requiring a Rust compiler along with the old
C toolchain.

# Going further

This approahc of surgically rewriting parts of an application works well, as it is
designed to have a minimal impact on the original project. It can be used as a
stepping stone to start replacing larger parts of the application, once all the
details of build systems and developer training are handled.

But some projects could never handle that kind of precise touch. Some libraries,
still in active use today, have highly coupled spaghetti code, relying heavily
on goto or setjmp, and are basically untested and unmaintained. This is one of
the rare cases where we'd recommend rewriting the whole project in Rust. This is
a place where this language can shine: you could write a whole new library,
completely API compatible with the old one, that you could drop into package
managers as an alternative.

Think of how many parts of our infrastructure we could replace like this,
bit by bit. It's a titanous task, so we need to start now.
Talk about replacing a C library with an API compatible Rust one
