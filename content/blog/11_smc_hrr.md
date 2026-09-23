+++
title = "Controlling an SMC HRR chiller via MODBUS"
date = "2026-09-23"

[taxonomies]
tags=["Rust", "instrumentRs"]
categories = ["Programming", "Science"]

[extra]
mathjax = true
repo_view = false
+++

It has been a few more weeks than anticipated
in the [last blog post],
but here is finally a new instrumentRs post.
In the past 1.5 months I was working on a new driver
for [SMC HRR] chillers
to test out the [instrumentRs v0.2.0] framework idea.
This post presents the findings of this work.

I also spent some time
developing an embedded driver for the
Analog Devices [LTC2686] eight-channel programmable digital-to-analog converter
using [device-driver].
The low-level driver is mostly finished,
the high-level driver still needs to be defined and written.
If you are interested, here's the
[repo for the ongoing LTC2686 work].
Why spend time writing embedded drivers?
Aside from it being fun and necessary for other projects,
[device-driver] is a very interesting and well designed framework
to write embedded drivers.
It is also one of the big inspirations behind the redesign
of [instrumentRs v0.2.0].
So I felt it necessary to dig into the recent
[device-driver] `v2` release and see how it is to write a driver with it.
Overall, [device-driver] is fantastic to write low-level drivers
and I still feel very strongly that it should
(and hopefully does) inspire [instrumentRs v0.2.0].

Before we go into the weeds:
If you are reading this and have no idea what I'm talking about,
take a look at the [previous blog posts] that discuss [instrumentRs v0.2.0].

## Recent changes

A few changes have been made to the [instrumentRs v0.2.0] repo
to extend capabilities for working with [SMC HRR] chillers:

- A `SilentInterval` was added that, if necessary, blocks the thread
  for a given amount of time to ensure that commands are sent
  with proper quiet time in between.
- A helper to read `n` bytes from the interface:
  Protocols like, e.g., [MODBUS] communicate via predefined command/package sizes.
  Thus, we do not have an "end-of-command" character
  but know in advance how many bytes we need to read.
  The new helper is useful for reading a certain number of bytes and return them.
- Two new error types were added:
  - A checksum error.
  - A negative response error to be returned when an instrument received
    a well-formed command but cannot apply it for some reason
    and thus sends back an invalid command/negative answer response.

## A driver for the SMC HRR chiller line

The [SMC HRR] chillers are rack-mountable water-water or water-air chillers.
They can be ordered with an RS-232C or RS-485 interface
that allows controlling the chiller from a computer.
The [communication functions operation manual] describes
the communication protocol;
see specifically chapter 3 on serial communication.

The chiller can be configured to communicate via two different protocols:
ASCII character string and binary data [MODBUS] RTU.
Here we chose [MODBUS] RTU for two reasons:

- Less bytes need to be sent per command
  and the data packages are fairly simple to assemble.
- Test the `u8`/byte mode of [instrumentRs v0.2.0].

The SMC HRR chiller driver can be found on the
[smc_hrr branch] in the [instrumentRs-drivers] repo.

### Communication protocol

The chiller can be set up for either RS-232C or RS-485 wiring.
Furthermore, the communication speed is either 9600 bps or 19200 bps.
As [instrumentRs v0.2.0] is generic over the interface,
the driver author does not need to worry about these settings
as the end user of the driver is in charge of providing an adequate interface.
However, the driver-author can of course make it simpler for the end user
to create such an interface (see below).

Each request message (write or read request)
will be followed by an answer from the chiller.
A write request will return the same message as was sent
as an acknowledgment.
A read request will return a read package.
Both requests can however also return a negative answer when:

- An unspecified function code was used.
- A register address out of range was specified.
- The data field was not normal.

Finally, if the slave address[^1] or the CRC-16 checksum are invalid,
the chiller will not respond at all.
This will result in an interface timeout error.

### Message framing

Each MODBUS RTU message frame to communicate with the chiller
must consist of the following:

| Start | Slave Address | Function | Data | Checksum (CRC) | End |
| ----- | ------------- | -------- | ---- | -------------- | --- |
| Silent interval | 1 byte | 1 byte | `n` bytes | 2 bytes | Silent interval |

The frame starts with a silent interval
that must be at least 3.5 characters long.
The message frame then expects the one byte slave address of the device as
up to 32 chiller can be connected to the same communication line.

Next follows the one byte long function code,
which must be one of the following:

- `0x04`: Read multiple registers.
- `0x06`: Write registers.
- `0x10`: Write multiple registers.
- `0x17`: Read/write multiple registers.

For this driver, we only use the first two function codes which,
simplifies the driver.
However, we discuss a potential outlook to make [instrumentRs v0.2.0]
more generic to handle multiple function codes or similar/other instrument peculiarities.

The function code is followed by `n` bytes of data
that must be packed big-endian.
The length of the data package depends on the command that is sent.

Then, a CRC-16 MODBUS RTU checksum (2 bytes)
needs to be appended to the message (packed little-endian)
to complete the message. The chiller will check this CRC-16
to verify that the command was transferred properly.

The message frame ends with another silent interval
that must be at least 3.5 characters long.

### Providing a serialport interface

As driver authors, we can choose to make life for the end user pleasant
by providing a default interface that contains reasonable defaults.
The end user can of course still choose to configure the interface
to their own liking.

For the SMC HRR driver I decided to add a "serialport" feature to the driver.
When activated, the [serialport] crate is added as a dependency
and the driver provides a `new_serialport_interface` function,
which is defined as:

```rust
pub fn new_serial_interface(path: &str, baud: BaudRate) -> SerialPortBuilder {
    serialport::new(path, baud.into())
        .data_bits(DataBits::Eight)
        .stop_bits(StopBits::One)
        .parity(Parity::Even)
        .timeout(Duration::from_secs(3))
}
```

This interface now holds the standard configuration for [SMC HRR] chillers,
i.e., the number of data bits, the stop bit, parity,
and a reasonable timeout in case of no response.
Two arguments are needed from the end user: the `path` of the port,
e.g., `/dev/ttyACM0` or `COM7`, and the configured baud rate.
As the chiller by itself only allows for two different baud rates,
`BaudRate` was implemented as a new type as:

```rust
pub enum BaudRate {
    /// 9600 bps.
    Bps9600,
    /// 19200 bps.
    Bps19200,
}
```

This will provide the user, if wanted, with a properly configured
[serialport] interface, which is likely the most common way of talking to the chiller.
Even if the user wants to specify another timeout,
they can achieve this by modifying the `SerialPortBuilder` after it gets returned.

### The silent interval

In order for our messages to be properly framed,
we need to ensure that the silent intervals of 3.5 characters
at the beginning and end of a message
are upheld.
In [MODBUS] RTU mode, each character is 8 bits long and has an additional stop bit,
which totals to 9 bits.
As we do not know how the user will configure the interface,
let us simply assume the slower baud rate of 9600 bps.[^2]
We can calculate the time it takes to send one character as

\begin{equation}
t_\mathrm{char} = \frac{(8 + 1) \\ \mathrm{ bits}}{9600 \\ \mathrm{ bits/s}} = 937.5 \\ \mathrm{ µs}.
\end{equation}

To ensure the proper silent intervals between write commands,
we have to make sure that at the line stays quiet for at least seven characters,
i.e., 3.5 characters at the start and 3.5 characters at the end.
Thus, we want a total silent interval of at least $6562.5 \\ \mathrm{µs}$.

To facilitate such silent intervals,
a new `SilentInterval` has been added to the
`instrumentrs::transport` module.
It looks as following (comments removed for conciseness):

```rust
use std::time::{Duration, Instant};

#[derive(Clone, Debug, PartialEq, Eq, PartialOrd, Ord)]
pub struct SilentInterval {
    duration: Duration,
    last_write: Instant,
}

impl SilentInterval {
    pub fn new(duration: Duration) -> Self {
        Self {
            duration,
            last_write: Instant::now(),
        }
    }

    pub fn block(&mut self) {
        let dt = Instant::now() - self.last_write;

        if dt < self.duration {
            std::thread::sleep(self.duration - dt);
        }

        self.last_write = Instant::now();
    }
}
```

Here, [`std::time::Instant`] is ideal as it is guaranteed monotonic
and is, per its documentation, best used in combination with [`std::time::Duration`].

The chiller driver itself was then extended to hold the following fields:

```rust
pub struct SmcHrr<I: Read + Write> {
    pub(crate) interface: I,
    pub slave_address: SlaveAddress,
    pub(crate) silent_interval: SilentInterval,
}
```

The silent interval, if required, should ultimately be an option of the macro
that generates the instrument.
When a new instrument is created
the duration of the silent interval is configured in the `new` function
(and is constant/driver defined)
and `Instant::now()` is used for setting the instant of the "last" write command.

To ensure that silent intervals are always upheld
we can now add the following function to write packages to the chiller:

```rust
fn write_package<I: Write>(
    interface: &mut I,
    silent_int: &mut SilentInterval,
    pkg: &[u8],
) -> Result<(), InstrumentError> {
    silent_int.block();

    interface.write_all(pkg)?;
    interface.flush()?;
    Ok(())
}
```

From within the instrument, the interface and the silent interval can
be passed to this write function as mutable references along with the data package.
The block function will always make sure
that we fulfill the required amount of silence between commands.
If enough time has passed by itself, the block will update
the silent interval with the last time a command was written
and not block the thread at all.

### Command metadata

As a quick reminder,
the transport trait that a driver author needs to implement for an instrument
looks currently like this:

```rust
pub trait Transport<W: Writable, WR: Writable> {
    type Channel;

    fn sendcmd(
        &mut self,
        cmd: W,
        idx: Option<Self::Channel>,
        args: Option<&[W]>,
    ) -> Result<(), InstrumentError>;

    fn query(
        &mut self,
        cmd: W,
        idx: Option<Self::Channel>,
        args: Option<&[W]>,
    ) -> Result<WR, InstrumentError>;
}
```

Each `sendcmd` or `query` call takes commands,
an optional channel, and optional arguments (i.e., the data).
As shown above, the chiller has in fact four possible ways to talk to it,
which is represented in the function code.
For this driver however, we only used two of these four function codes:
the ones to read and write registers.
Thus, the function codes could directly be implemented in
`sendcmd` (write registers) and `query` (read registers).
However, there will be other instruments that require more command specific handling.

Technically, `args` is a general enough variable name
such that it could possibly hold other arguments than the data
we want to send to the instrument.
But such an approach is not clean.
My proposition is to do the following:

- Rename `args` to `data` to make it clear that this is the data we send.
- Add another `type Metadata` to the transport structure
  that can serve to provide instrument/command specific metadata.

For instruments that do not need metadata, this type could default
to a "zero-type" that would be provided by `instrumentRs`.
However, for instruments that do need it,
the user could specify their own metadata type
and provide them via the driver generation macro to the transport trait functions.
Such a metadata type would add a lot of flexibility for developing instrument drivers.

### Complex instrument commands

In order to control the chiller,
the first thing an end user needs to do is to set
the chiller's remote mode to "serial".
However, if this is done with a running chiller
the chiller is turned off
as the control mode bit and running/stopped bit
are held in the same register.
Such quirks come up frequently when developing instrument drivers;
most instrument have some particularities.

Here, a special function was required
that first checks the status of the chiller
and then adequately sends the correct command to keep the current
running/stopped state unchanged.

```rust
pub fn set_serial_remote_instructions(
        &mut self,
        value: SerialRemoteInstruction,
    ) -> Result<(), InstrumentError> {
        let mut args = value.to_writable();

        let st = self.get_status()?;
        if st.is_running() {
            args[1] += 0x01;
        }

        self.sendcmd(&[0x00, 0x0C], None, Some(&[&args]))
    }
```

First, we get the current status
via an adequate command that is already implemented
and which we will be able to generate
with the driver generation macro.
If the chiller is running, the necessary bits
are set on the arguments that will be sent along
to the write routine.
Otherwise, these bits are left at zero, i.e., the chiller is
already stopped and should stay stopped.

Such special cases cannot be reasonably handled by
a driver generation macro.
However, since multiple `impl` blocks are allowed,
instrument quirks like these can always be managed manually.
Automatically handling complex instrument commands is thus a non-goal
for [instrumentRs v0.2.0].

### Tests

Chiller driver tests were written using instrumentRs' `MockInterface`.
Tests were written after lab testing, i.e.,
after making sure that what was expected from the chiller also happened on the hardware.
This is especially important as many manuals contain errors.

Test writing worked very well and
the tests now assert the end user interface,
which is highly useful as we want to ensure the driver even after a refactor.
While I previously used [rstest] for testing multiple cases,
I gave [proptest] a spin for this driver and have to say that I really like it.
[proptest] requires much less boilerplate
and the tests looks cleaner and more understandable.
As an example, here is the test for verifying
that the correct slave address gets sent:

```rust
proptest! {
    #[test]
    fn get_discharge_pressure_with_various_base_addresses(addr in 1_u8..=32) {

        let mut exp_writes = [vec![addr, 0x04, 0x00, 0x02, 0x00, 0x01]];
        extend_with_crc(&mut exp_writes);

        let mut exp_reads = [
            vec![0x01, 0x04, 0x02, 0x00, 0xAA], // answer chiller is on
        ];
        extend_with_crc(&mut exp_reads);

        let mut inst = smock!(SmcHrr, exp_reads, exp_writes);

        let slave_address = SlaveAddress::try_from_writable(format!("{addr:02}")).unwrap();
        inst.set_slave_address(slave_address);

        // ensure command sending is done with correct slave address
        let val: Pressure = u!(inst.get_discharge_pressure()).into();
        assert_eq!(val.as_kilopascals(), 170.0);

        // ensure retrieving slave address works
        assert_eq!(inst.get_slave_address(), slave_address);
    }
}
```

Testing through the 32 possible slave addresses is very clean
and understandable.
Personally, I very much prefer this over other tests I wrote
and will migrate to [proptest] for [instrumentRs v0.2.0].

Finally: The [communication functions operation manual]
for the SMC HRR chiller line contains various examples of reads and writes.
Using the `MockInterface`, these could directly be included as tests
for our chiller driver.

## Outlook

I am currently quite happy with the overall framework
for [instrumentRs v0.2.0].
The next big steps will focus on the following:

- A better parser for the `Parameter` derive macro.
- Adding a metadata type to the transport trait, as discussed above.
- A macro to derive `ParameterU8`, e.g., for drivers like the one discussed here.
- Laying out the overall structure of an instrument.
- The instrument generation macro.

Of course, `async` support is also still a goal!

## Comments, questions, ideas?

As always,
if you have comments or questions,
please get in touch with me.
You can do so via [instrumentRs v0.2.0] on GitHub,
[e-mail](mailto:reto@galactic-forensics.space),
[Bluesky],
or [Matrix].
I'm excited to hear your ideas!

[^1]: While "slave address" is an outdated term,
it is used here to be consistent with the vendor documentation.
We follow the [Embassy HAL developer's guide and API guidelines]
where applicable.
Keeping with the same terminology as the vendor
enables the end user to read the documentation
of the chiller and not be confused by differing terminology.
[^2]: If we always had a [serialport] interface to deal with,
we could query the baud rate from the interface.
However, if the chiller is connected to a serial interface, via,
e.g., a [Moxa serial device server], we have no unique possibility
of querying the baud rate.
Thus, taking the slowest baud rate available seems to be a conservative approach.

[Bluesky]: https://bsky.app/profile/galactic-forensics.space
[communication functions operation manual]: https://static.smc.eu/binaries/content/assets/smc_global/product-documentation/operation-manuals/en/om_hrr_w004en-i.pdf
[device-driver]: https://device-driver.com/
[Embassy HAL developer's guide and API guidelines]: <https://github.com/embassy-rs/embassy/blob/150583b7d20b24f9ec3f0c62d3ffab1187bc4d33/API-GUIDELINES.md>
[instrumentRs v0.2.0]: <https://github.com/trappitsch/instrumentRs2>
[instrumentRs-drivers]: https://github.com/trappitsch/instrumentRs-drivers
[last blog post]: <https://blog.galactic-forensics.space/blog/10-derive-macro/>
[LTC2686]: <https://www.analog.com/en/products/ltc2686.html>
[Matrix]: <https://matrix.to/@trappits:epfl.ch#/@trappits:epfl.ch>
[MODBUS]: <https://en.wikipedia.org/wiki/Modbus>
[Moxa serial device server]: https://www.moxa.com/en/products/industrial-edge-connectivity/serial-device-servers
[previous blog posts]: <https://blog.galactic-forensics.space/tags/instrumentrs/>
[proptest]: https://crates.io/crates/proptest
[repo for the ongoing LTC2686 work]: <https://github.com/trappitsch/ltc2686>
[rstest]: https://crates.io/crates/rstest
[serialport]: https://crates.io/crates/serialport
[SMC HRR]: <https://www.smc.eu/en-gb/products/rack-mount-type-hrr%7E178291%7Enav#s>
[smc_hrr branch]: https://github.com/trappitsch/instrumentRs-drivers/tree/smc_hrr
[`std::time::duration`]: https://doc.rust-lang.org/std/time/struct.Duration.html
[`std::time::Instant`]: https://doc.rust-lang.org/std/time/struct.Instant.html
