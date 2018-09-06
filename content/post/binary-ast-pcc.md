+++
title = "JavaScript Binary AST diaries - How to replace proving with validating for fun and profit"
description = ""
tags = ["JavaScript", "Mozilla", "parsing", "ast", "compiler", "performance", "SpiderMonkey", "Shift", "newsletter", "pcc", "verification", "safety"]
categories = ["JavaScript", "Mozilla", "performance", "binjs", "newsletter", "types", "research"]
draft = false
date = 2018-02-23T10:44:39+01:00
+++

In this entry, I’d like to discuss one of the most interesting and unusual aspects of the Binary AST: how we gain performance by turning *proof-building* into *validation*, and why this is very good news for performance (and maybe not so good news for file size).

<!--more-->

# In the previous episodes

> The JavaScript Binary AST is a proposed compression format for JavaScript, designed to compress source code and make it much faster to load. This is a work in progress by Mozilla, Bloomberg and Facebook.


# Proofs, in JavaScript?

As every other programming language, the execution model of JavaScript could not work without some guarantees. For instance, you do not execute a JavaScript file if you do not first have a guarantee that it is syntactically correct. A file is not syntactically correct if it a labelled `break` statement uses a label that does not exist, etc.

Whenever a developer needs to deal with guarantees of this kind, the simplest way to do it without introducing bugs is to make sure that the program can only reach the point where the guarantees are necessary *after* all the checks have been completed. Quite often, the component in charge of performing the checks will issue some kind of object once the checks are complete, and this object is required to proceed with execution. If this is the only way of obtaining the object, the object is considered a *proof* that the checks have passed.


For instance, in JavaScript VMs, the component in charge of checking the syntax is called the Parser. The Parser is also the only component that can convert a JavaScript source code into the data structure known as an AST (Abstract Syntax Tree), which is required by the next step, the Bytecode Emitter. In this setting, the AST is a proof that the syntax has been checked. This proof is used by the the Bytecode Emitter, which will itself produce Bytecode, etc. 

Now, all of this is important to us because some of the verifications that the JavaScript VM needs to perform have a strong impact on performance.


# Direct evals and indirect evils

In addition to checking the syntax, the Parser must check many other, more subtle, properties of the program. For instance, consider the most dynamic construction in the JavaScript language: `eval`. This magic function takes a string and executes it immediately, as if it were a source code. From the point of view
of VM implementors, it’s scary because it can change the structure of the program itself in interesting and unpredictable ways:

```js

(function() {
  var a = 10;
  function foo(code) {
    eval(code);
    return a;
  }

  console.log(foo(""));      // "10"
  console.log(foo("var a")); // "undefined"
})();
```

(seriously, don't use `eval`).

Because `eval` is so powerful, making `eval` work in a JIT is very hard. Hard enough that it basically cannot optimize a function if it thinks that there may be a direct call to `eval` lurking somewhere in this function.

For this reason, in addition to checking that the syntax is valid, the Parser also checks which functions contain a direct call to `eval`. This information is stored in the AST and used later during the execution to decide which functions can be optimized and which cannot. In other words, the AST also serves as a *proof* that checking whether a function contains `eval` has been performed.

One of the reasons for which we don’t like checking for direct calls to `eval` is that it’s actually harder than it looks:

```js
(function() {
  var a = 10;
  function foo(code) {
    eval(code);
    return a;
  }
  console.log(foo("")); // 10
  console.log(foo("var a")); // still 10, this time

  function eval() {
    // Yes, by appending something to our source code, we have removed the `eval`.
  }
})();
```

or

```js
(function() {
  "use strict"; // Yes, the "use strict" is necessary here. You can try removing it and see what happens.
  var backup = eval;  

  (function() {
    var a = 10;
    var eval = backup; // This is eval, except it isn’t.
    function foo(code) {
      eval(code);
      return a;
    }
    console.log(foo("")); // 10
    console.log(foo("var a")); // still 10
  })()
})()
```

In theory, before the Parser could *build a proof* that there is no direct eval, it should check all the nooks and crannies of the function. For performance reason, Parsers cheat: they simply assume that
anything that looks like `eval(...)` is a direct call to `eval`, even if it is actually a call to another function that happens to be called `eval`. Despite this optimization, the Parser needs to visit all the blocks and all the nested functions of a function before it can build its proof. This is slow, complex,
and this cannot be skipped.


# From proof-building to validation

Now, the JavaScript Binary AST handles `eval` and a few other things a bit differently.

Firstly, in a Binary AST file, every function and every block contains an **Asserted Scope**,
which is a bag of additional properties, including a boolean flag `hasDirectEval`.

Secondly, the Binary AST Parser allows itself to throw instances of `SyntaxError` after the execution of the code has started (the name of the exception may yet change), in particular if it finds out that the
**Asserted Scope** is lying.

These two modifications change everything.

## Validation

Firstly, they change everything for the Parser, as the Binary AST Parser may now follow a much simpler algorithm:

- if we see a call `eval(...)` from a function or block with `hasDirectEval: false`, we were lied to,
throw a `SyntaxError`;
- if we see a block or a nested function, we can check its flag `hasDirectEval` – if it is `true`
and the parent function or block has `hasDirectEval: false`, we were lied to, throw a `SyntaxError`;
- otherwise, proceed.

This *validation* algorithm is considerably simpler than the original *proof-building* algorithm.

## Streaming compilation

Secondly, if you are familiar with Streaming Compilation, you may see that these changes are exactly the changes necessary to allow the language to
be stream-compiled:

- when encountering a function, the Bytecode Emitter can immediately read the `hasDirectEval` property
  and decide whether to emit optimized or deoptimized bytecode;
- at this stage, `hasDirectEval` may be lying, but we only need to make sure that we have finished checking 
  `hasDirectEval` before we actually execute the code.

In other words, we will be able to start emitting code much earlier, which means that we can start running code much faster. Mozilla recently released [a streaming compiler for WebAssembly](https://hacks.mozilla.org/2018/01/making-webassembly-even-faster-firefoxs-new-streaming-and-tiering-compiler/), which considerably improves the startup time for WebAssembly code. *Streaming compilation is a game
changer for WebAssembly performance and we expect that we will be able to achieve similar results for JavaScript with Binary AST*.

If you’re wondering why JavaScript VMs currently don’t provide streaming compilers, it is simply because the Bytecode Emitter currently needs to wait until the end of parsing to get the information it needs just to start generating the bytecode. This includes not only direct `eval`, but also declared variables or captured names.

These limitations are lifted by the Binary AST.

## Lazy parsing

Finally, this representation lends itself to lazy parsing. Where the original Parser needed to look at the entire function to infer that there is no direct eval, we may now:

- validate only we are certain we are going to execute, using the algorithm above, which may throw `SyntaxError`;
- if there was no `SyntaxError`, execute that code;
- later, once we know that we are going to execute a piece of code that hasn’t been validated yet, repeat the process.

Consider how this will affect JavaScript applications. Modern JavaScript applications quite often consist in a few main files and then one or several frameworks webpacked together. However, applications often only use a few of the features of these frameworks. Even when they use nearly all the features of these frameworks, they typically only use a few of them during startup. In either case, **lazy parsing can also be a game-changer in terms of performance, as it lets us start executing JavaScript code much, much faster**. 

If you’re wondering why JavaScript VMs currently don’t provide lazy parsers:

- the current semantics of SyntaxError require the Parser to check the entire source for syntax errors before executing;
- the current syntax of JavaScript make it so that the Parser simply cannot skip anything in the source code, because it cannot find the end of a block/function/string/... without parsing it first;
- many of the proofs needed by the Bytecode Emitter can only be built by inspecting the entire source code.

All three of these limitations are lifted by the Binary AST.

**Note** Modern JavaScript VMs implement something different that is also sometimes called lazy parsing, but still requires parsing the entire file. The "lazy" aspects consist in allocating memory to store the
proof only when that proof is needed, and require re-parsing parts of the file that have already been
parsed when the proof is needed. This does improve startup performance, but not nearly as much as we
can achieve with fully lazy parsing.

# Benchmarks?

At this stage, we are still knees-deep in the implementation of the Firefox/SpiderMonkey Binary AST parser. We are getting close to being able to provide benchmarks on full parsing, but lazy parsing
and streaming compilation aren't implemented yet. In other words, we do not have any reliable benchmarks yet. We do have extremely encouraging early proof-of-concept early benchmarks with and without lazy parsing, but we know better than to trust them, so we don’t want to publish them.

Also, there is a cost to storing in a file all the Asserted Scopes needed to perform lazy parsing. Very simply, it increases the file size, which increases the download duration. While we are actively working on minimizing the size increase, we also believe that the parsing speed improvements will make this worth.

# Further readings

- [Latest specifications of the JavaScript Binary AST](https://binast.github.io/ecmascript-binary-ast/)
- [Reference implementation (encoder and decoder)](https://github.com/binast/binjs-ref)  ([help wanted!](https://github.com/binast/binjs-ref/issues?q=is%3Aissue+is%3Aopen+label%3A%22Help+wanted%22))
- [Firefox/SpiderMonkey implementation status](https://bugzilla.mozilla.org/showdependencytree.cgi?id=1349917&hide_resolved=0).

If you are familiar with [Proof-Carrying Code](https://en.wikipedia.org/wiki/Proof-carrying_code), you have probably already realized that the vocabulary used in this entry is taken from PCC and related fields. While the techniques used here do not use formal logics or dependent type systems, they are a form to PCC.
