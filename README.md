> Developed and published from this repository.  `frame-nv` began under
> `orbit/frame-nv` in the [novo-lang](https://github.com/novolang) monorepo
> and graduated out of it with its history; issues and pull requests
> belong here.

# frame-nv

A packet on a stream that has no packets. A serial link, a pipe and a
TCP connection all deliver bytes, in whatever sized pieces they feel
like, with no mark where one message ends and the next begins. This
package puts the messages back.

```novo
use frame

fn main() [io]
    let wire = frame.encode(payload)          // delimiter included
    match frame.decode(wire)
        Ok(back) => println("${bytes.len(back)} bytes")
        Err(e)   => println(e.message())
```

```
novo pkg add frame-nv
```

## The frame

```
+-------------+---------+------------------+
| LEB128(len) | payload | CRC-32C, 4 bytes |
+-------------+---------+------------------+
|<------ the check covers this ---->|

        …then COBS over all of it, then 0x00
```

Three things, each doing one job:

- **A delimiter** — `0x00` — so a reader can find the end of a frame
  without being told its length. It works because COBS removes every
  zero from the frame's own bytes, which is why the stuffing is here and
  not optional.
- **A length** — an unsigned LEB128 varint, one byte for anything under
  128 — so a reader knows how much of what it found is payload.
- **A check** — CRC-32C over the length and the payload together — so a
  reader can tell a corrupted frame from a good one. This is the part a
  delimiter and a length cannot do: a frame whose bytes changed in
  transit is still a well-formed frame.

## What it gives you

The API is on [the package's page](https://novo-lang.org/packages/frame-nv),
generated from these sources: every `pub` declaration with its signature,
its effect row and the comment block written above it. A table of names
here would be a second original, and the second original is the one that
goes stale.

## Reading a stream

`decode` needs a whole frame, and a link delivers whatever fits in one
read. `Framer` holds the difference:

```novo
use frame

fn receive(chunks: [Bytes]) [io]
    var f = frame.framer()
    for chunk in chunks
        for result in f.push(chunk)
            match result
                Ok(payload) => handle(payload)
                Err(e)      => println("dropped a frame: ${e.message()}")
```

A frame that does not decode arrives as an `Err` in that list rather
than being silently dropped, so a receiver can count what it lost. Either
way the next delimiter resynchronises the stream — one bad frame costs
one frame, which is the whole point of having a delimiter.

`pending_len` is there for a link that can be fed faster than it is
delimited: a receiver that must cap its buffering asks, and calls
`reset` when the answer is too large or when the link drops.

## What can go wrong, and what you are told

`FrameError` has four variants and every one is a property of the bytes
that arrived rather than of the program that read them:

| Variant | Means |
|---|---|
| `Truncated(at)` | the frame ended before its length prefix said it would; `at` is how many bytes it held |
| `BadLength(at)` | the prefix is unreadable, or names a payload that does not match the bytes present |
| `BadCrc(expected, found)` | the frame's bytes changed on the way |
| `Cobs(inner)` | the stuffing layer refused the bytes, so the frame never became a length and a payload at all |

`e.message()` says which, and where.

## Numbers inside a payload

A payload is bytes, and this package neither imposes a shape on it nor
needs to know one. But the common payload is a handful of integers, and
writing those as fixed eight-byte words spends most of a frame on
leading zeros — so the three calls that put a signed integer in, size it,
and take it back out are here:

```novo
use std.bytes
use frame

fn reading(sensor: Int, celsius: Int) -> Bytes
    var w = bytes.cursor_le(bytes.zeros(frame.int_len(sensor)
                                          + frame.int_len(celsius)))
    let _ = frame.put_int(w, sensor)
    let _ = frame.put_int(w, celsius)
    frame.encode(w.finish())
```

`-1` costs one byte rather than the ten a plain unsigned varint would
spend on it, because the value is ZigZag-folded first. `take_int`
answers a `VarintError` rather than a `FrameError`: the frame was whole,
and what it carried did not parse — those are two different faults and a
receiver treats them differently.

## Writing into a buffer you already hold

`encode_into` takes a cursor and puts the frame in the caller's buffer.
The offset lives in the cursor rather than in a third parameter because
a `Bytes` passed as a parameter is borrowed — the caller still holds it,
so a write through it lands in a copy the caller never sees. A cursor
owns its buffer, so a write through one lands where the caller can read
it. That is the same rule the standard library's own writers follow.

```novo
var out = bytes.cursor_le(bytes.zeros(frame.max_encoded_len(bytes.len(payload))))
let written = frame.encode_into(out, payload)
```

`encode_into` still allocates one buffer of its own — the check covers
the length and the payload, so both have to exist before either can be
stuffed. What it does not allocate is the frame.

`max_encoded_len` is reached exactly by a payload with no zero byte in
it, except where the length, payload and trailer come to a whole number
of 254-byte COBS blocks: there it reserves one byte the encoder does not
need, because a full block at the end of the input stands for no zero
and needs no successor.

## What it costs

Encoding is one pass for the check and one for the COBS stuffing, and
two allocations — the length prefix and the body. Decoding is the
mirror, plus one slice for the payload. `Framer` holds one buffer and
concatenates each chunk onto it, so a receiver fed one byte at a time
pays one copy per byte; a receiver fed whole reads pays one per read.

`encode_into` panics when the cursor has less room than
`max_encoded_len` asks for, exactly as `c.put_u8` and `xs[i]` do: a
buffer the caller sized wrong is a mistake in the program. Malformed
**input** is never a panic — that is what `FrameError` is for.

## What it does not do

It does not decide what a payload means. There is no message type, no
sequence number and no address in the header — a protocol that needs
those puts them in the payload, where a reader that does not know the
protocol cannot misread them.

No retransmission, no acknowledgement, no flow control: this is the
frame, not the link layer above it. No encryption and no authentication
— a CRC catches accidents and stops nobody deliberate.

## Dependencies

It depends on [`cobs-nv`](https://novo-lang.org/packages/cobs-nv) for the
stuffing that makes a zero byte a delimiter,
[`crc-nv`](https://novo-lang.org/packages/crc-nv) for CRC-32C over the
prefix and payload, [`varint-nv`](https://novo-lang.org/packages/varint-nv)
for the signed integers `put_int` and `take_int` spend, and
[`leb128-nv`](https://novo-lang.org/packages/leb128-nv) for the unsigned
length prefix. The ranges are on
[the package's page](https://novo-lang.org/packages/frame-nv), read from
this manifest, with the whole closure folded under them.

`leb128-nv` is `^0.1.1` rather than `^0.1.0` because the decoder needs
`decode_at`, which answers a length prefix **and** the number of bytes it
occupied in one call; `0.1.0` had no way to ask, and a reader that has
the length but not where the payload starts cannot go on. Every other
range is the widest this package supports.

`varint-nv` depends on `leb128-nv` too, at `^0.1.0`. One version of a
package is built into a program, so what you get is the highest release
that satisfies both requirements — `0.1.1` — resolved once and pinned
once in your `novo.lock`. You list `frame-nv`; the five packages under it
arrive with it.

## Tests

```
novo test tests/frame_tests.nv
```

The four frames in the suite were computed from the format's definition
— LEB128 by hand, CRC-32C from the Castagnoli polynomial, COBS from
Cheshire and Baker's description — rather than by running this encoder,
so they are evidence that the bytes on the wire are the bytes the layout
says. Beyond those: a round trip at every payload length up to 300 for
both a zero-free and an all-zero payload, the property that no zero
reaches the wire before the delimiter, each of the five ways a frame can
be refused, and the framer fed one byte at a time, fed several frames at
once, and made to resynchronise after a corrupt one.
