# Changelog

Newest first.  Below `1.0.0` a breaking change bumps the **minor**
number and a compatible one the **patch**; see [Version numbers in the
Orbit package registry](https://novo-lang.org/docs/registry/semver.html).

## 0.1.0

First release: `max_encoded_len`, `encode_into`, `encode`, `decode`,
`framer` with `push` / `pending_len` / `reset`, the `FrameError` set,
and the `int_len` / `put_int` / `take_int` payload helpers.

- **One frame, three jobs.**  A `0x00` delimiter so a reader can find
  the end, an unsigned LEB128 length so it knows how much is payload,
  and a CRC-32C over both so it can tell a corrupted frame from a good
  one — the failure a delimiter and a length cannot catch.
- **A streaming reader.**  `Framer` takes bytes in whatever pieces a
  link delivers them and answers whole frames; a corrupt one is reported
  rather than silently dropped, and the next delimiter resynchronises.
- **A malformed frame is a value.**  `Truncated`, `BadLength`, `BadCrc`
  and `Cobs` each say where; nothing about bad input panics.
- **The vectors were computed from the format's definition**, not from
  this encoder, so the suite is evidence about the wire format.
- **Depends on `cobs-nv ^0.1.0`, `crc-nv ^0.1.0`, `varint-nv ^0.1.0`
  and `leb128-nv ^0.1.1`.**  The `leb128-nv` floor is `0.1.1` because
  the decoder needs `decode_at`; every other range is the widest this
  package supports.  `varint-nv` reaches `leb128-nv` at `^0.1.0`, so one
  version of it is built — the highest that satisfies both.
