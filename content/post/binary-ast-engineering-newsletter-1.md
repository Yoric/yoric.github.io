+++
title = "JavaScript Binary AST Engineering Newsletter #1"
tags = ["JavaScript", "Mozilla", "parsing", "ast", "compiler", "performance", "SpiderMonkey", "Babel", "newsletter"]
categories = ["JavaScript", "Mozilla", "performance", "binjs", "newsletter", "research"]
date = "2017-08-18T13:46:54+02:00"
+++

Hey, all cool kids have exciting Engineering Newsletters these days, so
it's high time the JavaScript Binary AST got one!


# Summary

JavaScript Binary AST is a joint project between Mozilla and Facebook to
rethink how JavaScript source code is stored/transmitted/parsed. We
expect that this project will help visibly speed up the loading of large
codebases of JS applications and will have
a large impact on the JS development community, including both web
developers, Node developers, add-on developers and ourselves.


# Context

The size of JavaScript-based applications – starting with webpages –
keep increasing, while the parsing speed of JavaScript VMs has basically
peaked. The result is that the startup of many web/js applications is
now limited by JavaScript parsing speed. While there are measures that
JS developers can take to improve the loading speed of their code, many
applications have reached a situation in which such an effort becomes
unmanageable.

The JavaScript Binary AST is a novel format for storing JavaScript code.
The global objective is to decrease the time spent between
first-byte-received and code-execution-starts. To achieve this, we focus
on a new file format, which we hope will aid our goal by:

- making parsing easier/faster
- supporting parallel parsing
- supporting lazy parsing
- supporting on-the-fly bytecode generation
- decreasing file size.

For more context on the JavaScript Binary AST and alternative
approaches, see [the announcement](https://yoric.github.io/post/binary-ast-newsletter-1/).


# Progress

## Benchmarking Prototype

The first phase of the project was spent developing an early prototype
format and parser to validate our hypothesis that:

- we can make parsing much faster
- we can make lazy parsing much faster
- we can reduce the size of files.

The prototype built for benchmarking was, by definition, incomplete, but
sufficient to represent ES5-level source code. All our benchmarking was
performed on snapshots of Facebook’s chat and of the JS source code our
own DevTools.

While numbers are bound to change as we progress from a proof-of-concept
prototype towards a robust and future-proof implementation, the results
we obtained from the prototype are very encouraging.

- parsing time x0.3 (i.e. parsing time is less than a third of original time)
- lazy parsing time x0.1
- gzipped file size vs gzipped human-readable source code x0.3
- gzipped file size vs gzipped minified source code x0.95.

Please keep in mind that future versions may have very different
results. However, these values confirm that the approach can
considerably improve performance.

More details in [bug 1349917](https://bugzilla.mozilla.org/show_bug.cgi?id=1349917).


## Reference Prototype

The second phase of the project consisted of building a second prototype
format designed to:

- support future evolutions of JavaScript
- support annotations designed to allow safe lazy/concurrent parsing
- serve as a reference implementation for third-party developers

This reference prototype has been implemented (minus compression) and is
currently being tested. It is entirely independent from SpiderMonkey and
uses Rust (for all the heavy data structure manipulation) and Node (to
benefit from existing parsing/pretty-printing tool Babylon). It is
likely that, as data structures stabilize, the reference prototype will
migrate to a full JS implementation, so as to make it easier for
third-party contributors to join in.

More details [on the tracker](https://github.com/Yoric/binjs-ref/).


## Standard tracks

As any proposed addition to the JavaScript language, the JavaScript
Binary AST needs to go through standards body.

The project has successfully been accepted as an ECMA TC39 Stage 1
Proposal. Once we have a working Firefox implementation and compelling
results, we will proceed towards Stage 2.

More details [on the tracker](https://github.com/syg/ecmascript-binary-ast/).



# Next few steps

There is still lots of work before we can land this on the web.


## SpiderMonkey implementation

We are currently working on a SpiderMonkey implementation of the
Reference Prototype, initially without lazy or concurrent parsing. We
need to finish it to be able to properly test that JavaScript decoding
works.

## Compression

The benchmarking prototype only implemented naive compression, while the
reference prototype – which carries more data – doesn’t implement any
form of compression yet. We need to reintroduce compression to be able
to measure the impact on file size.



# How can I help?

If you wish to help with the project, please get in touch with either
Naveed Ihsanullah (IRC: `naveed`, mail: `nihsanullah`) or myself (IRC:
`Yoric`, mail: `dteller`).


