+++
title = "instrumentRs v0.2.0 design ideas"
subtitle = "Writing instrument drivers in Rust"
date = "2026-06-10"

[taxonomies]
tags=["Rust", "instrumentRs"]
categories = ["Programming", "Science"]

[extra]
repo_view = false
+++

Over the years I have written many drivers
to control scientific equipment.
These were mostly in python
using the wonderful [instrumentkit].
As my tech stack has mostly moved to Rust however,
I have missed a consistent, simple-to-use way
to write instrument drivers.

This post discusses a very rough idea
on how such an improved driver kit might look like in Rust.
In addition to inspiration from [instrumentkit],
a lot of inspiration is also taken from the fantastic [device-driver] tool.

<!--more-->

## Background

End of July 2025 I published version `v0.1` of
[instrumentRs on crates.io].
This version relied heavily on using a
[cargo-generate] template to facilitate writing a driver.
Implementing a few drivers using `instrumentRs v0.1`,
most of which are for good reasons still unpublished,
quickly made me realize how clunky the interface
in fact is.
Some major limitations include:

- No async support.
- Interfaces require the driver author to work with specific crates.
- Handling of setting/getting of unitful quantities is clunky.
- The interface itself is wrapped in an `Arc<Mutex<>>`.
- Just providing templates doesn't really provide the necessary convenience.

Overall, `v0.1` is far away from being a simple aid
to write an instrument driver in Rust.

You don't need to know anything about `v0.1`
in order to continue reading.
This background information should only serve
to give a bit of additional information.
With that being said, let's discuss some current ideas
for the `instrumentRs v0.2.0` implementation
and its advantages.

## Architecture of `instrumentRs v0.2.0`

Below figure shows a very rough overview of the architecture.
This should allow for maximal flexibility
while enabling minimal repeatability by automatically generating code.

```plain
╔═════════════════════════════════════════════╗                         
║                                             ║                         
║               Paramter trait                ║                         
║                                             ║                         
║        Instrument | InstrumentAsync         ║   DSL / Macro generation
║ InstrumentChannel | InstrumentChannelAsync  ║                         
║                                             ║                         
╚═════════════════════════════════════════════╝                         
                                                                        
                                                                        
╔═════════════════════════════════════════════╗                         
║                                             ║                         
║               Transport trait               ║   Implemented by driver 
║      Interface read / write functions       ║           author        
║                                             ║                         
╚═════════════════════════════════════════════╝                         
                                                                        
                                                                        
╔═════════════════════════════════════════════╗                         
║                                             ║                         
║                  Interface                  ║    Provided by end user 
║  <Read + Write> | <AsyncRead + AsyncWrite>  ║                         
║                                             ║                         
╚═════════════════════════════════════════════╝
```

### The `Instrument`, `InstrumentChannel` structures and the `Parameter` trait

This part is the core of every instrument driver.
It describes how the end user interacts with the driver, i.e.,
what functions are available for getting and setting various instrument properties.
What does this entail?

Each instrument holds a generic interface `<I>`
that implements `Read` and `Write` in the blocking
and `AsyncRead` and `AsyncWrite` in the asynchronous case.
This interface encapsulates how to communicate with the instrument.
Furthermore, we need to consider:

- Instrument configurations.
  - Configurations, e.g., the number of channels can vary between different models
    while the communication protocol stays the same.
    Thus, such configurations might have to be end-user configurable.
  - Certain parameters are also fixed, e.g., how a command is terminated
    is often not changeable.
- Instrument-wide properties, e.g., the brightness of the instrument's display.
  - These are properties that are defined instrument-wide.
  - Properties can be read-only, write-only, or read-write.
  - Unitful properties should be handled in a unitful way.
- Instrument channels.
  - Each channel will integrate the same properties, e.g.,
    measure a voltage.
  - Channels can also have read-only, write-only, or read-write properties.
  - Unitful properties should be handled in a unitful way.

**Code for the core implementation of an instrument
should be automatically generated
using a to-be-defined domain specific language (DSL)
and/or appropriate macros.**

To allow for an automatically generated instrument core
we need to make sure that the types created by the driver author
can automatically be transformed into
values that are received from/written to the instrument.
In other words, these parameters must be "writable".
To make this possible, the macro will also define
a driver specific `Parameter` trait that will look somewhat like this:

```rust
pub trait Parameter<W: Writable>: Sized {
    fn to_writable(&self) -> W;
    fn try_from_writable(val: W) -> Result<Self, InstrumentRsError>;
}
```

Here `W` is the writable trait that is defined in `instrumentRs`.
More on this trait in the [transport section].

For any type that the driver author wants to make available to the end user
for interaction with the instrument,
the `Parameter` trait must be implemented.
Keeping the `Parameter` trait in the instrument driver itself
and not as part of the `instrumentRs` crate allows the driver author
to also implement it on basic types.

Two examples on how to implement the `Parameter` trait:

1. A channel that we can turn on and off, implemented as a new type.

   ```rust
   /// State of the channel.
   #[derive(Clone, Copy, Debug)]
   pub enum Channel {
       /// The channel is on.
       On,
       /// The channel is off.
       Off,
   }

   impl Parameter<String> for Channel {
       fn to_writable(&self) -> String {
           match self {
               Channel::On => "1".to_string(),
               Channel::Off => "0".to_string(),
           }
       }

       fn try_from_writable(val: String) -> Result<Self, InstrumentRsError> {
           match val.trim() {
               "0" => Ok(Channel::Off),
               "1" => Ok(Channel::On),
               _ => Err(InstrumentRsError::PlaceholderError),
           }
       }
   }
   ```

   Here, we implemented the `Parameter` trait.
   What this means is that when sending a channel state to the instrument,
   a `"0"` or a `"1"` would be sent (as a byte).
   When reading from the instrument, i.e., getting the state of a channel,
   the instrument in this case would send back a `"0"` or a `"1"`.
   Since we could get a bogus answer from the instrument as well,
   we implement a `try_from_writable` that can fail gracefully.
2. The same functionality could also be implemented by defining that
   `Channel` on and off can be set with a `bool`, e.g.,
   `true` for on and `false` for off.
   This is not the most idiomatic way, but let's just take it as an example.
   In this case, the driver author would implement the following:

   ```rust
   impl Parameter<String> for bool {
       fn to_writable(&self) -> String {
           if *self {
               "1".to_string()
           } else {
               "0".to_string()
           }
       }
   
       fn try_from_writable(val: String) -> Result<Self, InstrumentRsError> {
           match val.trim() {
               "0" => Ok(false),
               "1" => Ok(true),
               _ => Err(InstrumentRsError::PlaceholderError),
           }
       }
   }
   ```

   Of course, this also limits every `bool` now to be sent with these
   specific patterns, however,
   this is generally the case when certain settings on an instrument
   take `true` or `false`.
   This example however demonstrates why the `Parameter` trait
   must be implemented in the instrument driver itself.
   Defining it in `instrumentRs` would only allow a driver author
   to implement it on their own new types due to the [orphan rule].

  Note that in these examples,
  `Parameter` is implemented with an owned `String` as the writable type.
  The reason for this is that the bytes that come back in a query
  are parsed into a `String` before being passed on.
  More on this in the [transport section].

### Transport

The transport is how the commands get written to the instrument.
Various implementations exist, from simple [SCPI] instruments
to [MODBUS] communication protocols.
We want to be flexible!
I have even seen instruments that pack the bytes in a predefined structure
but mix the [endianness] depending on where in the command we are at.
Also, often some type of checksum is used that needs to be calculated
once the data package is assembled
and then added to the package before sending it.

Generally, we have two types of commands: `sendcmd` and `query`.
In the `sendcmd` case we usually want to just send a command
to the instrument and maybe check for an acknowledgment.
The `query` command generally first sends a command to request data
and then reads the interface to receive the data.
Generally, a driver author will need to implement the
`Transport` trait for their instrument.
This trait is fairly simple
and in the current `v0.2.0` implementation looks like this:

```rust
pub trait Transport<W: Writable, WR: Writable> {
    /// The send command that you need to implement.
    fn sendcmd(
        &mut self,
        cmd: W,
        idx: Option<usize>,
        args: Option<&[W]>,
    ) -> Result<(), InstrumentRsError>;

    /// The query command that you need to implement.
    fn query(&mut self, cmd: W, idx: Option<usize>) -> Result<WR, InstrumentRsError>;
}
```

The arguments that `sendcmd` and `query` require are as following:

- `cmd: W`: The command (property) that we want to write to.
- `idx: Option<usize>`: Optionally gives the channel index if this is a channel
  specific setting.
- `args: Option<&[W]>` A slice of the parameters that are necessary to
  assemble the final command that is sent to the instrument.
  These arguments are, e.g., the actual voltage we want to set to a channel.

The `sendcmd` function has to assemble the package, i.e., the whole command
that will be sent to the instrument including the terminator,
and then sends it to the instrument via the interface.
Upon success, it will return an `Ok(())` or an error upon failure.
Similarly, the `query` function will send a "give me this data" command to the instrument
and then return the result.

You might have noted that there are two generics in this trait,
both of which implement the `Writable` trait.
Why is this the case?

The `Writable` trait itself is in fact simply a very small helper trait
that allows us to accept various different ways of defining
what can be sent to an instrument.
It looks as following:

```rust
pub trait Writable {
    fn to_byte_slice(&self) -> &[u8];
}
```

All we have to expect from a type that can be sent to an instrument
is that we can represent it as a byte slice.

The reason that the `Transport` trait defines `W` for the to-send
types and `WR` for the return types is that
we are allowed to send borrowed values, e.g., `&str` to an instrument,
however, we need to return an owned value, e.g., `String`.
`instrumentRs` will already implement the `Writable` trait
for all necessary basic types.

Finally, `instrumentRs` also provides helper functions for the driver author.
These are used to simplify standard ways of sending to and querying the interface.
For example, the following helper function reads a reply byte by byte
until the terminator is detected and then returns a `Vec<u8>` of this byte.
How this `Vec<u8>` is then parsed is instrument dependent
and thus it will be up to the driver author to decide.
If no terminator is detected, the loop will generally break
due to some type of a time out error from the interface.

```rust
pub fn read_until_terminator<I: Read>(
    interface: &mut I,
    terminator: &[u8],
) -> Result<Vec<u8>, InstrumentRsError> {
    let mut ret = vec![];

    let mut buf = [0u8];
    loop {
        interface.read_exact(&mut buf).unwrap();
        ret.push(buf[0]);

        if ret.len() >= terminator.len()
            && ret.get(ret.len() - terminator.len()..ret.len()).unwrap() == terminator
        {
            break;
        }
    }
    Ok(ret)
}
```

Note: The `unwrap`s here will be replaced with proper error handling.
Furthermore, `InsrumentRs` should also provide appropriate `async` helper functions,
more on that below.

In this manner, the transport implementation can be as complex or simple as needed
for communicating with the instrument.
The driver author is responsible for the implementation,
but the boilerplate code for a whole instrument can be generated automatically.

### Interface

Finally, the interface is something that the driver author does not have to supply.
The end user of the driver simply chooses what interface the instrument supports
and then constructs the specific instrument with this interface.

## Errors

An additional benefit of this approach for the driver author is
that `instrumentRs` takes care of all the error handling.
These `InstrumentRsError`s should be publicly re-exported in the driver itself via

```rust
pub use instrumentrs::InstrumentRsError;
```

Note: Error handling is not yet properly implemented in the first test version.
Thus, you see `InstrumentRsError::PlaceholderError`
in some of the code snippets above.

## The end user experience

Let's see how this will look from the end user experience.
The test repo currently contains a driver for my [DigOutBox].
For all intent and purposes, this box is a simple [SCPI]-inspired
instrument that I built some time ago and, most importantly,
stands on my office desk for testing.

The following example shows how to setup and query
the name of the connected [DigOutBox] from the end user's perspective.

```rust
use digoutbox::DigOutBox;

pub fn main() {
    let port = "/dev/ttyACM0";
    let baud = 9600;

    let mut interface = serialport::new(port, baud).open().unwrap();
    interface.set_timeout(Duration::from_secs(3)).unwrap();

    let mut inst = DigOutBox::new(interface);

    println!("{:?}", inst.get_name());
}
```

We first specify the address and baud rate of the instrument
and then use the [serialport] crate to create and open the interface.
We can now simply create a new `DigOutBox` instrument
by providing the interface to the constructor.
No we are all set to interact with the instrument.
We can for example print out
the name of the instrument using the implemented routine.

## What remains to be defined?

### Testing

So far, we have only touched on how instrument drivers should look.
However, once a driver is written,
we also want to make sure that they are properly tested!
Thus, `instrumentRs` must supply a Mock interface
that allows straight-forward testing of a given instrument driver.
This will allow, e.g., for refactoring of code
without having access to the physical instrument.
Once drivers are implemented with tests, this will also help
with testing future developments of `instrumentRs`
as these drivers can serve as a test suite to avoid accidentally breaking functionality.

### Macro and DSL implementation

While I have thought a lot about the design on how it should look,
I have spent almost no time on how the DSL or the macros
will look for automatically generating the core code.
It will be very important to also allow in the DSL to properly document
the various commands that are possible,
as documentation generation is one of the great parts of Rust.

Furthermore, it would be nice to have a procedural macro
that allows deriving the `Parameter` trait implementation
for your new types.

Generally on this topic, I have a lot to learn here.
But what better way to learn than by having an actual project.

### Bound checks and unitfulness

Arguments that are sent to an instrument should
(1) allow for bound checks (ideally in the DSL)
and (2) be unitful.
Bound checks will prevent an end user from sending
illegal values, e.g., by accidentally setting an output channel
to `f64::MAX` volts.

Unitfulness is also very important as many instruments
will only accept certain values as integers or floats and assume certain units.
The end users should not be plagued by having to remember
to send values in the right units.
Instead, they should be given a type that can be created with a unit
that will then be converted to the unit the instrument expects.
The plan here is to use the [measurements] crate,
a lightweight dependency that enables dealing with physical units.
However, this decision is ultimately left to the driver author.
In the end, we want to avoid another "unit-incident"
as was the case for the [Mars Climate Orbiter].

### Async support

Version `v0.1.0` of `instrumentRs` only considers blocking interfaces.
One of the reasons for this is discussed in this [going async issue].
However, [serialport] has a PR with recent and active discussion to
[add async support].
`instrumentRs` should thus be able to generate `async` drivers
without much additional overhead for the developer.

One idea here is that the macro implements two instrument,
one for blocking and one for asynchronous interactions.
These instruments would either take a `<I: Read + Write>`
or an `<IA: AsyncRead + AsyncWrite>` generic interface.
The driver author then will need to add `async` support
by implementing a `TransportAsync` trait (to be defined).
This should not be too much work as the hardest part
of the transport definition is how to assemble the command package.
This assembly can take place outside the `sendcmd` and `query`
functions.
The `sendcmd` and `query` functions then need to be implemented
for blocking and asynchronous interfaces,
however, this implementation can hopefully take full advantage
of helper functions in the `instrumentRs` transport module.

Every instrument driver should ultimately have a
`blocking` and an `async` feature,
where the default is `blocking`.
This will allow the end user to choose what they want
without putting a lot of additional burden on the driver author.

One question that is still very open in my mind
is how to implement the `AsyncRead` and `AsyncWrite` traits
in a runtime agnostic way.
However, the [add async support] PR on [serialport]
might have some hints on how to accomplish this.

### Additional channels for, e.g., math functions

Oscilloscopes for example have channels but can also have additional
virtual channels that can be set up.
For example, a user can display the signal measured on channel 1
and display some math function that is performed on this signal.
In [instrumentkit], e.g., this [MAUI driver],
the math channel is integrated as every other channel but has its own
access structure.
How such a case should be handled in `instrumentRs` remains to be defined.

### Repositories

Currently, the `v0.1.0` [instrumentRs repo]
contains the `instrumentRs v0.1.0` crate
as well as some drivers that I created using it.
Drivers that are currently hosted there
should probably move to their own repo, maybe named
`instrumentRs-drivers`.
This way, driver management and toolbox development
will be properly separated.

If you are curious on where I am with `v0.2.0`,
go to [instrumentRs2 on GitHub].
This is only intended as a sandbox repo and,
once it is in a state that I'm happy with,
will be merged into the `instrumentRs` repo.

### Documentation

`instrumentRs` will be properly documented as a crate.
In addition, a book is planned to discuss details in depth
and give newcomers pointers on how to start writing an instrument.
Over the years I have written many drivers and my approach
by now on how to start a new driver is kind of standardized.
I think such a document might help some people to get into writing
their own Rust-based instrument drivers.

## Next steps

So what's next?
First of course I want to finish the `v0.2.0` implementation
and add proper error handling.
I also want to implement a Mock interface
that allows author drivers to easily set up tests.
This Mock interface will likely be heavily inspired by [instrumentkit],
where the commands that are passed back and forth are tested.
Finally, I also want to play with an `async` implementation
to see how simply it would be to implement both interface types
for device authors.

I also have access to instruments that require more complex transport handling.
These instruments require packing bytes properly,
return acknowledgments for every sent command, etc.
Implementing more drivers by hand and without macros should
give me invaluable hints if this current design idea is feasible or not.
Having access to instruments will also allow lab testing,
i.e., actually connecting the instrument and giving it a spin.
Is the designed interface ergonomic?
What are the friction points?

More blogposts are to come.
These posts are a good way for me to focus my thoughts
by writing the design ideas down.
Furthermore, these posts allow me to hopefully gather some input
from you!

## Comments, questions, ideas?

If you have comments or questions,
please get in touch with me.
You can do so via [instrumentRs2 on GitHub],
[e-mail](mailto:reto@galactic-forensics.space),
[Bluesky],
or [Matrix].
I'm excited to hear your ideas!

[add async support]: https://github.com/serialport/serialport-rs/pull/331
[Bluesky]: https://bsky.app/profile/galactic-forensics.space
[cargo-generate]: https://github.com/cargo-generate/cargo-generate
[device-driver]: https://github.com/diondokter/device-driver
[DigOutBox]: https://github.com/galactic-forensics/digoutbox
[endianness]: https://en.wikipedia.org/wiki/Endianness
[going async issue]: https://github.com/trappitsch/instrumentRs/issues/9
[instrumentkit]: https://github.com/instrumentkit/instrumentkit
[instrumentRs on crates.io]: https://crates.io/crates/instrumentrs
[instrumentRs repo]: https://github.com/trappitsch/instrumentRs
[instrumentRs2 on GitHub]: https://github.com/trappitsch/instrumentrs2
[Mars Climate Orbiter]: https://en.wikipedia.org/wiki/Mars_Climate_Orbiter
[Matrix]: https://matrix.to/@trappits:epfl.ch#/@trappits:epfl.ch
[MAUI driver]: https://github.com/instrumentkit/InstrumentKit/blob/main/src/instruments/teledyne/maui.py
[measurements]: https://docs.rs/measurements/latest/measurements/
[MODBUS]: https://en.wikipedia.org/wiki/Modbus
[orphan rule]: https://doc.rust-lang.org/book/ch10-02-traits.html#implementing-a-trait-on-a-type
[serialport]: https://docs.rs/serialport/latest/serialport/
[transport section]: #transport
[SCPI]: https://en.wikipedia.org/wiki/Standard_Commands_for_Programmable_Instruments
