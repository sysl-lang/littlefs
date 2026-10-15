# littlefs for sysl

A small fail-safe filesystem for the flash on a microcontroller, bound to
[littlefs](https://github.com/littlefs-project/littlefs) 2.11.3, vendored unchanged.

- **It survives power loss.** Every update is copy-on-write and commits atomically, so losing power in
  the middle of a write leaves the old version of a file or the new one, never a mix and never a broken
  filesystem.
- **It levels wear.** Flash blocks wear out after 10,000 to 100,000 erases, and littlefs spreads writes
  across them.
- **It needs no heap.** The caches, the allocator's bitmap and each open file's cache are arrays inside
  the sysl values that own them, sized by type parameters.

It is not what a PC reads: for an SD card that moves to a laptop, use FAT
([`sysl-lang/fat`](https://github.com/sysl-lang/fat)).

```hocon
dependencies {
  littlefs { git = "github.com/sysl-lang/littlefs", version = "<the first tag>" }
}
```

The module is `sh.sysl.littlefs`.

## Using it

A device implements `BlockDevice`: read and program pieces of a block, erase a block, sync. Then:

```sysl
import sh.sysl.littlefs.{CREATE, Fs, File, Geometry, READ, WRITE, file, filesystem}

var fs: Fs[Flash, 64, 16] = filesystem(&flash, Geometry(16, 16, 4096, 256, 500))
if fs.mount().is_err() then
    fs.format().unwrap()
    fs.mount().unwrap()

var f: File[64] = file()
fs.open(&f, "settings", WRITE | CREATE).unwrap()
fs.write(&f, "volume=7".bytes).unwrap()
fs.close(&f).unwrap()
```

`Fs[D, CACHE, LOOKAHEAD]`: `CACHE` is the size of the read and program caches and of every open
file's cache (a multiple of the read and program sizes that divides the block size), and the type
checks that a file's cache matches its filesystem's. `LOOKAHEAD` is the allocator's bitmap in bytes,
a multiple of 8, each byte covering 8 blocks. `Geometry` is the device's read, program and block sizes,
its block count, and `block_cycles`: how many erases a metadata block takes before littlefs moves it
(100 to 1000 is usual), or -1 for no wear levelling.

**A mounted `Fs`, an open `File` and an open `Dir` must stay where they are**, because littlefs keeps
their addresses. On a board they are module storage; a copy taken after `mount` or `open` is one
littlefs knows nothing about. (sysl's coming `@pinned` will refuse such a copy: see FINDINGS 3.)

| | |
|---|---|
| filesystem | `format`, `mount`, `unmount`, `fs_info`, `used_blocks`, `gc`, `grow` |
| paths | `mkdir`, `remove`, `rename`, `stat`, `exists`, `get_attribute`, `set_attribute`, `remove_attribute` |
| files | `open` (with `READ`, `WRITE`, `READ_WRITE`, `CREATE`, `EXCLUSIVE`, `TRUNCATE`, `APPEND`), `read`, `write`, `seek`, `tell`, `size`, `truncate`, `flush`, `close` |
| directories | `open_dir`, `read_dir`, `rewind_dir`, `close_dir` |

A path is at most `MAX_PATH` (511) bytes, the room the filesystem keeps to hand littlefs a
NUL-terminated copy. littlefs itself limits each name to 255.

## What it is compiled with, and what is left out

The package compiles littlefs with `LFS_NO_MALLOC` (it never allocates; every buffer is the binding's),
and with `LFS_NO_DEBUG`, `LFS_NO_WARN`, `LFS_NO_ERROR` and `LFS_NO_ASSERT`, so there is no `printf`,
no `<stdio.h>` and no `<assert.h>`. littlefs guards several misuses of its API **only** by assertion,
so with assertions compiled out the binding checks them before it calls: a write or truncate on a file
not opened for writing, a read on one not opened for reading, an open with no access mode, an
operation on a filesystem that is not mounted, a block smaller than 128 bytes, a `block_cycles` of 0,
caches that do not divide the block, a `grow` that would shrink. Each answers an `Error`.

Deliberately not bound: `lfs_fs_traverse`, `lfs_fs_mkconsistent` and `lfs_migrate` (littlefs 1.x
volumes), `lfs_dir_seek`/`lfs_dir_tell`, per-file attributes committed with the file
(`lfs_file_config.attrs`), and `LFS_THREADSAFE`'s lock hooks. Each is an addition rather than a
change, when something needs it.

## Bare metal

The package needs no operating system and no allocator, but on a freestanding target littlefs's C
needs a C library's headers and seven of its string functions, and today a sysl package has no way to
hand it `sysl-lang/cstd`'s. Two sysl changes accepted for alpha.14 close that (FINDINGS 1 and 2); until
then `sysl build-lib . --target thumb-freestanding` stops at `'string.h' file not found`. A target with
a C library (the Pico SDK's newlib, Zephyr, FreeRTOS) has the headers already.

## Tests

`sysl test .` runs 30 tests over a RAM device that behaves as NOR flash (erased bytes are 0xff and
programming only clears bits). Beyond the operations and every error they can answer:

- **Against littlefs driven from C.** A C program written straight against `lfs.h`, built from the same
  vendored source with the same options, runs the same sequence of operations (four rewrites of a 40,000-byte file, a rename across
  directories, an attribute, a seek and patch, a truncate, a directory made and removed); the two images must be the same
  bytes and the two devices must see the same number of reads, programs and erases. The traffic count
  is what sees the cache and lookahead sizes, which never change the image.
- **Power loss at every write.** A file is rewritten with power lost after 0, 1, 2, … programs and
  erases, until the update completes; after each, a fresh mount must read the old version or the new
  one.
- **A worn-out block** is passed over and a 12,000-byte file reads back intact.
- **Wear levelling**: rewriting one file 2,000 times, the most-erased block takes under a third of the
  erases it takes with levelling off.
- The layouts sysl declares for `struct lfs_config` and the others are checked against the header's
  `sizeof` and `offsetof` when the package compiles, for whichever target that is.

Mutations, each run and each red: read and program sizes swapped; the lookahead halved (seen only by
the traffic count); wear levelling off; a bad block reported as an I/O error; `NotFound` mapped to
`Exists`; `Current` and `End` seeking swapped; the power-loss device never losing power; a field of
`Config` moved (refused at compile time); a path handed over without its NUL.

**AddressSanitizer**: `SYSL_EXTRA_CFLAGS="-fsanitize=address -g" sysl test .` passes 30/30, with
`__asan_memcpy` and the report hooks linked into the test binary.

## Vendored

`sh/sysl/littlefs/c/` holds `lfs.c`, `lfs.h`, `lfs_util.c` and `lfs_util.h` from
[littlefs-project/littlefs](https://github.com/littlefs-project/littlefs) release v2.11.3, unchanged,
from the tarball whose SHA-256 is `15521be8aaed133c453e7337714e029a9f62480ab41f59c4ba5d18297a076207`.
Vendoring means upstream fixes do not arrive on their own: watch littlefs's releases and re-vendor.

Licensing: littlefs is BSD-3-Clause; the rest of this package is ISC. Both texts are in `LICENSE`.
