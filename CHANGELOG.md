# Changelog

All notable changes to tar-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.3 — 2026-09-12

### Changed — The README is rewritten in plain technical-writer prose; no signature changed.

## 0.0.2 — 2026-09-10

- **Toolchain floor is 0.8.9**: the bodies and signatures use what 0.8.9 added (`todo()`, a bound effect parameter, the four layers), and the manifest says so instead of letting an older toolchain fail on an undefined function.  No signature changed.

## [0.0.1] — 2026-09-09

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `tar` — the block arithmetic, exposed rather than hidden, because a
  caller doing random access into an archive does the same sums.
  `blocks_for`, `padding_for`, `is_zero_block`, `end_of_archive`, and
  the `TarFormat` choice between USTAR, PAX and the readable-only V7.
- `tarheader` — the 512-byte header block as a value with every field
  typed. No `prefix` field, because USTAR's two-field path split is
  `emit`'s business and not a caller's; no `checksum` field, because a
  stored checksum that disagreed with its block would be a lie a
  caller could propagate. `TarEntryKind` carries the `typeflag` bytes
  including the four the reader consumes, and `has_data` is the rule a
  reader that skipped data blocks after a symlink would break.
- `tarread` — the feed-and-drain reader, which is this package's
  load-bearing interface, and whose output is an EVENT stream rather
  than a list of entries: a tar archive is routinely larger than
  memory, so a reader that returned entries-with-contents would crash
  on the archives people actually have. The four long-path mechanisms
  — PAX `x` and `g`, GNU `L` and `K` — are resolved before a caller
  sees anything. `read_all<S: Read[e]>` is charged whatever the
  caller's stream costs (SPEC § 5.6) and returns the entry list with
  payloads discarded: it is `tar t`, not `tar x`.
- `tarwrite` — headers and data in, 512-byte blocks out. A state
  machine rather than a pure function per entry, because it counts the
  padding a caller cannot and because `end_entry` refuses a payload
  that disagrees with its header's declared size — which turns a
  caller's off-by-one into an error at the entry that caused it rather
  than an archive whose every later entry is at the wrong offset.
- `pax` — POSIX.1-2001's extended-header records, whose length prefix
  counts itself. `records_size` predicts a payload's length without
  building it, and `records_for` / `apply` are the bridge between a
  `TarHeader` and the records it needs.
- `tarerror` — ten reasons, each carrying the 512-byte block it
  happened in, with `byte_of` for the caller about to `dd skip=`.
  `is_recoverable` answers the question a recovery tool has: tar is
  unusually recoverable, and a single bad header need not end a
  restore.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the API suite reaches `not implemented: tar-nv.<fn>`.
  Run it with `--isolate` for one verdict per test naming the function
  it stopped at.
- Nothing here sanitises a path. `path` is a string the caller supplied
  or the reader found, and a caller extracting to disk must sanitise
  every `entry.header.path` and every symlink target before using it.
  The README says so where a reader will meet it.
- GNU sparse files, multi-volume archives and volume headers are
  reported as `UnsupportedEntryKind` with the raw type byte rather than
  guessed at.
