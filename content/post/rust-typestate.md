---
title: "Typestates in Rust"
date: 2018-02-22T15:15:55+01:00
lastmod: 2018-03-22T10:00:55+01:00
draft: false
keywords: ["Rust", "programming languages", "types"]
description: ""
tags: ["Rust", "programming languages", "types"]
categories: ["Rust", "programming languages", "mozilla"]
author: "David Teller"
---


A long time ago, the Rust language was a language with typestate. Officially, typestates were dropped long before Rust 1.0. In this entry, I’ll get you in on the worst kept secret of the Rust community: Rust still has typestates.

<!--more-->


# Wait, what’s a typestate?

Consider an object representing a file – let’s call that data structure a `MyFile`. Before a `MyFile` is opened, it cannot be read. Once a `MyFile` is closed, it cannot be read. In between, the file can be read. Typestates are a mechanism to let the type-checker check that you’re not making the following mistakes:

```rust
fn read_contents_of_file(path: &Path) -> String {
  let mut my_file = MyFile::new(path);
  my_file.open();  // Error: this may fail.
  let result = my_file.read_all(); // Invalid if `my_file.open()` failed.
  my_file.close();

  my_file.seek(Seek::Start); // Error: we have closed `my_file`.
  result
}
```

In this example, we have made two mistakes:

1. we have read a file that may not have been opened;
2. we have seeked in a file that was already closed.

In most programming languages, we can easily design the API
of `MyFile` to ensure that the first error is impossible,
simply by throwing an exception when the file fails to open.
Some standard libraries and frameworks decide to not follow
this principle for the sake of flexibility, but the capability
exists in the language itself.

The second error, however, is much harder to catch. Most
programming languages support the necessary features to
make this error hard, typically by closing the file upon
destruction or at the end of a higher-order function call,
but the only non-academic language that I know of that
can actually entirely prevent that error is Rust.


# Trivial typestates in Rust

How do we do this in Rust?

Well, the simplest way to do it is to introduce sane types
for our operations on `MyFile`:

```rust
impl MyFile {
    // `open` is the only way of constructing an instance of `MyFile`.
    pub fn open(path: &Path) -> Result<MyFile, Error> { ... }

    // `seek` requires an instance of `MyFile`.
    pub fn seek(&mut self, pos: Seek) -> Result<(), Error> { ... }

    // `read_all` requires an instance of `MyFile`.
    pub fn read_all(&mut self) -> Result<String, Error> { ... }

    // `close` takes an argument `self`, not `&self` or `&mut self`,
    // which means that it *moves* its object, effectively
    // consuming it.
    pub fn close(self) -> Result<(), Error> { ... }
}
impl Drop for MyFile {
    // Introduce a destructor to auto-close an instance of `MyFile`
    // if we haven't closed it yet.
    fn drop(&mut self) { ... }
}
```

Let's rewrite our above example:

```rust
fn read_contents_of_file(path: &Path) -> Result<String, Error> {
  let mut my_file = MyFile::open(path)?;
  // Note the `?` above. It's a simple operator that asks
  // the compiler to check whether the operation succeeded.
  // The *only* way to obtain a `MyFile` object is to
  // have a successful `MyFile::open`.

  // At this line, `my_file` is a `MyFile`, which means
  // that we may use it.

  let result = my_file.read_all()?; // Valid.
  my_file.close(); // Valid

  // Since `my_file.close()` consumed `my_file`,
  // this variable doesn't exist anymore.

  my_file.seek(Seek::Start)?; // Error, detected by the compiler
  result
}
```

This works even in more complex cases:


```rust
fn read_contents_of_file(path: &Path) -> Result<String, Error> {
  // As above.
  let mut my_file = MyFile::open(path)?;
  let result = my_file.read_all()?; // Valid.

  if are_we_happy_yet() {
      my_file.close(); // Valid
  }

  // Since `my_file.close()` consumed `my_file`,
  // this variable doesn't exist anymore in at least
  // one execution path.

  my_file.seek(Seek::Start)?; // Error, detected by the compiler
  result

  // If we haven't closed `my_file`, the destructor will close
  // it now.
}
```

The key here is that the Rust type system enforces the fact
that a variable may not be used after having been **moved**.
In our case, `my_file.close()` moved the value, effectively
consuming it.

Even if we had attempted to hide the variable somewhere
else, and reuse it after the call to `my_file.close()`,
we would have been thwarted by the compiler:

```rust
fn read_contents_of_file(path: &Path) -> Result<String, Error> {
  // As above.
  let mut my_file = MyFile::open(path)?;
  let result = my_file.read_all()?;

  let mut my_file_sneaky_backup = my_file;
  // Here, we have moved `my_file` to `my_file_sneaky_backup`.
  // So we cannot use `my_file` anymore.

  my_file.close(); // Error, detected by the compiler

  my_file_sneaky_backup.seek(Seek::Start)?;
  result

  // If we haven't closed `my_file`, the destructor will close
  // it now.
}
```

Let's try another way to cheat the compiler by making
the file accessible after it has been closed:


```rust
fn read_contents_of_file(path: &Path) -> Result<String, Error> {
  let my_shared_file = Rc::new(RefCell::new(MyFile::open(path)?));
  // Here, `my_shared_file` is a shared pointer to a mutable
  // instance of `MyFile`, pretty much as a regular
  // reference in Java, C# or Python.

  let result = my_shared_file.borrow_mut()
    .read_all()?; // Valid

  let my_shared_file_sneaky_backup = my_shared_file.clone();
  // We have cloned the pointer, creating a second manner
  // of accessing `my_shared_file`.

  // Let's make sure that we can use both the backup
  // and the orignal file:
  my_shared_file_sneaky_backup.seek(Seek::Start)?; // Valid
  my_shared_file.seek(Seek::Start)?; // Also valid

  // Ahah, now we can surely close `my_shared_file`
  // then manipulate `my_shared_file_sneaky_backup`,
  // as we would in Java, C# or Python!

  // Except we can't call `my_shared_file.close()`,
  // because the underlying `MyFile` is shared,
  // which means that nobody can *move* it
  my_shared_file.close(); // Error, detected by the compiler

  my_shared_file_sneaky_backup.seek(Seek::Start)?;
  result

  // If we haven't closed `my_file`, the destructor will close
  // it now.
}
```

Again, we have been thwarted by the compiler. Indeed, short
of explicitly going into `unsafe` mode, we will not be able
to break the invariant that `seek` cannot be used after `close`.

This example demonstrated the first brick of type states in
Rust: the **typed move operation**. So far, so good. But we are
only dealing with a trivial case, in which files only have two
states: **opened** and **closed**.

Let's see if we can handle more complex cases.

# Complex typestates

Instead of files, let us now consider the following (and admittedly pretty dumb)
network protocol:

1. Sender sends message "HELLO".
2. Receiver receives message "HELLO", responds with message "HELLO, YOU".
3. Sender receives message "HELLO, YOU", responds with random number.
4. Receiver receives number from sender, responds with the same number.
5. Sender receives same number from receiver, responds "BYE".
6. Receiver receives "BYE" from sender, responds "BYE, YOU".
7. Return to 1.

Any other message is ignored.

We can design our `Sender` (and similarly, `Receiver`) to
ensure that operations proceed in the right order. For
the moment, we don't care about identifying the
correspondant or the number.

For this purpose, we'll combine the **typed moves**
with another technique, well known to strongly-typed
functional programmers, known as **phantom types**.

```rust

// A series of 0-sized types representing
// the state of the sender. The value
// is not important, only the type
// (hence the name "phantom type").
struct SenderReadyToSendHello;
struct SenderHasSentHello;
struct SenderHasSentNumber;
struct SenderHasReceivedNumber;

struct Sender<S> {
  /// Actual implementation of network I/O.
  inner: SenderImpl;  
  /// 0-sized field, doesn't exist at runtime.
  state: S;
}

/// The following methods may be called regardless of the state.
impl<S> Sender<S> {
    /// The port used to connect the sender.
    fn port(&self) -> usize;

    /// Close the sender, once and for all.
    fn close(self);
}

/// The following method may be called only in a state SenderReadyToSendHello.
impl Sender<SenderReadyToSendHello> {
    /// Send hello.
    ///
    /// This method consumes the sender in its current state,
    /// returns it in a new state.
    fn send_hello(mut self) -> Sender<SenderHasSentHello> {
        self.inner.send_message("HELLO");
        Sender {
            /// Move the implementation of network I/O.
            /// The compiler is typically smart enough
            /// to realize that this requires no runtime
            /// operation.
            inner: self.inner,
            /// Replace the 0-sized field.
            /// This operation is erased at runtime.
            state: SenderHasSentHello
        }
    }
}

/// The following method may be called only in a state SenderHasSentHello.
impl Sender<SenderHasSentHello> {
    /// Wait until the receiver has sent "HELLO, YOU",
    /// respond with number.
    ///
    /// Return the sender in state `SenderHasSentNumber`
    fn wait_respond_to_hello_you(mut self) -> Sender<SenderHasSentNumber> {
        // ...
    }

    /// If the receiver has sent "HELLO, YOU", respond with number and
    /// return the sender in state `SenderHasSentNumber`.
    ///
    /// Otherwise, return the unchanged state.
    fn try_respond_to_hello_you(mut self) -> Result<Sender<SenderHasSentNumber>, Self> {
        // ...
    }
}


/// The following method may be called only in a state SenderHasSentNumber.
impl Sender<SenderHasSentNumber> {
    /// Wait until the receiver has sent number, respond "BYE".
    ///
    /// Return the sender in state `SenderReadyToSendHello`
    fn wait_respond_to_hello_you(mut self) -> Sender<SenderReadyToSendHello> {
        // ...
    }

    /// If the receiver has sent number, respond and return the sender
    /// in state `SenderReadyToSendHello`.
    ///
    /// Otherwise, return the unchanged state.
    fn try_respond_to_hello_you(mut self) -> Result<Sender<SenderReadyToSendHello>, Self> {
        // ...
    }
}
```

With this design, it is clear that the `Sender` can only follow the protocol:

- from step 1 (`SenderReadyToSendHello`, it may only go to step 3);
- from step 3 (`SenderHasSentHello`, it may only stay in step 3 or go to step 5);
- from step 5 (`SenderHasSentNumber`, it may only stay in step 5 or go back to step 1).

Any attempt to detract from the protocol will be blocked by the type system.

If you ever need to work with network protocols, device drivers,
industrial devices with specific safety instructions or OpenGL/DirectX/anything
else that requires you to perform complex interactions with the hardware,
you'll probably be happy to have such guarantees!

Welcome to the world of type states.

# Quick note: beyond type states

By the way, to continue on our network example,
what if we wanted to, say, store the number sent by the `Server` to
check that the response matched? Well, to do this, we can simply
store the number on `SenderHasSentNumber`:

```rust
struct SenderHasSentNumber {
    number_sent: u32,
}
```

Once again, the compiler will check that the code only accesses `number_sent`
when the sender is in state `SenderHasSentNumber`.

We will lose a (tiny) amount of performance, we the compiler won't be able to
optimize the transformation of the `Sender` between identical representations,
but that's generally worth it.

# Closing words

I hope that this quick demonstration has convinced you that the power of Rust
**typed move**, combined with **phantom types**,
is a great tool to ensure the safety of your code. It is used
all around the Rust standard libraries and many well-designed third-party
libraries.

For the moment, I am not aware of any other programming language with typed
move semantics (note: C++ has *untyped* move semantics), but I imagine that
other languages will eventually imitate Rust if this feature proves to be
well-liked. I, for one, couldn't do without :)

**edit** As remarked by Thomas Bracht Laumann Jespersen, a variant of
this method can also be used to encode session types in Rust. I'll
let you look at the [library](https://github.com/Munksgaard/session-types)
and the [paper](http://munksgaard.me/papers/laumann-munksgaard-larsen.pdf) for more details!