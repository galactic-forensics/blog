+++
title = "Testing drivers in instrumentRs v0.2.0"
subtitle = "The mock interface"
date = "2026-06-19"

[taxonomies]
tags=["Rust", "instrumentRs"]
categories = ["Programming", "Science"]

[extra]
repo_view = false
+++

In the [last blogpost]
we discussed design ideas for `instrumentRs v0.2.0`.
Here, I will mainly focus on
the mock interface for testing instruments drivers
developed using `instrumenRs v0.2.0`.
If you haven't already done so,
it is highly recommended that you read the [last blogpost] first
for context.

<!--more-->

In addition to a mock interface,
proper error handling now also been implemented.
This, turns out, was in fact fairly straightforward in comparison to `v0.1.0`,
as `instrumentRs v0.2.0` will be fully generic over the interface.
The [thiserror] crate was, as usual,
invaluable in error handling and propagation.

## Testing instrument drivers

To test an instrument driver
we want to make sure that when the end user calls a given method,
the right bytes are sent to the instrument.
Furthermore,
when an instrument is queried,
the correct query command should be sent,
but then the received response should also be decoded correctly.

This kind of testing can be facilitated by using a mock interface.
A driver author can use this mock interface
to implement tests for the instrument driver.
These tests should allow maintaining, refactoring, etc.,
the instrument driver in the future
without having access to the actual hardware.

### What should a simple test look like?

Let us consider once more the simple [DigOutBox].
Each of the 16 channels of this box can be turned on or off,
but we can also receive the state of these channels individually.
The function calls for these two functions
in the instrument channel implementation
look like this:

```rust
pub fn get_channel(&mut self) -> Result<Channel, InstrumentRsError> {
    ...
}

pub fn set_channel(&mut self, val: Channel) -> Result<(), InstrumentRsError> {
    ...
}
```

Here, the set/get values are represented by a
[DigOutBox] specific `Channel`,
which is a simple `enum` like this:

```rust
pub enum Channel {
    /// The channel is on.
    On,
    /// The channel is off.
    Off,
}
```

The [DigOutBox command interface description] tells us
what the instrument driver
needs to send to the interface when communicating.
For setting/querying a specific channel this would look like this:

- Turn channel 3 on:
  - Driver sends to instrument `"DO3 1\n"`.
- Query status of channel 2:
  - Driver sends to instrument `"DO2?\n"`.
  - Instrument responds with `"1\n"` if the channel is on
    or`"0\n"` if the channel is off.

Here, `\n` is the end of line terminator used by the [DigOutBox].

To test turning the channel on,
we want to be able to write a test that might look somewhat like this:

```rust
#[test]
fn turn_on_channel3() {
    let expected_writes = vec!["DO3 1\n".as_bytes().to_vec()];
    let interface = MockInterface::new(vec![], expected_writes);

    let mut inst = DigOutBox::new(interface);

    inst.channel(3).unwrap().set_channel(Channel::On).unwrap();
}
```

Here, we are first defining what we expect to write to the instrument
in the `expected_writes` variable.
This is exactly the command that needs to be sent to the instrument
in order to turn on channel 3.
The `MockInterface` requires `expected_reads` and `expected_writes`
to be in the form `Vec<Vec<u8>>` (more on this later).
We then create a new `MockInterface`.
To this mock interface we provide the expected reads
(no expected reads in this case, thus `vec![]`)
and the expected writes.
Then we create a new `DigOutBox` instrument and feed it the mock interface.
Finally, we call the appropriate function and unwrap heavily (for now).
The `MockInterface` itself ensures
that the actual write is the same as the expected write
and that the `expected_write` and `expected_read` vectors
are fully depleted at the end of the test.

Let's do another example:
To test reading the status of channel 2
and assuming it is turned on in the response
would look like this:

```rust
#[test]
fn read_channel2_on() {
    let expected_writes = vec!["DO2?\n".as_bytes().to_vec()];
    let expected_reads = vec!["1\n".as_bytes().to_vec()];
    let interface = MockInterface::new(expected_reads, expected_writes);

    let mut inst = DigOutBox::new(interface);

    let ch2_state = inst.channel(2).unwrap().get_channel().unwrap();
    std::assert_matches!(ch2_state, Channel::On);
}
```

Here we now have an expected read and an expected write.
We create a new `MockInterface` with these values.
Then, identical to the example above,
we create a new `DigOutBox` instrument
and run the wanted command.
However, this time we store the returned value in `ch2_state`
and assert that it matches the state
that we told the interface to send back to the driver.

### Does the driver author experience matter?

So far so good,
however, this is still fairly clunky.
In `instrumentRs v0.2.0`, testing should be as straightforward as possible
for the driver author.
The following functionality should allow for this:

- Expected read and write values can be provided in a more general way
  that agrees with how humans think about what is sent back and forth.
- Setting up the instrument with the mock interface is too verbose
  in above examples and should be streamlined.
- If an error occurs, we want to provide useful and nicely formatted error messages.

All of these improvements can be implemented with two fairly simple macros
that aid the driver author with testing.
To understand how these macros are set up,
let us first discuss the mock interface implementation and what it actually does.

## The mock interface

First off, the mock interface is solely used for testing an instrument driver.
While the driver itself from the perspective of the end user will depend on `instrumentRs`,
the mock interface itself is not required by the end user.
We thus put the mock interface behind a feature.
This allows a driver author to set up a `Cargo.toml` file
as in the following excerpt:

```toml
[dependencies]
instrumentrs = "0.2.0"

[dev-dependencies]
instrumentrs = { version = "0.2.0", features = ["mock-interface"] }
```

Thus, the mock interface itself will only be available when testing,
but the end user does not suffer the additional code and dependencies.
This also allows us also to add some dependencies
to make testing easier/prettier.

### The implementation

The mock interface itself is implemented as:

```rust
pub struct MockInterface {
    /// What we expect the interface to read from the device.
    ///
    /// This is a flattened vector of all the bytes that we expect to read.
    expected_read: Vec<u8>,
    /// Index where we are currently at in the `expected_read` when reading.
    ///
    /// At the end, this must be equal to the length of the `expected_read` vector.
    read_idx: usize,
    /// What we expect the interface to write to the instrument.
    ///
    /// If filled, this is a flattened vector of all bytes we expect to write.
    expected_write: Vec<u8>,
    /// Index where we are currently at in the `expected_write` when writing.
    ///
    /// At the end, this must be equal to the length of the `expected_write` vector.
    write_idx: usize,
    /// Number of flushes we expect.
    ///
    /// Every time a full command is written the interface must be flushed. Thus, this number is
    /// equal to the number of full write commands.
    flush_exp: usize,
    /// Number of flushes counted.
    ///
    /// We expect one flush to be called for every full package that is sent to the device.
    flush_rec: usize,
}
```

We implement a `new` constructor for creating a `MockInterface` as following:

```rust
impl MockInterface {
    pub fn new(expected_reads: Vec<Vec<u8>>, expected_writes: Vec<Vec<u8>>) -> Self {
        ...
    }
}
```

The user in this case would provide `Vec<Vec<u8>>`s for the expected reads and writes.
Each inner `Vec<u8>` is one full command that is written or read
and must include the terminator, if required.
This also allows us to count the number of expected flushes of the instrument:
Each time a full command is written, the instrument must be flushed.
Thus, the number of counted flushes must agree with the length of `expected_writes`
that is supplied when constructing a new `MockInterface`.

To make the mock interface compatible with any instruments,
we also need to implement the `std::io::Read` and `std::io::Write` traits.
These implementations contain
the error checking of expected read/write versus received read/write.

**Mock errors**

In order to provide excellent errors,
`instrumentRs v0.2.0` provides a `MockError` type with nicely formatted messages.
Below example shows a simplified version of `MockError`:

```rust
pub enum MockError {
    UnexpectedWrite {
        expected: u8,
        expected_char: char,
        recieved: u8,
        received_char: char,
    },
    NoMoreReadData,
    NoMoreWriteData,
}
```

For clarity, all derives for [thiserror] were omitted.
These errors have implementations to turn them into `std::io::Error` types,
a requirement to return them from the implementations of
`std::io::Read` and `std::io::Write`.

The `UnexpectedWrite` error triggers when a byte in the data written to the interface
is not the same as the expected byte.
The error message will provide the mismatching expected and received byte
as a `u8` and as a `char`.

The `NoMoreReadData` is emitted
when the driver tries to read more data from the interface,
but the `expected_read` data has already been fully read.
This error will mainly trigger when working with instruments
that do not read up to a terminator but rather read a defined number of bytes.

The `NoMoreWriteData` error analogously will trigger
when the driver tries to write more data to the interface
than are expected from `expected_write`.

Below, we show how to interact in a nice and helpful way with these errors.

**Panics**

For proper testing,
we also want to make sure that all `expected_read` and `expected_write` bytes
were used up during the test
and that `flush` was called the appropriate amount of times during the test.
Since we pass ownership of the mock interface to the instrument driver,
we do not have access to it after the tests are finished
to call some kind of `finalize()`
function that checks for proper depletion and flushing.
However, we can call such a `finalize()` check when the interface itself is dropped
as following:

```rust
impl Drop for MockInterface {
    fn drop(&mut self) {
        if !thread::panicking() {
            self.finalize();
        }
    }
}
```

This will run the above mentioned checks and,
if any of them fail,
will panic.
If the thread the test is running in has already panicked,
we ignore these `finalize()` checks to let the user deal
with one problem at a time.
This also keeps the error/panic messages during testing neat.

<!-- markdownlint-disable MD026 -->
### The driver author experience matters!
<!-- markdownlint-enable MD026 -->

As demonstrated above, using using the `MockInterface` as it is defined is clunky
and leads to code that is hard to read.
Above examples have, e.g.,
many `.as_bytes().to_vec()` statements.
Let's make this more convenient!

To set up an instrument test, e.g., using the `DigOutBox` driver
as in the example above,
the `smock!` (synchronous mock) macro is provided.
This simplifies the test above to read the state of channel 2 to the following:

```rust
static TERM: &str = "\n";

#[test]
fn read_channel2_macro() {
    let expected_writes = ["DO2?"];
    let expected_reads = ["1"];

    let mut inst = smock!(DigOutBox, expected_reads, expected_writes, TERM);

    let ch2_state = inst.channel(2).unwrap().get_channel().unwrap();
    std::assert_matches!(ch2_state, Channel::On);
}
```

We first statically define the terminator since we will use it for all tests.
The `smock!` macro then takes the following arguments:

1. The instrument driver itself,
   which has, since it is an `instrumentRs` driver,
   a well defined constructor.
2. The values we expect to read from the device.
   These values can be provided in any iterable list of types
   that are compatible with the [Writable trait].
3. The values we expect to write to the instrument.
   Again, any iterable list of values compatible with the
  [Writable trait] are allowed.
4. (Optional:) The terminator, which must also simply be compatible
   with the [Writable trait].
   If a terminator is given, its byte(s) representation will be automatically appended
   to each expected read and write.

The `smock!` macro then returns the instrument itself loaded with the mock interface.
This structure allows us to define expected reads and writes
in an easy to understand form
and create an instance of the instrument
that is loaded with an adequate (synchronous) mock interface.

One more issue to address here is the simple unwrapping of the result.
While generally fine for a test,
it is not the most convenient way to debug failing tests.
Since the `MockInterface` must return `std::io::Error` types,
unwrapping a test error returns an error message that is not very pretty.
For example, if we change the expected write above to `"DOut2?"`,
we would get the following output when running with `cargo test`:

![Error message in test when simply unwrapping a result](/assets/08/unwrapping.png)

Even if you terminal window is wider,
this is an ugly error message.

We thus provide a second macro to unwrap mock errors in a pretty fashion.
The whole failing test would then turns into this:

```rust
static TERM: &str = "\n";

#[test]
fn read_channel2_macro_pretty() {
    let expected_writes = ["DOut2?"];
    let expected_reads = ["1"];

    let mut inst = smock!(DigOutBox, expected_reads, expected_writes, TERM);

    let ch2_state = u!(inst.channel(2).unwrap().get_channel());
    std::assert_matches!(ch2_state, Channel::On);
}
```

The `u!` macro simply matches on the result.
If the result is `Ok(inner)`, `inner` is returned.
If the result is however an `Err(e)`,
the error message `e` is formatted colorful
using [termcolor] and then printed to `stderr`.
After printing the macro panics to abort the test.
The terminal output now looks like this:

![Error message when using the provided `u!` macro instead of unwrapping results.](/assets/08/u_macro.png)

The error message is now nicely formatted:

- The error type itself is shown in bold red.
- The "expected byte" line, showing the expected byte and its character representation,
  is printed in blue, as it is likely the "good" value.
- The "received byte", showing the received byte and its character representation,
  is printed in yellow, as it is likely the "bad" value.
- The subsequent panic message indicates the proper line
  in which the test failed and adequately states so.

For driver authors that can, for some reason or another,
not display colorful terminal output,
a `uplain!` macro is also provided that does the same formatting,
however, does not color the output.

Note that the [termcolor] crate is of course only a dependency
when the "mock-interface" feature is activated.
Thus, this dependency will not be forwarded to the end user
as it is only necessary for testing/development.

### Testing the mock interface

In order to ensure that the mock interface is indeed doing what we want it to do,
two mock instruments (enough with the mocking already!)
were created that interact either with `&str`/`String` or with `&[u8]`/`Vec<u8>`
for the `W`/`WR` types used in the [transport module].
These integration tests allow us to keep the mock interface itself tested
such that the correct error messages get returned
and ensure that the mock interface itself keeps on working.

## What errors did I find?

After writing the mock interface and writing a few tests for [DigOutBox],
I quickly discovered the following two errors in the previous version
of my example driver:

- The number of flushes were in fact incorrect.
  The interface was at times flushed twice after a write.
- The terminator was appended twice
  when sending certain commands to the instrument.

When testing the mock interface itself,
I also found an error in how the read index advanced
when writing tests where no terminator is sent,
but an exact number of bytes is read.

Even if not getting/setting all properties of an instrument are tested,
a few tests can really smoke out errors in how the transport is implemented!
In case you needed it,
how I found my own bugs shows once more how valuable tests are.

## Next steps for `instrumentRs v0.2.0`

So what is next for `instrumentRs`?
The [previous next steps] still apply,
however, error handling and the creation of a mock interface
are finished for now.
I anticipate to next work on implementing `asycn` support
and, once worked out, write another blogpost about it.
The longer term outlook remains the same as published
in the [previous next steps] section.

## Comments, questions, ideas?

As always, if you have comments or questions,
please get in touch with me.
You can do so via [instrumentRs2 on GitHub],
[e-mail](mailto:reto@galactic-forensics.space),
[Bluesky],
or [Matrix].
I'm excited to hear your thoughts and ideas!

[DigOutBox]: https://github.com/galactic-forensics/digoutbox
[DigOutBox command interface description]: https://digoutbox.readthedocs.io/latest/firmware/#scpi-commands-and-serial-connection
[Bluesky]: https://bsky.app/profile/galactic-forensics.space
[instrumentRs2 on GitHub]: https://github.com/trappitsch/instrumentrs2
[last blogpost]: /blog/07-instrumentrs2
[Matrix]: https://matrix.to/@trappits:epfl.ch#/@trappits:epfl.ch
[previous next steps]: /blog/07-instrumentrs2#next-steps
[termcolor]: https://crates.io/crates/termcolor
[thiserror]: https://crates.io/crates/thiserror
[transport module]: /blog/07-instrumentrs2/#transport
[Writable trait]: /blog/07-instrumentrs2/#transport
