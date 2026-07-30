+++
title = "InstrumentRs: #[derive(Parameter)]"
date = "2026-07-30"

[taxonomies]
tags=["Rust", "instrumentRs"]
categories = ["Programming", "Science"]

[extra]
repo_view = false
+++

As promised in the [last blog post],
I spent some time developing a derive macro
that allows deriving how new types will be written to
and read from an instrument.
This blog post is about these changes,
how they work,
and how they still need to be improved.

<!--more-->

If you are reading this and have no idea what I'm talking about,
take a look at the [previous blog posts] that discuss [instrumentRs v0.2.0].

## Recent changes

Let's first talk about the changes since the [last blog post].
I first did some housekeeping work:
The project is now properly structured into a workspace
and tests, formatting checks, and clippy
are run in CI.
The restructuring of the project as a workspace
was also important as it allows adding a macros crate,
the topic of this blog post,
that contains the code to help with developing instrument drivers.

## The `InstrumentParameter` trait

Let's first recapitulate briefly since it's been a while.
The idea behind the `InstrumentParameter` trait,
which was before this blog post only called `Parameter` trait,
is the following:
Our instrument drivers needs to be able to send a command,
encoded in some way to bytes, to the instrument.
When the driver receives an answer,
it should also be capable of decoding this answer.
This is where the `InstrumentParameter` trait comes in,
which looks like this:

```rust
pub trait InstrumentParameter<W: Writable>: Sized {
    fn to_writable(&self) -> W;
    fn try_from_writable(val: W) -> Result<Self, InstrumentError>;
}
```

This trait must live in the instrument driver crate due to the [orphan rule]
and will be defined when using the instrument creation macro.
The generic `<W>` type that it takes is a type
that implements the `Writable` trait, which is defined in instrumentRs.
This trait simply defines how instrumentRs turns this parameter into a byte slice,
which is a requirement for sending it to the instrument.

### Example: Setting the display illumination

Let's assume the following scenario:
An instrument has a display
for which we can turn the background illumination on or off.
The following commands in this example control the light:

- `DispLight 1`: Turn the display illumination on.
- `DispLight 0`: Turn the display illumination off.

The instrument itself will have a `set_display_light(...)`
and a `get_display_light(...)` function
in order to set/get the status of the light.
These functions know how to stitch the command `DispLight`
and the parameter `0` or `1` together.
However, we want to pass a more user friendly type
to the set function
instead of making the user pass a `"1"` or a `"0"` to the setter.
Similarly we want to return a useful type when getting the status.
In typical Rust fashion, we can define a new type as following:

```rust
pub enum DisplayLightStatus {
    On,
    Off,
}
```

If we want to turn the display light on,
the `set_display_light(...)` function will have to take
a `DisplayLightStatus` argument
and then stitch the command properly together.
In order to do so, `DisplayLightStatus`
must implement `InstrumentParameter<String>`.
Here we use `String` since we can form all commands
by stitching strings together,
which is likely the default communication way for most instruments.
For `DisplayLightStatus`, the implementation could look somewhat like this:

```rust
impl Parameter<String> for DisplayLightStatus {
    fn to_writable(&self) -> String {
        match self {
            Self::Off => String::from("0"),
            Self::On => String::from("1"),
        }
    }

    fn try_from_writable(val: String) -> Result<Self, InstrumentError> {
        match val.trim() {
            "0" => Ok(Self::Off),
            "1" => Ok(Self::On),
            _ => Err(InstrumentError::BadInstrumentResponseString { msg: val }),
        }
    }
}
```

Here, `to_writable` is used when setting the status,
`try_from_writable` is used when reading the status.

Let us now consider a more complicated scenario:
We have a second command that sets the full display light status,
which for example reasons means the status (on/off) and the brightness.
The command might look like this:
`DispLightFull ST,BRIGHT`.
Here, `ST` would be the status (so `0` or `1`)
and `BRIGHT` the brightness that can be set, for this example,
between low (`LOW`), medium (`MED`) and maximum (`MAX`).
We now define a new brightness type

```rust
pub enum Brightness {
    Low,
    Medium,
    Max,
}
```

and implement `InstrumentParameter` for it.
This implementation is very analogous to the one for `DisplayLightStatus`.

Then we define a new struct:

```rust
pub struct DisplayStatusFull {
  status: DisplayLightStatus,
  brightness: Brightness,
}
```

and implement `InstrumentParameter` for it.
This implementation is going to be more complicated
but could look somewhat like this:

```rust
impl InstrumentParameter<String> for DisplayStatusFull {
    fn to_writable(&self) -> String {
        format!(
            "{},{}",
            self.status.to_writable(),
            self.brightness.to_writable()
        )
    }

    fn try_from_writable(val: String) -> Result<Self, InstrumentError> {
        let vsp: Vec<&str> = val.trim().split(',').collect();

        if vsp.len() != 2 {
            return Err(
                InstrumentError::BadInstrumentResponseString { msg: val }
            );
        }

        let status = DisplayLightStatus::try_from_writable(
            vsp[0].to_string()
        )?;
        let brightness = Brightness::try_from_writable(
            vsp[1].to_string()
        )?;

        Ok(Self { status, brightness })
    }
}
```

This would allow us creating instrument functions
that set/get the full display status.
However, for the `DisplayStatusFull` struct
we ended up having to hand-write a parser for the returned command.

## Deriving `impl InstrumentParameter`

As above example shows, implementing `InstrumentParameter`
for all of our new types requires a lot of boilerplate code.
Since instrumentRs should make writing instrument drivers simpler
and this boilerplate code is well defined from the beginning,
a derive macro was developed that allows us deriving
this implementation.
The [instrumentRs v0.2.0] repository now contains a
proc-macro library crate.
The main crate then reexports the `Parameter` derive macro.
Currently, the simple cases shown above can easily be derived.
Further extensions to handle more complicated cases are planned;
more on this below.

### Deriving for enums

Let us derive `InstrumentParameter` for `DisplayStatus`.
This would currently look as following:

```rust
use instrumentrs::Parameter;
use crate::InstrumentParameter;

#[derive(Debug, Parameter)]
#[cmd("{}")]
pub enum DisplayLightStatus {
    #[param("1")]
    On,
    #[param("0")]
    Off,
}
```

The way that we specify commands (`cmd`) and parameters (`param`)
is inspired by [thiserror].
Here we simply say that the command will take one parameter.
To do so, we just pass a string that contains one placeholder (curly brackets);
nothing fancy about that.
The `param` argument on each of the variants then defines
how this variant is transferred into a `String`.
The macro takes care of the rest and implements
`InstrumentParameter` for this enum.
The trait has, of course, to be in scope.

### Deriving for structs

Implementing `InstrumentParameter` manually for the `DisplayStatusFull` struct
was quite a bit more involved but fear not!
The `Parameter` derive macro allows us to write the following:

```rust
#[derive(Debug, Parameter)]
#[cmd("{},{}")]
pub struct DisplayStatusFull {
    status: DisplayLightStatus,
    brightness: Brightness,
}
```

That is a lot shorter than hand-rolling a parser for every new type.
All types that are stored in the struct must of course also implement
`InstrumentParameter` for this to work.
If one of these implementations is missing,
the compiler will however give us a decent error message
pointing us to the correct issue.

To make this more general,
we could also define the following:

```rust
#[derive(Debug, Parameter)]
#[cmd("{1},{0}")]
pub struct DisplayStatusFull {
    brightness: Brightness,
    status: DisplayLightStatus,
}
```

Here we basically swapped the fields in the struct.
Since the instrument however still needs the command in the same form as before,
we used positional placeholders in the `cmd` definition.
If we use unordered placeholders as above,
the derive macro will simply assume that they are in order of the fields.
Since positional placeholders require us to sort
the results that the parser outputs,
using them will cost you a sliver of performance.
While available as an option, positional placeholders in the derive macro
are thus not recommended.

### What is missing

While the derive macro is helpful,
certain must-have features are still missing.

For example, let us assume we have the following scenario:

```rust
#[derive(Debug, Parameter)]
#[cmd("{}")]
pub enum Inner {
    #[param("1")]
    Inner1,
    #[param("2")]
    Inner2,
}

pub enum Outer {
    Outer1(Inner),
    Outer2(Inner),
}
```

Currently, the derive macro cannot be applied to derive
`InstrumentParameter` for outer.
This should however be possible in a similar way as for
the `DisplayStatusFull` struct above
since `Inner` implements `InstrumentParameter`.

For the brightness example, you might have considered the fact
that brightness could be defined as a number between 0 and 100.
This would allow us to create a new type

```rust
pub struct Brightness {
    value: u8,
}
```

and create a `try_new` method
that would allow for range checking the brightness value.
Either way, there is currently no good way to derive `InstrumentParameter`
for this `Brightness` struct.

Finally, the derive macro currently only works on strings
in which the parameters are separated by delimiters.
There should also be a possibility to define
how many characters long a parameter is going to be.
For example, if an instrument would return two numbers
that each are ten characters long but are otherwise not separated,
the derive macro should make a parser for this scenario available.

### Non-goals

These missing features represent some cases that are going to be very common.
However, there will always be special cases that require
manual implementation of the `InstrumentParameter` trait.
The goal of the derive macro is not to take care of all cases,
but to simplify the most common ones!

The cases that are common will depend on
instrument driver author reports.
If you feel that the derive macro should contain an additional case,
please raise an issue and we can discuss it.
However, completeness for deriving `InstrumentParameter` in all special cases
is a non-goal for the derive macro.

## Changes in the `Channel` structure

After implementing the derive macro I immediately applied it
to the [Lakeshore 336] driver.
You can find the updated driver on the
[instrumentRs-driver lakeshore336 branch].
This is of course when I noticed some of the missing parts above.
However, another issue showed up:

The `Channel` of the [Lakeshore 336] driver,
which can distinguish between inputs (sensor) and outputs (e.g., heater),
also implemented `InstrumentParameter` until now.
However, this is actually not ideal
and could for other instruments even be incorrect.

The `Channel` structure should solely be used to distinguish internally
between different types of channels.
However, it should never be requested from
or returned to the user.
Since it is never returned to the user,
`try_from_writable` does not need to be implemented.
What we really need to send to the instrument
is the inner part that is held by `Channel`.

To solve this, I removed the `InstrumentParameter` implementation on `Channel`.
Instead, I added an `inner_to_writable` function,
which can be used when implementing the transport.
For the [Lakeshore 336] this now looks as following:

```rust
pub enum Channel {
    In(Input),
    Out(Output),
}

impl Channel {
    pub fn inner_to_writable(&self) -> String {
        match self {
            Channel::In(i) => i.to_writable(),
            Channel::Out(o) => o.to_writable(),
        }
    }
}
```

For clarity, all derives, and feature specifications were omitted.

Another reason for not implementing `InstrumentParameter`,
the reason why it might be in fact incorrect for other instruments,
is that the mapping might not be unique.
Sure, in the case of the [Lakeshore 336] input channels map to `A`-`D`
and output channels to `1`-`4`.
However, these could easily both be mapped to the same values
in another instrument
and the real distinction would take place in the get/set functions.
Thus, `try_from_writable` could not be implementable in
a unique way.

Ultimately, the `Channel` structure will be implemented
by the instrument defining macro.
However, the instrument driver author will need to be aware of the fact
that `inner_to_writable` must be used when writing the transport.
This of course is an issue for adequate documentation and examples.

## Outlook: A parser for instrument commands

One limiting part of the current implementation is,
as mentioned above,
that parsing requires delimiters between the parameters.

One solution around this is to have different parsers for different scenarios.
These parses could live in instrumentRs itself.
The derive macro then could directly hook into these parsers
for the `try_from_writable` implementations.

I am currently playing with this idea in the
[command parser branch].
The parsers are not intended
to be considered as a part of the public API.
The macro itself will need to check that the provided command structure is valid.
Then, any issues while parsing would automatically require the return of an
`InstrumentError::BadInstrumentResponseString` error.

Why write my own parser?
The parser will be required at run time,
i.e., when the driver user is working with the instrument.
Since these parsers (so far at least)
seem fairly straight forward to implement,
I prefer to keep them internal for the moment
without requiring another dependency.
This might however change.

## Next steps

The next steps stay pretty much the same as before:

- The derive macro needs additional capabilities.
- More instrument drivers will be written to test the framework.

These two tasks will go hand-in-hand:
Writing more instrument drivers will help defining
what the derive macro should look like,
which will in return make writing more instrument drivers easier.
The current state of the macro helps already
by reducing a lot of boilerplate.
The next update however will likely not be until the end of August
as some other tasks take priority over the next couple of weeks.

## Comments, questions, ideas?

As always,
if you have comments or questions,
please get in touch with me.
You can do so via [instrumentRs v0.2.0] on GitHub,
[e-mail](mailto:reto@galactic-forensics.space),
[Bluesky],
or [Matrix].
I'm excited to hear your ideas!

[Bluesky]: https://bsky.app/profile/galactic-forensics.space
[command parser branch]: https://github.com/trappitsch/instrumentRs2/tree/command_parser
[instrumentRs v0.2.0]: https://github.com/trappitsch/instrumentRs2
[instrumentRs-driver lakeshore336 branch]: https://github.com/trappitsch/instrumentRs-drivers/tree/lakeshore336
[last blog post]: https://blog.galactic-forensics.space/blog/09-instrumentrs2-lakeshore336/
[Lakeshore 336]: https://blog.galactic-forensics.space/blog/09-instrumentrs2-lakeshore336/
[Matrix]: https://matrix.to/@trappits:epfl.ch#/@trappits:epfl.ch
[orphan rule]: https://doc.rust-lang.org/book/ch10-02-traits.html#implementing-a-trait-on-a-type
[previous blog posts]: https://blog.galactic-forensics.space/tags/instrumentrs/
[thiserror]: https://docs.rs/thiserror/latest/thiserror/
