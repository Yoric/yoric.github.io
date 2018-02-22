- By Kannan Vijayan, Mike Hoye


_"The key to making programs fast is to make them do practically nothing."_
                      - [Mike Haertel, creator of GNU Grep](https://lists.freebsd.org/pipermail/freebsd-current/2010-August/019310.html).

Binary AST - "Binary Abstract Syntax Tree" - is Mozilla's proposal for specifying
a binary-encoded syntax for JS with the intent of allowing browsers and other
JS-executing environments to parse and load code as much as 80% faster than
standard minified JS.

It has recently cleared Stage 1 of the TC39 standards process, and while the
final byte-level format isn't completely nailed down, we're confident
that the final implementation will deliver the impressive performance
improvements promised by the prototype.

Before we get into the implementation details, however, we'd like to put some
common concerns about Binary AST to rest, most importantly that Binary AST is
not a new executable format or "bytecode for JS". It is a very efficient
repackaging of the same syntactic data but by design a Binary AST file
carries no more or less information than its JS source file, and can be
converted to and from valid JS source code with identical semantics.

As we work towards a production implementation, we'd like to share our reasoning
and motivations behind the project requirements and the design choices we've made.


## The Problem

David's [opening newsletter](https://yoric.github.io/post/binary-ast-newsletter-1) post covers the problem in detail.

... but in short: the modern Web a lot of JS, and every part of downloading,
parsing, compiling, optimizing and executing that JS is time-consuming.
While the world has made remarkable progress in the last five years on JS
compilation, optimization and execution, we haven't seen comparable progress
on the parsing front.

Even on fast modern hardware browsers spend more than 500ms parsing the Facebook home
page's 7MB of uncompressed JS. Other top sites like LinkedIn are comparable, and the 
numbers on lower end or older hardware are a lot worse. We have an opportunity to
make a real, tangible improvement in this space, and with that in mind our initial 
goal was to design a source format that allows for extremely fast parsing.

First, we should understand why JS parsing is currently so tedious. The _"What if we
just made the parser faster?"_ section in David's article details a number
of reasons, but there's a fundamental, unavoidable issue underpinning all of them:

*Parsing is slow because the parser has to look at and process every byte
of the source it loads, regardless of whether that code is about to run
or not*.

What's interesting is that JS engines already realize this to some degree
and try their best to avoid fully parsing as much code as they can get
away with.  This technique is called lazy parsing and all major JS engines
employ it. When code is first loaded, most functions are not fully parsed.
Instead a faster "syntax-error-only" parse is run to check for errors.
Later on, if the function is called, it will be re-parsed with a "full"
parser to generate bytecode for execution.

This technique delays heavyweight parsing until after page-load
on the hopes that the function is only called after page-load is finished
(or better yet never).  This increases the total time spent parsing the
function if it is ever called, but can be a worthwhile trade-off for
the improvement in start-up time.

Unfortunately we've basically reached the far end of the performance curve for
optimizing the parsing of existing JS syntax.  Worse, recent syntax
additions to the language, such as [destructuring syntax](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment), has added more ambiguity
and parsing complexity, resulting in slower parsing times overall.

To get a sense of why we've decided on the approach we took with Binary AST
it's worth a quick overview of how different platforms do program encoding, 
and the advantages and tradeoffs of each.

### Native Apps

Native apps are the obvious extreme when it comes to start-up performance.
Native binaries load and start executing almost instantly, and not coincidentally
native executable formats specify almost no verification of the executable data.

In this context load-time work involves parsing an executable file's section 
tables and doing initial linking, but
native program loaders will avoid the high cost of doing byte-level
verification of executables simply by not doing any verification at all. 
Consequently if a program is malformed the best-case outcome is that it
only crashes instead of doing something malicious or insane, but the good
news is that "do nothing" is something all computers can do really,
really fast.

It's important to note that native code formats (e.g. the ELF object file
format) treat their input as random access.  Aside from some key tables
describing the structure of the source, a native-code loader generally
never looks at a part of the source until it needs to execute it. 

The benefits here are obvious: this approach lets native programs load
and begin execution instantly and per-byte work for loading native code
is almost zero, because a native-code loader can ignore all loaded code
completely until it is run.

With all that in mind, the most important characteristic of native code
that we wanted to bring to Binary AST is that the per-byte work involved 
in loading and parsing code is very close to zero.

### Java

The original Java spec required heavy verification of a loaded class file.
The classes, class structure, functions, function signatures, and foreign
interface use must be verified across the class file.  Methods must have
their bytecode verified for stack and type consistency and dependent
class files may also be loaded and recursively verified.

This work is mitigated somewhat by Java's use of a binary classfile format.

Binary formats are generally much faster to parse than text representations of the same
content; see the `The Performance Binary Encoded Formats` section below for a quick treatise on why.
While Java takes advantage of a binary bytecode format to parse and
process the source much faster than it could plaintext code, Java's overall approach
is both cautious and very expensive. You end up with a very high degree
of confidence in the consistency and integrity of the execution environment,
but that comes at the cost of front-loading a lot of heavy work before you can run
a single instruction, which is not a great fit for the human experience of the Web.

### JavaScript

With no type checking, JS requires much lighter verification of loaded
code than Java. Only syntax checking is needed so, even though JS ships
a much higher-level source format that's harder to parse than Java's class
files, the per-byte work of verifying the source is much lower.

The tradeoff is that JS's text syntax forces parsers to tokenize the source
(split it up into atomic "words" before further decisions can take place) and then
re-construct the logical structure of the code. This is much more time consuming
than parsing binary structures.

The result is that even with the lighter verification burden, parsing is enough
of a bottleneck for JS load times that all major JS engines use the syntax-error-only
parsing technique to mitigate it on first load.

### Dart

Google's Dart language deserves a special mention here: Google's engineers
noted the hit to load-times incurred by heavy load-time verification and
decided to make all syntax errors lazy as well.

The lazification of syntax errors allowed Dart to greatly reduce the cost
of load-time parsing.  Code only needs to be scanned for a handful
of tokens such as brackets (for checking for consistent nesting and finding
the boundaries of function bodies) and strings and comments (because the parser
needs to ignore brackets within strings and comments).  These checks are simple
enough to be accelerated by SIMD data-parallel instructions.

Dart's approach was to design their language syntax and semantics to
allow the lazy "syntax-error-only" parser to run as fast as possible, and
the upshot of this is that per-byte work for loading Dart is much lower than
for JS.

Dart still used a plaintext source format, and was thus subject to
the same scanning requirements as all other plaintext formats: every byte of
the source must be scanned at load time. Having said that, the core idea of
designing language structures to make the fastest parsing techniques available as
efficient as possible is an extremely important one.

### Analysis

Reflecting on above examples drove us to reconsider our problem statement: the
best way for us to "fix" the parsing issue is to eliminate the need for
parsing code entirely unless that code is about to run.

Our goal with Binary AST then, is simple but aggressive: design a source format
that allows a parser to achieve load-time performance as close as possible to
native programs, by enabling the parser to skip looking at code
it's not about to execute.

The challenge is to do this while respecting and adhering to the
general ethos of the JS language, being easily accessible to
web developers, and having a good compatibility with the existing
JS ecosystem.  We'd like developers to have the same access to original source
and symbolicated exception stacks that they enjoy with plaintext
JS.

This set of constraints has led to the design decisions in Binary AST.
With the goal of eliminating unnecessary parsing, we identified and
addressed the aspects of plaintext JS that prevent that goal.

The design we arrived at is not so much a "new syntax" as it is
a pre-processed re-packaging of the information in a JS source file.
The Binary AST format can be converted to and from plaintext JS,
including no information that plaintext JS doesn't, and we intend
to ship bidirectional tooling alongside the final product to make
using and deploying Binary AST as easy as possible for web developers,
site owners, tool builders and browser vendors alike.

In subsequent articles, we'll address individual features in
the Binary AST proposal, and how those features enable to
the goals we have set out above.

## Why Not WebAssembly

One of the common questions we've seen so far is why WebAssembly doesn't
fit the role - in one of two ways: either to ask
developers to use wasm instead, or to translate JS code to wasm to ship.

Neither of these options are a viable solution to the problems we face.

One may tell developers to build "large apps" in wasm, but not all pages
or applications start out large.  Most pages are part of a continuously
updated, active platform that makes for difficult ground-up rewrites.
In any case, it's natural to expect JS codebases to grow as content
continues to become more dynamic.

The latter option of compiling JS to wasm before shipping is also infeasible
from the face of it.  Precompiling JS to CPU-granularity operations
for efficient run-time execution is basically impossible - the language
simply does not _allow_ for a static analyzer to peek very far into runtime
behaviour.  Common programs may easily use completely different types at the
same codesite depending on the input the program receives, which the static
analyzer has no way to guess at.  Precompiling JS to wasm would yield
an unbearably slow program.

Another proposal is that we could ship JS payloads as a wasm-compiled
VM engine that executes some custom "bytecode".  This approach faces the issue
that the page will have to ship the engine to the user in addition to the actual
program content, and then the engine will run on top of wasm.  If the engine is
a simple interpreter, it'll be slow.  If it's an optimizing, jit-compiling
interpreter, then the size of the engine you're shipping ahead of your content
becomes an even bigger issue, and the implementation needs to wait until
Wasm adds primitives to support garbage-collecting VMs.

For better or worse, the web has a favoured family of languages, and that
family is JS and any language that cleanly maps to it.  It is imperative
that we ensure that the most widely used language on the web is kept fast
and performant for the hundreds of millions of users that it serves every
day.

## Conclusion

We started off identifying and understanding the reasons parsing can be slow
and how to make it fast.  The analysis points to binary encoding as having
promise to improve parse times, and even more significant gains from a format
that allows the parser to "skip around".

David Teller implemented a prototype version of our ideas on a reduced subset
of JS, and was able to produce an 80% performance improvement in
parse times, with no loss in representational efficiency (size of file) compared
to minified JS.

On the basis of those results, we proposed and obtained Stage 1 clearance for
the feature in TC39.  We are currently in the process of working on 
a proper implementation that demonstrates these gains within a browser environment.

Please follow along as we further discuss the design choices of Binary AST in
future articles.


## Addenda: The Performance Of Binary Encoded Formats

For the purpose of completeness, I'd like to present a light treatise
on why parsing text formats is generally far slower than parsing binary
formats.  There are two major reasons:

### Reason 1

Parsing structure out of text is expensive.  For example, the integer
`126` would be represented as three characters in a text format.  The
pseudo-assembly code to parse integers in text would look something like:

```
  mov r0, 0                 # initialize result r0 to 0
loop:
  load *srcPtr, r1          # load next character.
  cmp r1, '0'               # 
  iflt end                  # break out if char < '0'
  cmp r1, '9'               #
  iflt end                  # break out if char > '9'
  mul r0, 10                # scale r0 up by 10 [EXPENSIVE]
  sub r1, '0'               # map '0'=>0, '1'=>1, etc.
  add r0, r1                # add the digit to the result
  add srcPtr, 1             # increment text position.
  goto loop
end:
```

A full execution of this code for the string '126' would run the loop
three times, executing 15 arithmetic instructions, 3 multiplies, 7 branches,
4 loads (one for each char in the integer, and then the char after that so
we know to terminate parsing), and a move.

In a binary format, we might use the 'varuint' representation for integers.
This is encoded by using 7 bits in each byte to encode the number, with one
bit indicating whether more bytes follow.  An encoding of '126' in varuint
format might be `126|0x80 == 0xfe` in binary.  Some pseudo-assembly to parse
this might be:

```
  mov r0, 0                 # initialize result r0 to 0
loop:
  load *srcPtr, r1          # load next byte
  tstbit r1, 0x80           # test stop bit.
  ifnz last                 # branch to "last" if stop bit = 1
  shl r0, 7                 # |
  or r0, r1                 # | add the byte to the result
  add srcPtr, 1             # increment text position.
  goto loop
last:
  and r1, 0x7f              # clear the stop bit from the byte.
  shl r0, 7                 # |
  or r0, r1                 # | add the byte to the result
```

A full execution of
this code for that input would run the loop exactly once, and perform 4
arithmetic operations, a single branch, a single load, and a move.

These are both slightly naive implementations of the approach, but they
serve to demonstrate the basic point.  If we compare the kinds and number
of instructions executed for parsing the former vs latter, we see the
following:

```
Instruction Type    Text Format     Binary Format
-------------------------------------------------
Fast Arithmetic         15               5
Multiplies              3                0
Branches                7                1
Loads                   4                1
Moves                   1                1
```

The binary encoding reduces the cheap arithmetic operations by more than 3x.
It drops branches, by 7x, and memory loads by 4x.  It eliminates multiplies entirely
and converts them into left-shifts.  Multiplies take around 3x the time of "simple"
arithmetic like shifts, compares, adds, and bit-operations.

The specific differential in operations executed is going to depend on the
specific hardware architecture, but this is an order of magnitude difference
in execution cost.

This property applies in general to most binary-encoded data.  It's structured
much more closely to the machine's native representations of data, and thus
is much faster for a program to convert into a usable internal form.

### Reason 2

Binary formats are expected to be produced by programs, not people.  There's no
expectation that a developer should be able to directly modify that binary encoding
after it is generated.  Instead of updating the binary encoding, we update
the source and re-translate to binary.  This means binary encodings
are "static".  Every part of a binary encoding can assume that the contents
of the rest of the encoding are exactly the same as when it was originally
produced.

This property allows binary formats to embed direct
references from one part of the file to another, which allows
the building of complex, directly navigable data structures in
the encoded text.

Plaintext source code does not allow for this, because the validity
of a direct reference from one part of the file to another would
be compromised as the programmer edits the source file.  Text source
must be scanned linearly, and there's no jumping around allowed.

This ability to embed "pointers" in the encoding also allows for
a much more machine-friendly structuring of information.
