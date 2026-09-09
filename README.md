# tar-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`. Installing this package works;
calling it panics with `not implemented`.

## What this is

The tar format — USTAR (POSIX.1-1988) and PAX (POSIX.1-2001), with V7
readable — as arithmetic over 512-byte blocks, in a package with no
filesystem anywhere in it. A header is a value with every field typed,
not a 512-byte blob you index into. The reader is a feed-and-drain
state machine that emits events, so an archive larger than memory
streams through it. The writer returns blocks and you decide where
they go. Paths, modes, owners and times are values a caller supplies.

It is for the program that has to read or write an archive without
being an extractor: a package manager unpacking a tarball into its own
store, a build tool producing one deterministically, a backup verifier
listing what is inside, `archive-nv` doing the disk half, and a device
unpacking a firmware bundle from a flash region.

```
novo pkg add tar-nv
novo pkg build
novo test
```

## The one example that will work

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

`write_entry` takes the size from the data rather than from the
header, so the one mistake this API can make — a header that declares
a length the payload does not have — is not available in the
convenience form. The streaming form catches it at `end_entry`.

## The layer, and why

`core` — no effects at all, on a package whose whole subject is
archived files.

That is not a contradiction, it is the design. There is no compression
in tar, no index, no checksum over the archive, and no framing beyond
one number: 512. An archive is a sequence of 512-byte blocks; an entry
is a header block plus `ceil(size / 512)` data blocks; the archive ends
with two blocks of zeroes. Everything this package does is integer
arithmetic over bytes the caller already holds, and the largest thing a
reader carries between chunks is one partial block.

The one function that meets a stream stays inside the budget by
**binding** its cost rather than spending one:

```novo
pub fn read_all<S: Read[e]>(src: S) -> Result<[TarEntry], TarError> [e]
```

`S: Read[e]` binds the effect parameter of the standard library's
`Read` trait and the clause uses it, so the row means *whatever the
impl behind `S` supplies*. A file charges its caller `[io]`; an
in-memory buffer charges nothing; `tar-nv` is charged neither.

## The load-bearing interface

```novo
pub fn reader() -> TarReader
pub fn feed(r: TarReader, chunk: Bytes) -> (TarReader, [TarEvent])
pub fn finish(r: TarReader) -> Result<[TarEvent], TarError>
```

Three calls, and the events are the interface's real claim:

```novo
pub enum TarEvent
    TarHeaderSeen(entry: TarEntry)
    TarDataSeen(chunk: Bytes)
    TarEntryDone
    TarArchiveDone
```

**A list of entries would not have worked here.** csv-nv's reader
returns rows because a row is small; a tar entry is a file, and an
archive is routinely larger than memory — that is what tar is *for*. A
reader that returned entries-with-contents would crash on the archives
people actually have. So the output is an event stream, a caller writes
each file out as its bytes arrive, and nothing bigger than a chunk is
ever held. `read_all` exists but returns the entry LIST with the
payloads discarded: it is `tar t`, not `tar x`, and it says so.

The reader resolves tar's four ways of storing an over-long path
before a caller sees anything: a PAX extended header (`x`) is parsed
and applied to the entry that follows, a global one (`g`) is
remembered, and GNU's long-name (`L`) and long-link (`K`) entries
become the next entry's path. None of the four reaches a caller as an
event, because a caller who had to handle them would be writing a tar
reader.

## The reference implementation

GNU tar and Python's `tarfile` for behaviour, with POSIX.1-2017 as the
specification and `bsdtar`'s corpus as the oracle for the archives that
are legal but strange. The distinction that decides what a test may
assert:

**Specification, and binding**

- The 512-byte block, the header layout, the `ustar\0` + `00` magic,
  the 155/100 prefix/name split, the `typeflag` bytes.
- The checksum: the unsigned sum of the 512 bytes with the eight
  checksum bytes counted as spaces. This is the single most common
  defect in a hand-rolled tar writer.
- PAX records as `"%d %s=%s\n"` with the length counting **itself**,
  and the rule that a later record for a key replaces an earlier one.
- The end of an archive: two consecutive zero blocks.
- That a directory, a link, a device and a FIFO have no data blocks.

**Convention, which this package follows and a test may not treat as
correctness**

- GNU's 10,240-byte record padding after the two zero blocks. A tape
  convention; this package writes the two blocks and lets a caller
  append its own zeroes, because it has no idea what it is being
  written to.
- The trailing `/` on a directory's path. Not required, written by
  everyone, and relied on by readers that meet a pre-POSIX NUL
  typeflag.
- Base-256 numeric fields for sizes over 8 GiB. GNU's extension, now
  universal, and what PAX exists to replace.
- The signed-checksum fallback, which some 1980s writers produced from
  a compiler with signed `char`. Accepted on read, never written.

**Deliberately not ported:** GNU sparse files (reported as
`UnsupportedEntryKind` with the raw type byte, so a caller can see what
it has), multi-volume archives, and anything to do with a filesystem —
that is `archive-nv`, which depends on this.

## Where the sans-IO rule shows through

- **`tarheader.header` defaults `mtime` to 0, not to now.** Reading a
  clock is `[time]` and `core` has none. A caller who wants the real
  time passes it to `with_mtime`. The upside is that an archive this
  package writes is reproducible by construction.
- **Nothing here sanitises a path.** `path` is a string the caller
  supplied or the reader found in a header, and this package neither
  rejects `../` nor rewrites an absolute path, because it does not
  know what a path means on your machine and has no filesystem to
  check against. **A caller extracting to disk must sanitise every
  `entry.header.path` before using it**, and the same goes for a
  symlink's `link` target. This is the classic tar vulnerability and
  it lives on the caller's side of this seam by construction.
- **Modes are numbers, not octal strings.** 420 is `rw-r--r--` and 493
  is `rwxr-xr-x`; the format stores octal text and this package
  converts, so the digits are written in one place.

## Status

Every function is `todo()`. `novo test` runs the API suite, and every
assertion in it reaches `not implemented: tar-nv.<fn>` — which is the
expected result until the bodies land, and is what makes the suite a
description of the interface rather than of nothing.
`novo test --isolate` is the readable form: one verdict per test,
naming the function it stopped at.

| module | public types | functions | implemented |
| --- | --- | --- | --- |
| `tar` | 1 | 6 | no |
| `tarheader` | 2 | 14 | no |
| `pax` | 1 | 7 | no |
| `tarread` | 3 | 10 | no |
| `tarwrite` | 1 | 10 | no |
| `tarerror` | 1 | 3 | no |
| **total** | **9** | **50** | **no** |
