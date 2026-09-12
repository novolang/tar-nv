# tar-nv

tar is an archive format: many files concatenated into one, each preceded by a
header. It is specified by
[POSIX.1-2017](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/pax.html#tag_20_92_13_06),
in two versions — USTAR (POSIX.1-1988) and PAX (POSIX.1-2001). This package
reads and writes both in novo-lang, with V7 archives readable, and touches no
filesystem at all.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared with
its full signature, but every body is a `todo()` that panics when called. The
package is published so its design can be reviewed and depended on before it is
implemented. Version 0.1.0 will be the first working release.

## What tar is

tar has no compression, no index, no checksum over the archive, and no framing
beyond one number: 512.

An archive is a sequence of 512-byte blocks. An entry is one header block
followed by `ceil(size / 512)` data blocks, the last padded with zeroes. The
archive ends with two blocks of zeroes.

A header block holds the path, the size, the mode, the owner, the modification
time and a type byte, each as fixed-width octal or ASCII text, plus a checksum
over the block itself. A `.tar.gz` is one of these archives run through gzip
afterwards; the compression is not part of the format.

| Value | Size |
| --- | --- |
| Block | 512 bytes |
| Header name field | 100 bytes |
| Header prefix field | 155 bytes |
| End of archive | two zero blocks |

USTAR's 100-byte name and 155-byte prefix cannot hold every path, and its octal
size field cannot hold a file over 8 GiB. PAX answers both with *extended
records*: a pseudo-entry whose data is a list of `"%d %s=%s\n"` lines, applying
to the entry that follows.

## Install

```
novo pkg add tar-nv
```

## Example

```novo
use tarwrite
use tarheader
use tar

fn one_file_archive(name: Str, body: Str) -> Result<Bytes, TarError>
    let w = tarwrite.writer(PaxFormat)
    let (w2, entry) = tarwrite.write_entry(w, tarheader.header(name, 0), bytes.from_str(body))!
    let (_, tail) = tarwrite.finish(w2)!
    Ok(bytes.concat([entry, tail]))
```

`write_entry` takes the size from the data rather than from the header, so the
one mistake this interface could make — a header declaring a length the payload
does not have — is not available in the convenience form. The streaming form
catches it at `end_entry`.

Build and test with:

```
novo pkg build
novo test
```

Today `novo test` fails on purpose: every assertion reaches
`not implemented: tar-nv.<fn>`. `novo test --isolate` is the readable form, with
one verdict per test naming the function it stopped at.

## What the package contains

| Module | Contents |
| --- | --- |
| `tar` | The format's arithmetic: the block size, how many blocks a payload takes, the padding, the zero-block test, and the end-of-archive blocks. |
| `tarheader` | A header as a value with every field typed, its constructors and setters, the parser, the emitter, the checksum rule, and the USTAR path test. |
| `pax` | PAX extended records: parsing, emitting, lookup, and applying a record set to a header. |
| `tarread` | The reader: `feed` and `finish`, the events they emit, `read_all` over a stream you supply, and `list`. |
| `tarwrite` | The writer: `write_header`, `write_data`, `end_entry`, the one-call `write_entry`, and `finish`. |
| `tarerror` | `TarError`, its message, the block and byte it happened at, and `is_recoverable`. |

Nine public types and 50 public functions. No function in this package opens a
file, reads a clock or performs any input or output. Paths, modes, owners and
times are values a caller supplies.

The one function that meets a stream charges its caller rather than declaring an
effect of its own:

```novo
pub fn read_all<S: Read[e]>(src: S) -> Result<[TarEntry], TarError> [e]
```

A file charges `[io]`; an in-memory buffer charges nothing.

## How to choose an entry point

**Reading an archive: `tarread.reader`, `feed` and `finish`.**

```novo
pub fn reader() -> TarReader
pub fn feed(r: TarReader, chunk: Bytes) -> (TarReader, [TarEvent])
pub fn finish(r: TarReader) -> Result<[TarEvent], TarError>
```

`feed` answers events, not entries:

```novo
pub enum TarEvent
    TarHeaderSeen(entry: TarEntry)
    TarDataSeen(chunk: Bytes)
    TarEntryDone
    TarArchiveDone
```

A tar entry is a file, and an archive is routinely larger than memory — that is
what tar is for. A reader that returned entries with their contents would fail
on the archives people actually have. With events, a caller writes each file out
as its bytes arrive and nothing bigger than a chunk is ever held.

**Listing an archive: `tarread.list` or `tarread.read_all`.** Both answer the
entries with the payloads discarded. They are `tar t`, not `tar x`.

**Writing an archive: `tarwrite.write_entry` when the file is in memory**, or
`write_header`, `write_data` and `end_entry` when it is not. `finish` writes the
two zero blocks.

**Composing with compression:** run this package's output through
[flate-nv](https://novo-lang.org/packages/flate-nv) for a `.tar.gz`. It is a
composition the caller makes, which is what lets a program that only needs
`.tar` avoid a DEFLATE implementation entirely.

## The rules a user needs

1. **Nothing here sanitises a path.** `path` is a string the caller supplied or
   the reader found in a header. This package neither rejects `../` nor
   rewrites an absolute path, because it does not know what a path means on your
   machine and has no filesystem to check against. **A caller extracting to disk
   must sanitise every `entry.header.path` before using it**, and the same goes
   for a symlink's target. This is the classic tar vulnerability, and it lives
   on the caller's side of this boundary by construction.
2. **`tarheader.header` defaults the modification time to 0, not to now.** This
   package reads no clock. Pass the real time to `with_mtime` if you want one.
   The upside is that an archive this package writes is reproducible by
   construction.
3. **Modes are numbers, not octal strings.** 420 is `rw-r--r--` and 493 is
   `rwxr-xr-x`. The format stores octal text and this package converts, so the
   digits are written in one place.
4. **The reader resolves all four ways of storing an over-long path before a
   caller sees anything.** A PAX extended header is parsed and applied to the
   entry that follows; a global one is remembered; and GNU's long-name and
   long-link entries become the next entry's path. None of the four reaches a
   caller as an event, because a caller who had to handle them would be writing
   a tar reader.
5. **The checksum counts the eight checksum bytes as spaces.** It is the
   unsigned sum of the 512 bytes with that substitution. This is the single most
   common defect in a hand-written tar writer. `tarheader.checksum` is public.
6. **A later PAX record for a key replaces an earlier one**, and a record's
   declared length counts itself.
7. **A directory, a link, a device and a FIFO have no data blocks.**
   `tarheader.has_data` answers the question.
8. **`tarheader.fits_ustar` says whether a path needs PAX**, before anything is
   written. A path that does not fit and a format of USTAR is
   `PathTooLong(path, cap)`.

The errors are `BadHeaderChecksum(block, want, got)`, `BadUstarMagic(block)`,
`BadNumericField(block, field, text)`, `ShortBlock(offset, got)`,
`UnexpectedEnd(offset)`, `BadPaxRecord(block, why)`, `PathTooLong(path, cap)`,
`FieldTooLarge(field, value)`, `UnsupportedEntryKind(block, kind)` and
`TarSourceFailed(cause)`. Each carries the block and the byte it happened at,
through `tarerror.block_of` and `tarerror.byte_of`, and
`tarerror.is_recoverable` says whether a reader can carry on past it.

## What is specification and what is convention

**POSIX, and binding on this package:**

- The 512-byte block, the header layout, the `ustar\0` and `00` magic, the
  155-and-100 prefix-and-name split, and the type bytes.
- The checksum rule in rule 5.
- PAX records as `"%d %s=%s\n"` with the length counting itself, and a later
  record replacing an earlier one for the same key.
- Two consecutive zero blocks at the end of an archive.
- That a directory, a link, a device and a FIFO have no data blocks.

**Convention, which this package follows and a test may not treat as
correctness:**

- GNU's 10240-byte record padding after the two zero blocks. It is a tape
  convention. This package writes the two blocks and lets a caller append its
  own zeroes, because it has no idea what it is being written to.
- The trailing `/` on a directory's path. Not required, written by everyone, and
  relied on by readers that meet a pre-POSIX type byte.
- Base-256 numeric fields for sizes over 8 GiB. GNU's extension, now universal,
  and what PAX exists to replace.
- The signed-checksum fallback, which some 1980s writers produced from a
  compiler whose `char` was signed. Accepted on read, never written.

## What is not included

- **GNU sparse files.** They are reported as `UnsupportedEntryKind` with the raw
  type byte, so a caller can see what it has.
- **Multi-volume archives.**
- **Compression.** See "How to choose an entry point".
- **Anything to do with a filesystem.** That is
  [archive-nv](https://novo-lang.org/packages/archive-nv), which depends on this
  package and declares its own effects.
- **Path sanitisation.** See rule 1.

## Related packages

This package has no dependencies.

- [archive-nv](https://novo-lang.org/packages/archive-nv) is the half that
  touches a disk: reading a directory into an archive and extracting one out of
  it.
- [flate-nv](https://novo-lang.org/packages/flate-nv) compresses the result into
  a `.tar.gz`.
- [zip-nv](https://novo-lang.org/packages/zip-nv) is the other archive format,
  and differs in that it has an index and per-entry compression.

## The reference implementation

GNU tar and Python's `tarfile` for behaviour, with POSIX.1-2017 as the
specification and `bsdtar`'s corpus as the oracle for the archives that are
legal but strange.

## Implementation status

| Module | Public types | Functions | Implemented |
| --- | --- | --- | --- |
| `tar` | 1 | 6 | no |
| `tarheader` | 2 | 14 | no |
| `pax` | 1 | 7 | no |
| `tarread` | 3 | 10 | no |
| `tarwrite` | 1 | 10 | no |
| `tarerror` | 1 | 3 | no |
| **Total** | **9** | **50** | **no** |

## Licence

Apache-2.0. See `LICENSE`.
