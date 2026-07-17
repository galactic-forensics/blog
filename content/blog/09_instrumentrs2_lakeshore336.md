+++
title = "Lakeshore 336 Temperature Controller Driver with instrumentRs v0.2.0"
date = "2026-07-17"

[taxonomies]
tags=["Rust", "instrumentRs"]
categories = ["Programming", "Science"]

[extra]
repo_view = false
+++

It has been a few weeks since my last post.
Since then I have focused on the async interface,
a note on this at the end of this blog post,
and on integrating a new instrument
using [instrumentRs v0.2.0].
Here are my findings on writing a more complex driver.

<!--more-->

If you are reading this and have no idea what I'm talking about,
take a look at the [previous blog posts] that discuss [instrumentRs v0.2.0].

## The Lakeshore 336 temperature controller

The [Lakeshore 336] cryogenic temperature controller
is an instrument that allows the user to connect up to four thermocouples
and read their temperatures
and connect up to four outputs to regulate the read temperature,
e.g., by using heaters.
I needed a new driver for this instrument for our
[cryostorage project].
A blog post on this project will hopefully come soon.
As a side note: The [cryostorage project]
was the original reason I started with instrumentRs.

I created a new repository for instrumentRs specific drivers called
[instrumentRs-drivers].
This repository is coupled to the `v0.2.0` development.
The goal is to move all previous drivers over here after `v0.2.0`
is released and has moved into the [main `instrumentRs` repo]
and after existing drivers have been updated to `v0.2.0`.

If you want to see the whole Lakeshore 336 driver,
you can currently find it in the [`lakeshore336` branch]
of the [instrumentRs-drivers] repo.
If you read this later
it might have already been merged into main.
Note that documentation is still lacking
in the current draft.
Furthermore, not all functions of the instrument are implemented.
However, enough functionality is there
to run this driver for the [cryostorage project]
and to learn how my original idea for instrumentRs
needs to be extended.

## Extensions to [instrumentRs v0.2.0]

This section discusses several extensions to what [instrumentRs v0.2.0]
will need to provide to a driver author
and what changes are necessary to smoothly implement the Lakeshore 336 driver
into the current framework.
The discussion focuses on the major changes.
Some minor details have also been adopted:
You can find a full diff in [this instrumentRs PR] (now merged).

### Error handling

First off, the errors that instrumentRs provides were renamed
to `InstrumentError`.
This way, a driver can simply publicly reexport this error type
and end users can see a reasonable error name.
Of course, this public reexport could be translated into more useful name,
e.g., `Lakeshore336Error`, however,
for now we keep it at `InstrumentError`.

Errors in [instrumentRs v0.2.0] have also been extended with the following:

- **`ParseIntError` and `ParseFloatError`**:
  When parsing returned values from an instrument
  to an integer or float, this error from `std::num` can now also be returned
  as an `InstrumentError`.
  This was crucial to get the Lakeshore 336 instrument working.
- **Out-of-range errors**: Several out-of-range errors were implemented.
  These can be used when range checking is done
  while setting a specific property of a driver.
  All the out-of-range errors take the value the user entered,
  as well as the minimum and maximum values that are allowed
  for the specific range,
  and return an adequate error message.
  The following out-of-range error types
  are now available to a driver author:
  - `UnitfulValueOutOfRange`: This error takes the numeric value
    of the unitful quantity that is out of range as an `f64`.
    Additionally, the unit is provided as a string.
    This allows displaying the unitful quantity and
    accepted range
    for displaying an out-of-range error.
    This is specifically kept generic
    as instrumentRs should not dictate a unit-handling crate.
  - `FloatValueOutOfRange`: An `f64` is out of range.
  - `IIntOutOfRange`: A signed integer is out of range.
    This takes values as types of `isize`.
  - `UIntOutOfRange`: An unsigned integer is out of range.
    This takes values as types of `usize`.
- **`Other`**: Any other error types that a driver author might want to return.
  This error takes a `String` to display a driver-author-chosen
  error message.
  The `Other` error should generally be avoided as `InstrumentRs` should contain
  specific errors for a broad range of cases.
  However, this could still be useful in certain cases
  and is inspired by [`std::io::ErrorKind::Other`].
  If you find yourself writing an instrument driver
  and need to reach often to the `Other` error type,
  you should raise an issue and report this,
  as likely a more specific error is missing in `InstrumentError`.

In general, `InstrumentError` is labeled with `#[non_exhaustive]`,
which will allow adding additional errors as they come along.
While Rust crates typically implement their own error types,
[instrumentRs v0.2.0] will differ here a bit from the norm
in order to make it simple to write instrument drivers.
This is especially useful as most of the instrument driver code
will be macro generated.

### Handling multiple types of `Channels`

The [DigOutBox] example that we used in [previous blog posts]
had one major down side that was recognized immediately
when starting to write the Lakeshore 336 driver:
General instruments can have more than one type of channel.
In this instrument we have `Input` channels where sensors are connected
and `Output` channels where, e.g., a heater might be hooked up.
The previous idea for [instrumentRs v0.2.0] only allowed
for one type of channel.

To alleviate this issue, the `Transport` trait,
which is part of the [transport module], was generalized
as following (highlighted parts show the changes):

```rust,hl_lines=2 7 15-16
pub trait Transport<W: Writable, WR: Writable> {
    type Channel;
    /// The send command that you need to implement.
    fn sendcmd(
        &mut self,
        cmd: W,
        idx: Option<Self::Channel>,
        args: Option<&[W]>,
    ) -> Result<(), InstrumentError>;

    /// The query command that you need to implement.
    fn query(
        &mut self,
        cmd: W,
        idx: Option<Self::Channel>,
        args: Option<&[W]>,
    ) -> Result<WR, InstrumentError>;
}
```

Here, channel identifier now takes a `Channel` type that must be defined
in the instrument driver itself.

For the Lakeshore 336 driver,
a `Channel` type was defined as following:

```rust
pub enum Channel {
    In(Input),
    Out(Output),
}
```

The specific input and output channels look as following:

```rust
pub enum Input {
    InA,
    InB,
    InC,
    InD,
}

pub enum Output {
    Out1,
    Out2,
    Out3,
    Out4,
}
```

This looks, at first glance, more complicated
however, most of the complexity can be abstracted away for the user.
Furthermore, this additional complexity has several advantages.
First off, we can now easily distinguish in the driver specific transport implementation
which kind of channel we are dealing with.
The user would continue getting an input or output channel directly,
from an instrument `inst` as:

```rust
let in_a = inst.channel(Input::InA);
let out_1 = inst.output(Output::Out1);
```

The beauty for the user now is that this does not return a `Result` anymore.
Since we have fixed the possible input and output channels,
we know that the channel cannot be out of range.
Furthermore, the channels can be named identical or similar
to what they are named on the actual instrument.
The Lakeshore 336 temperature controller for examples uses
channels A through D for temperature input sensors
and channels 1 through 4 for output channels.

The implementation on how to get a channel in the Lakeshore 336 driver
looks now like this:

```rust
impl<I: Read + Write> Lakeshore336<I> {
    pub fn channel(&mut self, idx: Input) -> Lakeshore336Input<'_, I> {
        Lakeshore336Input::new(idx, self)
    }

    pub fn output(&mut self, idx: Output) -> Lakeshore336Output<'_, I> {
        Lakeshore336Output::new(idx, self)
    }
}
```

The private new methods for creating an input or output
also take their respective `idx` types,
however, they store internally the `Channel` type:

```rust
pub struct Lakeshore336Input<'a, I: Read + Write> {
    device: &'a mut Lakeshore336<I>,
    idx: Channel,
}
```

The channel identifier can then be used in the `Transport` trait
to create the required command packages that will be sent to the instrument.
Here, the driver author can now match on the `Channel` type itself
to determine if an input or output channel is required.

Overall, the instrument driver
has become slightly more complicated for the driver author,
however, with significant benefits downstream for the user.
Furthermore, channel generation
should all ultimately be part of the macro-generated driver.
Thus, this extension to a generic `Channel` type
should carry minimal overhead for the driver author.

### Query now also takes arguments

The `Transport` trait, as shown above, was further extended
such that the `query` function now also take channels
and arguments (both are optional).
Thus, the signature of `sendcmd` and `query` in the trait are now identical.
This change was required as the Lakeshore 336, e.g.,
has query functions where user supplied arguments must be provided.

### Wrapping a command in a `char`

To set the name of a channel in the Lakeshore 336,
a given user string must be wrapped in `"`
before sending it to the instrument.

```rust
/// Set the name of a channel.
pub fn set_channel_name(&mut self, channel_name: &str) -> Result<(), InstrumentError> {
    let wrapped = format!("\"{}\"", channel_name);
    self.device
        .sendcmd("INNAME", Some(self.idx), Some(&[&wrapped]))
}
```

Above example shows the manual implementation.
We first create a `wrapped` variable for which we simply wrap the command
in `"`.
This is then passed on to the `sendcmd` function.

This feature should be provided in the driver-creation macro
such that the driver author can specify a wrapping character if necessary.

## Towards a `cargo-generate` template

As the crate structure for an instrument driver keeps on getting more complicated,
providing a [cargo-generate] template becomes interesting and important.
Such a template should:

- Add the specific instrumentRs dependencies and dev dependencies.
- Provide a simple structure on what must be set up with a simple instrument.
- Generate an appropriate `Cargo.toml`
  with the specific features listed that are recommended for a new driver.
- Set up the testing infrastructure using the mock interface.

While a template is going to be fairly opinionated,
it should significantly lower the activation energy to write an instrument driver.

### Features

For the [cryostorage project] I require that instrument configurations
can be de-/serialized.
This allows loading/storing instrument configurations
to a file.
De-/serialization requires [`serde`] as a dependency.
This option was put behind a "serde" feature
as not every user of the driver will need this functionality.

InstrumentRs should provide such feature gated additions
by default when generating a new driver structure from a [cargo-generate] template.

Furthermore, sync and async versions should also be feature-gated
in a default way by such a template to make drivers that use instrumentRs
have a common feel to the end user.
This will also require the implementation of a default feature.
For example, a user adding a dependency on a new driver should,
when default features are activated,
get the blocking version of the driver.

### Mock interface testing

A template will also set up example tests
that use the mock interface.
This again should make it straightforward for the driver author
to start writing tests for their instrument driver.

## Next steps

The next steps have slightly changed from the previous discussion,
however, mostly the order is different.
After implementing an instrument drivers,
the following next steps will be taken on:

### Housekeeping

First off, there is some necessary housekeeping work to do.
There are several features by now and various tests
that only run with specific features.
Thus, a CI workflow probably a [cargo-nextest] configuration will be set up.
This should of course also include formatting checks,
build checks against nightly, etc.;
the standard bunch of CI runs.

In preparation for the first macro to land,
the whole project should also move into a workspace of its own.
This will allow adding the upcoming proc macros
in their respective crates
while keeping the dependencies in sync.

### Derive macro for `Parameter` trait implementations

One thing I noticed while writing the Lakeshore 336 driver is
that I had to repeat myself very often when implementing
the driver's own [`Parameter` trait] for new instrument types.
Implementing especially `Parameter<String>` for newtypes
should be possible with a derive macro,
such that we can simply derive this implementation.
This might in fact look similar to how [`thiserror`]
derives `Error` for a given `enum`.

### Implement more drivers

Having learned a lot from writing a Lakeshore 336 driver with the framework,
more driver work is required to solidify the base.
At least one instrument that requires command acknowledgment
and an instrument that requires more complicated command packing
should be implemented to find currently existing weak points
in the [instrumentRs v0.2.0] idea.

### Async instrument drivers

As you might have noticed,
previous posts mentioned async interfaces in the next steps.
I have in fact a functional async version available
if you want to have a peek.
It is available on the
[instrumentRs2 async_support branch].
This branch and its examples are currently fully functional.

So why is this post not talking about it?
In short: Because I don't yet fully understand if it is correct
and why.
When I started writing the async part, I had to read
up on how to return `async fn` in traits[^1]
and on `Pin` and `Unpin`[^2].
The development also made ample use of clippy without always fully understanding
(as of yet) why I had to, e.g., add `Unpin` where it's now at.
As always, I'm letting this problem simmer for a bit in my brain,
which should help me digest and understand it over time.
With some additional reading I hope to be able to explain what is going on here.
Full disclosure: I am fairly new to the async world
and thus, this will take a bit more time.

If you are interested in this feature and would like to discuss things through,
I would be more than happy to do so!
Learning something new and understanding concepts better
is one of my main reasons for writing instrumentRs.

So until I further understand my own async implementation,
I will restrain from posting about it.
I will of course keep on hacking at it and feeling out
what it can and cannot do.

## Comments, questions, ideas?

If you have comments or questions,
please get in touch with me.
You can do so via [instrumentRs2 on GitHub],
[e-mail](mailto:reto@galactic-forensics.space),
[Bluesky],
or [Matrix].
I'm excited to hear your ideas!

[^1]: [Announcing `async fn` and return-position `impl Trait` in traits]
[^2]: [Module pin]

[Announcing `async fn` and return-position `impl Trait` in traits]: https://blog.rust-lang.org/2023/12/21/async-fn-rpit-in-traits/
[Bluesky]: https://bsky.app/profile/galactic-forensics.space
[cargo-generate]: https://github.com/cargo-generate/cargo-generate
[cargo-nextest]: https://nexte.st/
[cryostorage project]: https://github.com/trappitsch/cryostorage
[this instrumentRs PR]: https://github.com/trappitsch/instrumentRs2/pull/1
[DigOutBox]: https://github.com/galactic-forensics/digoutbox
[instrumentRs-drivers]: https://github.com/trappitsch/instrumentRs-drivers
[instrumentRs v0.2.0]: https://github.com/trappitsch/instrumentRs2
[instrumentRs2 on GitHub]: https://github.com/trappitsch/instrumentrs2
[instrumentRs2 async_support branch]: https://github.com/trappitsch/instrumentRs2/tree/async_support
[Lakeshore 336]: https://www.lakeshore.com/products/categories/overview/temperature-products/cryogenic-temperature-controllers/model-336-cryogenic-temperature-controller
[`lakeshore336` branch]: https://github.com/trappitsch/instrumentRs-drivers/tree/lakeshore336
[main `instrumentRs` repo]: https://github.com/trappitsch/instrumentRs
[Matrix]: https://matrix.to/@trappits:epfl.ch#/@trappits:epfl.ch
[Module pin]: https://doc.rust-lang.org/std/pin/index.html
[`Parameter` trait]: https://blog.galactic-forensics.space/blog/07-instrumentrs2/#the-instrument-instrumentchannel-structures-and-the-parameter-trait
[previous blog posts]: https://blog.galactic-forensics.space/tags/instrumentrs/
[`serde`]: https://docs.rs/serde/latest/serde/
[`std::io::ErrorKind::other`]: https://doc.rust-lang.org/std/io/enum.ErrorKind.html
[`thiserror`]: https://docs.rs/thiserror/latest/thiserror/
[transport module]: /blog/07-instrumentrs2/#transport
