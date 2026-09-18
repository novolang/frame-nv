# frame-nv

A byte stream has no message boundaries. A serial link, a pipe and a TCP
connection all deliver bytes, in whatever sized pieces they choose, with
no mark where one message ends and the next begins. A frame is one
message on such a stream, wrapped so that a reader can find where it
ends and can tell a corrupted one from a good one. This package writes
and reads that frame.

The frame is built from three published formats. Consistent Overhead
Byte Stuffing removes every zero byte from a run of data, and is
described by Cheshire and Baker in "Consistent Overhead Byte Stuffing",
IEEE/ACM Transactions on Networking, volume 7 number 2. CRC-32C is the
cyclic redundancy check over the Castagnoli polynomial, defined in RFC
3720 appendix B. LEB128 is the unsigned little-endian base 128 varint of
the DWARF debugging format, section 7.6.

## What a frame is

This is the layout, before the stuffing runs over it and the delimiter
is appended.

```
+-------------+---------+------------------+
| LEB128(len) | payload | CRC-32C, 4 bytes |
+-------------+---------+------------------+
|<------ the check covers this ---->|

        …then COBS over all of it, then 0x00
```

Three parts, each answering one question.

A **delimiter** is the byte that ends a frame. Here it is `0x00`. It
lets a reader find the end of a frame without being told the length
first. It is unambiguous because COBS removes every zero from the
frame's own bytes, so the only zero on the wire is the one that ends a
frame.

A **length** is the count of payload bytes. Here it is an unsigned
LEB128 varint, which is one byte for any count below 128. It lets a
reader know how much of what it found is payload.

A **check** is a value computed over the bytes it covers. Here it is
CRC-32C over the length prefix and the payload together. It lets a
reader tell a corrupted frame from a good one. A frame whose bytes
changed in transit is still a well-formed frame, so this is the fault a
delimiter and a length cannot catch.

| Quantity | Value |
| --- | --- |
| Delimiter | `0x00`, one byte |
| Length prefix | unsigned LEB128, one byte below 128 |
| Check value | CRC-32C, 4 bytes, little-endian |
| Bytes the check covers | the length prefix and the payload |
| Most bytes a frame occupies | `frame.max_encoded_len(payload_len)` |
| Zero bytes in a frame before its delimiter | 0 |
| Fixed cost of a frame carrying nothing | 7 bytes |

## Install

```
novo pkg add frame-nv
```

## Example

```novo
use std.bytes
use frame

fn main() [io]
    // A payload with a zero byte in it.
    let payload = bytes.from_hex("11002233") ?? bytes.zeros(0)

    // The frame, delimiter included. It holds one zero, at the end.
    let wire = frame.encode(payload)
    println(bytes.to_hex(wire))   // 030411072233a9ba6bea00

    // The payload back, or the reason the frame was refused.
    match frame.decode(wire)
        Ok(back) => println(bytes.to_hex(back))   // 11002233
        Err(e)   => println(e.message())
```

## What the package contains

| Module | Contents |
| --- | --- |
| `frame` | The whole package. `max_encoded_len`, `encode` and `encode_into`, `decode`, the four `FrameError` variants, the `Framer` streaming reader with `push`, `pending_len` and `reset`, and the `int_len`, `put_int` and `take_int` calls for integers inside a payload. |

Every `pub` declaration, with its signature, its effect row and its
documentation, is on [the package's
page](https://novo-lang.org/packages/frame-nv), generated from these
sources.

## How to choose an entry point

**`frame.encode` takes the whole payload and answers a fresh buffer.**
The buffer is exactly the length the frame needed. This is the call for
a program that holds a message and wants the bytes to write.

**`frame.encode_into` writes into a `Cursor` you already own.** Size the
destination with `frame.max_encoded_len` first. This is the call for a
program that sends on a schedule and wants to allocate once.

**`frame.decode` reads one whole frame.** It accepts the frame with or
without its trailing delimiter. This is the call for a program whose
transport already delivers whole messages.

**`frame.framer` reads a stream.** It holds the bytes of a frame that
has not ended yet, and each `push` answers the frames its chunk
completed. This is the call for a program reading a serial link, a pipe
or a socket.

**`frame.int_len`, `frame.put_int` and `frame.take_int` put signed
integers in a payload.** The value is ZigZag-folded and then LEB128
encoded, so `-1` occupies one byte rather than the ten a plain unsigned
varint spends on it. This is the call for the common payload, which is a
handful of numbers.

## The rules a user needs

1. **A frame carries no zero byte before its delimiter.** COBS removes
   every zero from the length prefix, the payload and the check, so a
   receiver may split its input at each `0x00` and have split it at
   frame boundaries.
2. **`decode` reads one frame, and `Framer.push` splits a stream.** Two
   frames in one buffer are not a frame. The trailing delimiter is
   optional, because a receiver that split at each zero has already
   removed it.
3. **Every fault of the bytes is a value and never a panic.** `decode`
   and `Framer.push` answer an `Err`, so a receiver counts the fault and
   reads the next frame.
4. **The variant says which layer refused the frame.** `Cobs` means the
   bytes never became a length and a payload at all. `Truncated` and
   `BadLength` mean they did, and the length disagreed with what was
   there. `BadCrc` means the check value disagreed with the payload, and
   it is the one that means corruption in transit. The other three
   usually mean the receiver started reading mid-stream.
5. **The next delimiter resynchronises the stream.** One bad frame costs
   one frame. A `Framer` reports the bad one and goes on reading the
   link.
6. **`encode_into` panics when the cursor has less room than
   `max_encoded_len` asks for.** It panics exactly as `c.put_u8` and
   `xs[i]` do, because a buffer the caller sized wrong is a mistake in
   the program. Malformed input is never a panic, which is what
   `FrameError` is for.
7. **`max_encoded_len` is reached exactly by a payload with no zero byte
   in it.** The one exception is a length, payload and trailer that come
   to a whole number of 254-byte COBS blocks. There the bound reserves
   one byte the encoder does not need, because a full block at the end
   of the input stands for no zero and needs no successor.
8. **A `Framer` holds every byte it is fed until a delimiter arrives,
   and nothing caps that.** A receiver that must limit its buffering
   reads `pending_len` and calls `reset` when the answer is too large or
   when the link drops.
9. **`take_int` answers a `VarintError` and not a `FrameError`.** The
   frame was whole, and what it carried did not parse. Those are two
   different faults, and a receiver treats them differently.
10. **Encoding is two passes and two allocations.** One pass computes
    the check and one runs the COBS stuffing. The two allocations are
    the length prefix and the body the stuffing reads. `encode_into`
    still makes the second of them, because the check covers the length
    prefix and the payload and both must exist before either can be
    stuffed. What it does not allocate is the frame. Decoding is the
    mirror, plus one slice for the payload. A `Framer` concatenates each
    chunk onto its buffer, so a receiver fed one byte at a time pays one
    copy per byte, and a receiver fed whole reads pays one copy per
    read.

## What is not included

- **A meaning for the payload.** The payload is bytes, and this package
  neither imposes a shape on it nor needs to know one.
- **A message type, a sequence number and an address.** A protocol that
  needs those puts them in the payload, where a reader that does not
  know the protocol cannot misread them.
- **Retransmission, acknowledgement and flow control.** This is the
  frame and not the link layer above it.
- **Encryption and authentication.** A cyclic redundancy check catches
  accidents, and it stops nobody deliberate.
- **A limit on what a `Framer` holds.** No chunk is refused, so a sender
  that never writes a delimiter would grow the buffer without bound.
  `pending_len` and `reset` are how a receiver caps it.
- **A build for a microcontroller.** Every entry point here allocates or
  speaks `Bytes`, and the package ships no device program.

## Related packages

- [cobs-nv](https://novo-lang.org/packages/cobs-nv) does the stuffing
  that makes a zero byte a delimiter. This package is required at
  `^0.1.0`.
- [crc-nv](https://novo-lang.org/packages/crc-nv) computes the CRC-32C
  over the length prefix and the payload. This package is required at
  `^0.1.0`.
- [leb128-nv](https://novo-lang.org/packages/leb128-nv) encodes and
  decodes the unsigned length prefix. It is required at `^0.1.1` rather
  than `^0.1.0`, because the decoder calls `decode_at`, which answers a
  length prefix and the number of bytes it occupied in one call. `0.1.0`
  has no such call, and a reader that has the length but not where the
  payload starts cannot go on.
- [varint-nv](https://novo-lang.org/packages/varint-nv) encodes and
  decodes the signed integers `put_int` and `take_int` write and read.
  This package is required at `^0.1.0`, and it requires `leb128-nv` at
  `^0.1.0` as well. One version of a package is built into a program, so
  the highest release that satisfies both requirements is resolved once
  and pinned once in your `novo.lock`. You list `frame-nv`, and the five
  packages under it arrive with it.

The ranges are on [the package's
page](https://novo-lang.org/packages/frame-nv), read from this manifest,
with the whole closure folded under them.

## Tests

```
novo test tests/frame_tests.nv
```

The four frames in the suite were computed from the format's definition
rather than by running this encoder, so they are evidence that the bytes
on the wire are the bytes the layout says. LEB128 was computed by hand,
CRC-32C from the Castagnoli polynomial, and COBS from Cheshire and
Baker's description.

Beyond those four, the suite asserts a round trip at every payload
length up to 300 for both a zero-free and an all-zero payload, the
property that no zero reaches the wire before the delimiter, each of the
five ways a frame can be refused, and the framer fed one byte at a time,
fed several frames at once, and made to resynchronise after a corrupt
one.

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
