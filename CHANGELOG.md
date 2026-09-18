# Changelog

Every published version, newest first.  Below `1.0.0` a breaking change
bumps the minor number and a compatible one the patch; see [Version
numbers in the Orbit package
registry](https://novo-lang.org/docs/registry/semver.html).

## 0.1.5 — 2026-09-18

The documentation and comments in plain prose; no signature changed.

## 0.1.4 — 2026-09-08

- The manifest declares `layer = "core"`.  The public API requires no
  effects, and `novo pkg publish` checks the code against that
  declaration.  No code changed.  The layers are described under Design
  in the [publishing
  guide](https://novo-lang.org/docs/publishing.html#design).
- The sources are in the canonical form `novo fmt` prints.  Spacing and
  alignment only; no code changed.

## 0.1.3

Every `pub` item carries its documentation, and the examples in it are
compiled and run.  No code changed.  Every byte on the wire is what
0.1.2 produced, and all four dependency ranges are unchanged.

- The comment block directly above each declaration is that
  declaration's documentation, and its first sentence is the summary a
  reader meets before opening anything.  The methods on `Framer` and
  `FrameError.message` carry their own.  `novo doc` turns the lot into
  [the package's page](https://novo-lang.org/packages/frame-nv).
- Nine worked examples, each of which compiles and runs.  A frame with a
  zero in its payload is written out in hexadecimal with its single
  trailing delimiter.  A flipped byte is shown being caught by the
  check.  A frame arriving one byte at a time is shown answering on its
  last byte.  Three integers go into a payload and come back out.  A
  fenced `novo` block in a documentation comment is compiled by
  `novo doc` and run by `novo test src/frame.nv`.
- The README no longer lists the `FrameError` variants and their fields.
  Those are on the generated page.  The README keeps what a generator
  cannot say, which is what each refusing layer means and why the next
  delimiter resynchronises the stream.

## 0.1.2

The tarball carries the licence text.  No code changed.  Every signature
and every byte on the wire is what 0.1.1 shipped.

- `LICENSE` ships with the package.  It is on the publish allow-list, so
  the tarball carries the Apache-2.0 text rather than only naming it in
  the manifest.

## 0.1.1

A patch: the same frames, byte for byte.  The hand-computed fixtures,
the streaming reader and the corruption cases all still pass.

- The manifest carries the fields the registry browses by.  `category`,
  `tags`, `repository` and `maintainers` were added after `0.1.0` was
  published, and a published version is never replaced, so this release
  is the first one the packages page can shelve and filter.
- The packages under it are republished.  `varint-nv 0.1.1` and
  `leb128-nv 0.1.2` rewrite their bit arithmetic onto the operators with
  their vectors unchanged.  The ranges here already admit them, so a
  `novo pkg update` picks the whole diamond up.

There is no bit arithmetic in this module to rewrite.  It delegates the
length prefix to `varint-nv`, the checksum to `crc-nv` and the framing
to `cobs-nv`, and what is left is buffer arithmetic.

## 0.1.0

First release: `max_encoded_len`, `encode_into`, `encode`, `decode`,
`framer` with `push`, `pending_len` and `reset`, the `FrameError` set,
and the `int_len`, `put_int` and `take_int` payload calls.

- One frame, three parts.  A `0x00` delimiter so a reader can find the
  end, an unsigned LEB128 length so it knows how much is payload, and a
  CRC-32C over both so it can tell a corrupted frame from a good one.
  The last is the failure a delimiter and a length cannot catch.
- A streaming reader.  `Framer` takes bytes in whatever pieces a link
  delivers them and answers whole frames.  A corrupt one is reported
  rather than silently dropped, and the next delimiter resynchronises.
- A malformed frame is a value.  `Truncated`, `BadLength`, `BadCrc` and
  `Cobs` each say where.  Nothing about bad input panics.
- The vectors were computed from the format's definition and not from
  this encoder, so the suite is evidence about the wire format.
- Depends on `cobs-nv ^0.1.0`, `crc-nv ^0.1.0`, `varint-nv ^0.1.0` and
  `leb128-nv ^0.1.1`.  The `leb128-nv` floor is `0.1.1` because the
  decoder calls `decode_at`.  Every other range is the widest this
  package supports.  `varint-nv` requires `leb128-nv` at `^0.1.0`, so
  one version of it is built, and it is the highest that satisfies both.
