---
title: "HolyJIT: Could we write a multi-staged JIT compiler?"
date: 2018-03-06T07:33:25+01:00
lastmod: 2018-03-06T07:33:25+01:00
draft: true
keywords: []
description: ""
tags: ["mozilla", "Rust", "jit", "javascript"]
categories: ["mozilla", "programming languages", "research"]
author: "David Teller"
---

> **Credit**
>
> While I'm the author of this blog post, 95% of the work was done by Nicolas B. Pierron.
>
> So far, my role in this project has largely been to play the wise old advisor,
> nodding and smiling mischeviously whenever Nicolas started exploring new ideas,
> and emitting cryptic comments in Reverse Jedi Notation.

A few months ago, we published a [short (and mysterious) blog post](https://blog.mozilla.org/javascript/2017/10/20/holyjit-a-new-hope/) in which we
mentioned HolyJIT, an early research project towards a novel approach to writing JITs.

In this blog post, I would like to detail a bit more the ideas behind HolyJIT.

<!--more-->

# Anatomy of a JIT compiler

If you're not already familiar with JIT compilers, I suggest you first [take a look
at Wikipedia](https://en.wikipedia.org/wiki/Just-in-time_compilation) before proceeding,
because the rest of this blog post won't be very useful to you.

For performance reasons, real-world JIT compilers such as SpiderMonkey, v8 or
HotSpot often contain several implementations of the language. For instance:

- a pure interpreter;
- a naive compiler;
- an optimizing compiler.

The pure interpreter can start running immediately, which is very useful for code that
the program is only ever going to run once. The naive compiler will take some time to compile
the code fed to it, but will produce faster code once it's done, so it is useful for code
that is used repeatedly. Finally, the optimizing compiler will take much more time to
compile the code, but will produce much faster code, so it is useful for performance-critical code.

That's three implementations of the *same* language (sometimes more) in the *same* compiler.
In fact, the inline cache strategy often contains a fourth implementation of large subsets
of the the same language.

This has two large drawbacks:

- if two implementations disagree on behavior, the language is broken in subtle and unpredictable
    ways, which is really bad for safety and security;
- any change to one implementation must be mirrored in all others, which is generally pretty tricky.

Can we do better?

# Our objective

The objective with HolyJIT is to see if we can develop a toolbox to:

- write JITs in a strongly-typed programming language (in this case, Rust);
- have a single specification as source of the semantics of the target language, regardless of the number of components that are actually going to implement different execution/compilation strategies;
- share these instruments across a large variety of JIT-compiled target languages;
- reduce the size of the Trusted Computing Base.

Some of the ideas in HolyJIT are closely related to multi-stage compilation,
as demonstrated for instance in MetaOCaml or Template Haskell. By opposition
to such languages, though, the target is explicitly JITs and aims to let JIT
developers pick or build their own strategies. By opposition to MetaOCaml,
which requires shipping a full OCaml compiler as part of the executable,
HolyJIT will not require shipping a full Rust compiler.

I should mention, again, that HolyJIT is early stage research. Some of what
will be discussed in this blog post is tested, some is just ideas.

For the rest of this post, I'll use *Host Language* to describe the language
in which the JIT is implemented (here, Rust) and *Target Language* to describe the
language which we are going to JIT-compile (for instance, JavaScript or regular expressions).

# A second look at implementations

Consider the pure interpreter. For the sake of simplicity, we'll assume
that this is an interpreter of bytecode.

It contains:

- for each opcode, a specification (as source code in the host language);
- a representation of state (typically, a stack, an environment, etc.);
- a main loop, going through each opcode.

Actually, assuming that the interpreter is compiled, during compilation,
the specification is translated to a Control Flow Graph, which is itself
translated to executable format. This executable format may be LLVM IR,
machine language, WebAssembly, etc.

Let's summarize this:

- specification as source code in Host Language;
- representation of state;
- main loop;
- translate specification to CFG;
- translate CFG to Executable.

The exact same scheme also describes both the naive native compiler
and the optimizing native compiler. Furthermore, since all need to
interact in the same executable, they typically share:
- the representation of state;
- the main loop.

In most JITs, the various implementations of the same language within
a single JIT differ:
- by having different specifications as source code;
- by producing different CFGs (or other intermediate representation);
- by generally translating to different executable formats (e.g. the interpreter
    is compiled by a C++ compiler, while the executable code produced by the
    native compilers is produced by some kind of runtime assembler).

Now, recall that these different specifications as source code are different
not because they implement different Target Languages but because the
specification contains both the semantics and the strategy used to
produce the executable.

Let's see what we can do about this.

# From the interpreter to the compiler

Consider operation `+` in JavaScript. If we are
building a stack machine, [the specifications](http://www.ecma-international.org/ecma-262/7.0/index.html#sec-addition-operator-plus) can
be translated straightforwardly as the following
opcode `Op::Add` and specifications-as-code:

```rust
opcode!(stack, Op::Add => {
    let lval = stack.pop()?;
    let rval = stack.pop()?;
    let lprim = lval.to_primitive()?;
    let rprim = rval.to_primitive()?;
    match (lprim.type_(), rprim.type_()) {
        // `+` may be used for string concatenation.
        (String, _) | (_, String) => {
            let lstring = lprim.to_string()?;
            let rstring = rprim.to_string()?;
            stack.push(JSString::concat(lstring, rstring)?)?
        }
        // `+` may be used for numeric addition.
        _ => {
            let lnum = lprim.to_number()?;
            let rnum = lprim.to_number()?;
            stack.push(JSNumber::add(lnum, rnum)?)?
        }
    }
})
```

In the above code, we have used a macro `opcode!`.
Rust macros are powerful enough to perform sophisticated
rewriting on the AST and communicate with the rest of the
toolchain.

We have also used `?` to represent all exceptional cases,
including both user-level exceptions (such as `Error`)
and VM-level exceptions (such as out-of-memory). Rust
makes it easy to not confuse them.

Once we have thus specified all opcodes, as well as
the main loop and the representation of state, we
have a fully working interpreter. When encountering
`Op::Add`, this interpreter will execute exactly the code above.

Now, assuming a sufficiently flexible toolchain (such as the
existing Rust toolchain), we can extract the CFG for each opcode
and use it twice:

1. convert the CFG to Executable;
2. keep the CFG as Data.

> **Build-time tool introduced** Conversion of specifications-as-source
> into CFG as Data.

The first CFG as Executable is exactly what we're already
using for our interpreter. With out-of-the-box Rust, this
is the native executable produced by `rustc`.

Using this CFG as Data, we can now implement the following
program, executed at runtime:

1. Read a chain of bytecode.
2. Convert chain of bytecode to a vector of CFG-as-Data.
3. Convert vector of CFG-as-Data to Executable.

This, along with the main loop, is a naive native compiler.
We have obtained this native compiler by introducing the
following tools in our toolchain:

> **Runtime tool introduced** Convert CFG-as-Data to Executable.

> **Runtime tool introduced** Combine blocks of CFG-as-Data.

These tools are entirely independent from the target language.

In effect, **we have extracted the compiler from the interpreter**.

While this result is not new (that's the entire point of Multi-Stage Compilation),
the technique we have employed is, as far as we can tell, both novel and more
flexible than alternative approaches.

For one thing, with a single specification as source, we have *both* the
interpreter and the compiler. This is a good thing, since we need both of
them to produce a high-performance JIT.

In particular, we can go further.

# From the compiler to the tracing compiler

**Caveat** At this stage of the project, this section is mostly hypothetical.

Consider, again, the implementation of `+` in JavaScript.
Let's say that we wish to implement a tracing JIT compiler
as an inline cache generator.

I assume it's a well-known result, but experience indicates
that we can turn an interpreter into a
naive tracing compiler by performing
a simple (and automated) rewrite of the CFG.

Let's inform our toolchain that we want to do this:

```rust
opcode!(stack, Op::Add => {
    let lval = stack.pop()?;
    let rval = stack.pop()?;
    trace!{
        let lprim = lval.to_primitive()?;
        let rprim = rval.to_primitive()?;
        match (lprim.type_(), rprim.type_()) {
            // `+` may be used for string concatenation.
            (String, _) | (_, String) => {
                let lstring = lprim.to_string()?;
                let rstring = rprim.to_string()?;
                stack.push(JSString::concat(lstring, rstring)?)
            }
            // `+` may be used for numeric addition.
            _ => {
                let lnum = lprim.to_number()?;
                let rnum = lprim.to_number()?;
                stack.push(JSNumber::add(lnum, rnum)?)
            }
        }
    }
})
```

So far, HolyJIT was extracting a single CFG from the specification
as source. Macro `trace!` instructs it to also extract a second
CFG (we'll call it a *Tracing CFG*), obtained by transforming
the *Interpreter CFG* to emit a trace of CFG-as-data.

For this, we need three tools:

> **Build-time tool introduced** Conversion of CFG into tracing
> CFG.

> **Build-time tool introduced** Introducing the CFG-as-data for inline
> caching.

> **Runtime tool introduced** Letting the CFG assemble CFG-as-data.

This new Tracing CFG, which was, once again, extracted
from the specification as source, may now be stored as
CFG-as-data, and executed much as the naive compiler.


In other words, **with minimal annotations, we can extract a tracing compiler from the interpreter**.

# FAQ

## What about security?

It is still too early to be sure. The ability to combine
CFGs dynamically and convert them to executable is basicaly
the same level of power as what current JITs do, so we don't
expect to make a difference here.

We hope that the ability to mostly decouple the compilation
strategy from the target language will make it easier to
fix security issues, though, and to experiment with new
security strategies.

## What about performance?

Our initial experiments lead us to believe that we can end
up with essentially the same generated code as existing JITs,
with the bonus that we can more easily experiment with
additional optimization strategies.

## How many ways do you need to translate CFG to Executable?

There are several possible implementation strategies. One
possible way is to generate a portable assembly format such
as WebAssembly. We could therefore translate CFG to WebAssembly both
at build-time (for the interpreter) and at runtime (for
the native compilers).

Of course, we may decide to generate directly optimized Executables
at build-time (for the interpreter), effectively reusing the existing
Rust infrastructure, and a different target at runtime.

## Isn't the CFG just a new bytecode format?

Somewhat, yes. Most JITs use at least one intermediate format
between the bytecode and the executable. The CFG here is
such a format. So, in effect, some kind of very high-level
bytecode.

## Wait, what about actually optimizing the CFG?

Yes, the discussion above didn't mention constant
propagation, dead code elimination, constant
subexpression elimination, ...

Of course, they need to be part of the result. At this
stage, there are several places at which they may be
added, but the most reasonable seems to be while
generating Executable from CFG. The main question
is whether we always want to apply these
optimizations or whether they should take place
only in some strategies. The former would be more
regular and easier to trust, while the latter
would let Target Language developers further
fine-tune their JIT strategies.

## Do you actually need a bytecode for JavaScript?

Not necessarily. This seems to be the simplest way to do
things, but it's not written in stone.

## How do you switch from the interpreter to the naive compiler to the optimizing compiler?

Not determined yet. If we wish our toolkit to be as generic as possible,
decision might be left in the hands of the implementor of the target language.

## Why Rust and not X?

Many other languages would work too. Rust combines the ability to
define low-level data structures and algorithms with strong type-safety,
no dependency on a specific garbage-collector, and a very flexible toolchain.

Plus, we like it.

## How close are you to release?

In a galaxy far, far away.

## Where can I find the code?

A proof of concept is available [on github](https://github.com/nbp/holyjit).