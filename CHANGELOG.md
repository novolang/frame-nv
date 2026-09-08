# Changelog

Newest first.  Below `1.0.0` a breaking change bumps the **minor**
number and a compatible one the **patch**; see [Version numbers in the
Orbit package registry](https://novo-lang.org/docs/registry/semver.html).

## 0.1.4 — 2026-09-08

- **Declares its layer**: `layer = "core"` in the manifest — the public API requires no effects, and `novo pkg publish` now checks the code against that budget.  No code changed.  The layers are described under Design in the [publishing guide](https://novo-lang.org/docs/publishing.html#design).

## 0.1.3

Documentation: the reference is generated from the code, and the
examples in it are doctests.  No code changed — every byte on the wire
is what 0.1.2 produced, and all four dependency ranges are unchanged.

- **Every `pub` item is documented under Go's rule**, the comment block
  directly above the declaration, its first sentence the summary a
  reader meets before opening anything.  The methods on `Framer` and
  `FrameError.message` carry their own.  `novo doc` turns the lot into
  [the package's page](https://novo-lang.org/packages/frame-nv).
- **Nine worked examples, and they run.**  A frame with a zero in its
  payload is written out in hex with its single trailing delimiter, a
  flipped byte is shown being caught by the check, a frame arriving one
  byte at a time is shown answering on its last, and three integers go
  into a payload and come back out.  A fenced `novo` block in a
  documentation comment is compiled by `novo doc` and run by
  `novo test src/frame.nv`.
- **The README's `FrameError` table is gone.**  The variants and their
  fields are on the generated page now; the README keeps the part a
  generator cannot say — which layer refusing means what, and why the
  next delimiter resynchronises the stream.

## 0.1.2

Developed in its own repository from this version.  `novolang/frame-nv` is
where the sources live, where CI runs and where releases are tagged;
the novo-lang monorepo no longer carries a copy.  No code changed —
every signature and every byte on the wire is what 0.1.1 shipped.

- **`LICENSE` ships with the package.**  It is on the publish
  allow-list, so the tarball now carries the Apache-2.0 text rather
  than only naming it in the manifest.

## 0.1.1

A patch: the same frames, byte for byte.  The hand-computed fixtures,
the streaming reader and the corruption cases all still pass.

- **The manifest carries the fields the registry browses by.**
  `category`, `tags`, `repository` and `maintainers` were added after
  `0.1.0` was published, and a published version is never replaced, so
  this release is the first one the packages page can shelve and
  filter.
- **The packages under it are republished.**  `varint-nv 0.1.1` and
  `leb128-nv 0.1.2` rewrite their bit arithmetic onto the operators
  with their vectors unchanged; the ranges here already admit them, so
  a `novo pkg update` picks the whole diamond up.

There is no bit arithmetic in this module to rewrite: it delegates the
length prefix to `varint-nv`, the checksum to `crc-nv` and the framing
to `cobs-nv`, and what is left is buffer arithmetic.

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
