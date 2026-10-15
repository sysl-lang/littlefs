# What vendoring littlefs found in sysl

Each finding is what was being written, the shortest input that shows it, what happened, what should
have, and what the package does meanwhile. Found against sysl 0.1.0-alpha.12. Nothing below is worked
around in the package.

## 1. A package's C cannot be compiled against a dependency's headers (gap)

On a bare-metal target littlefs needs a C library: `lfs_util.h` includes `<string.h>` and
`<inttypes.h>`, and `lfs.c` calls `memcpy`, `memset`, `memcmp`, `strchr`, `strcpy`, `strspn` and
`strcspn`. `sysl-lang/cstd` exists for exactly this. It exports all seven, carries `include/string.h`,
and its manifest names "a vendored C package that includes `<string.h>`" as the consumer it is for. But
nothing lets littlefs's carried C be compiled with `-I <cstd>/include`:

```
sysl build-lib . --target thumb-freestanding
error: the C compiler refused this file's 'c const' block:
sh/sysl/littlefs/c/lfs_util.h:44:10: fatal error: 'string.h' file not found
```

A `requires.headers` requirement plus `--include-path` would make every consumer type a path into its
own package cache. Wanted: a package publishes include directories for the C of the packages that
depend on it. This is the cross-package half of the vendored-C manifest block asked for while
probing TinyUSB, where a library's own include root could not be named either.

**Status: accepted for alpha.14** (sysl branch `vendored-c`, with the TinyUSB block).

## 2. A dependency cannot be taken on some targets only (gap)

cstd must be linked on a freestanding target, and must not be on a hosted one, where its exported
`strlen` and `memcpy` would collide with the system's C library. Nothing in a manifest says "this
dependency only where the target has no C library" (Cargo's `[target.'cfg(…)'.dependencies]`).
Without it, littlefs either cannot name cstd at all, which leaves every bare-metal consumer to wire it
by hand, or breaks every hosted build.

**Status: accepted for alpha.14** (sysl branch `target-deps`).

## 3. Nothing stops a struct being copied after C has taken its address (gap)

`lfs_t` keeps a pointer to its configuration and runs a list through every open `lfs_file_t`, so a
mounted filesystem and an open file must stay where they are. The org's usual answer, a `&T` box,
needs a heap, and littlefs is for boards without one, where the state lives in module storage and is
reached with `*self`. Nothing refuses:

```sysl
fs.mount().unwrap()
var copy = fs      // littlefs still points at `fs`; `copy` is a filesystem it knows nothing about
```

Rust has `Pin`, C++ deletes the copy and move constructors. The package documents the rule (a mounted
`Fs`, an open `File`, an open `Dir` stay put) and has a test that a filesystem moved *before* it is
mounted works.

**Status: accepted for alpha.14 as a `@pinned` struct attribute** (sysl branch `pinned`): a pinned
value is never copied or moved by value. Assignment, by-value arguments, returns, captures,
destructuring and by-value fields of a type that is not itself pinned are all refused. `Fs`, `File` and
`Dir` take it when alpha.14 ships. Since a pinned value cannot be returned, `filesystem()` and `file()`
become initialisers of storage the caller owns, and the moved-before-mount test goes.
