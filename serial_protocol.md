# The SparkVial protocol 0.0.1

The protocol has two parts, the **sampling protocol** (The device samples a
sensor and reports it's value periodically), and a **control protocol** (The
master commands outputs/actuators).

## Opening notes
If something in this document is not clear to you, or there's a linguistical
error, you may open an issue at [the repo](https://github.com/SparkVial/spec).

## Glossary
- `line`:  A communication channel between two devices (devices/hubs/a master).
- `device`: A device controlled by the system, a combination of zero or more
  sensors and actuators.
- `hub`: A device that identifies itself as multiple devices, or takes multiple
  physical devices as inputs and identifies as them to the master.
- `master`: There exists one in every system, it commands all `device`s and
  `hub`s below it.

TODO: Add the rest

## Universal assumptions

The line is assumed to be semi-reliable (full reliability is not guaranteed,
but packet loss is assumed to be very low), but may be disconnected and
connected by the user at any time, including in the middle of a message.

Because of this, we favor a low overhead for non-missed packets while high
overhead for missed packets is OK.

Below, lost packet also indicated a disconnected device.

After a timeout of **200ms** between end of request and start of response a master
or a hub must assume the device has been disconnected.

If anyone detects there was a lost packet and cannot recover, they must stop
all communication. The other side should detect this and issue a scan command
(as a master) or wait (as a device).

If anyone detects there was a lost packet and can recover, they must wait 100
ms and then flush their input buffer. The code interacting with the protocol
must be informed if there was a loss of data.

The master *typically* can recover, while devices *typically* can't.

All multi byte fields are little endian.

Device disconnections and connections are assumed to be infrequent, but a new
connection must result in a valid state within 1 second.

## Sampling
The protocol is stream based, the master defines how streams are generated and
periodically requests their contents. Each sample is paired with a timestamp,
and each time the master requests the stream contents he resets the timestamp.
This is because the timestamp is limited by default to 1 minute (2^16/10^3
seconds), aka 16 bits of milliseconds. This can be tweaked by the master at
scan time.

When the device sends a stream chunk as requested by the master, it's sent with
a CRC16. If the CRC16 is correct, the master must send an ACK message, which
lets the device remove the sent data from it's buffer. If the CRC16 is isn't
correct the master must send a NACK message, and the device must resend the
data. A ring buffer is recommended on the device for this purpose.

## Control
This part of the protocol is for infrequent and/or short commands. It has two
main goals: Initialization, Stream management, and Writing.

### Initialization
The master must initialize the session by sending this command, which includes
the *protocol version*, the *timestamp size*, and the *timestamp resolution / interval*.

#### Protocol version
A constant 0. If it is not, the hubs and devices shall not respond.

#### Timestamp size
Defines the size of the timestamp on each sample in bits. Value <=> size mapping is defined in [The actual protocol](#the-actual-protocol).

#### Timestamp resolution / interval
Defines the amount of time each unit of the timestamp represents. Value <=> time mapping is defined in [The actual protocol](#the-actual-protocol).

### Stream management
TODO

### Writing
TODO

## The actual protocol
*NB: '`;`' means a byte boundary, protocol constants are in binary form.*

### Master -> Devices
- Initialize
    - Preamble (`0000`)
    - Protocol version (`0000; 00`)
    - Timestamp size (`xx`)
        - 0 = 8 bits
        - 1 = 16 bits
        - 2 = 32 bits
        - 3 = 64 bits
    - Timestamp resolution / interval (`xxxx;`)
        - 0 = 10^-6 (microseconds)
        - 1 = 10^-3 (milliseconds)
        - 2 = 1 second
        - 3 = 2 seconds
        - 4 = 5 seconds
        - 5 = 10 seconds
        - 6 = 20 seconds
        - 7 = 30 seconds
        - 8 = 1 minute
        - 9 = 2 minutes
        - 10 = 5 minutes
        - 11 = 10 minutes
        - 12 = 20 minutes
        - 13 = 30 minutes
        - 14 = 1 hour
        - 15 = 24 hour
- Heartbeat
    - Preamble (`0001`)
    - Content (`xxxxxxxx;`)
- Create stream
    - Preamble (`0010`)
- Delete stream
    - Preamble (`0011`)
- Read stream
    - Preamble (`0100`)
- Write
    - Preamble (`0101`)