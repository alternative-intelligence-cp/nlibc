# nlibc

The Nitpick-native replacement for the C standard library — memory, strings, I/O,
process control, filesystem, time, and raw syscalls — implemented entirely in
Nitpick with **no C dependencies of any kind**.

The `n` prefix is not decorative. In this ecosystem it asserts that a library has
been **verified free of C**; libraries not yet verified carry the `nitpick-`
prefix instead.

## Why this exists

Nitpick is a safety-critical language subject to formal verification
requirements, and its compiler may not depend on C, C++, Rust, Python, or any
third-party toolchain. Everything in the trusted computing base has to be
verifiable, and an unverified runtime breaks that guarantee outright. There is a
second, sharper reason: once execution crosses the FFI barrier the runtime cannot
intercept a fault and route it through `failsafe`, which defeats the
controlled-shutdown guarantee the language is built around.

So the C library has to be replaced rather than wrapped.

## Relationship to the compiler

`nlibc` is not only an application-facing library. LLVM emits calls to a small set
of runtime symbols on its own — for wide-integer division, and for the large
struct copies and zero-fills that ordinary code produces constantly:

```
__divti3   __udivti3   __modti3   memcpy   memset
```

Every one of those would normally be satisfied by compiler-rt or libc. `nlibc`
supplies them instead, which makes it a hard dependency of any Nitpick program,
not an optional convenience.

## Status

**Early.** The repository is being populated by porting
`REPOS/ARCHIVE/libn/src` — 58 files, ~17,700 lines, 610 public functions — while
bringing it in line with design decisions made for the current compiler. The port
is demand-driven: enough to unblock the compiler first, the rest as it is needed.

Nothing here can be compiled yet; the target compiler does not exist. Code and
tests are written now and executed once it does.

See `meta/NLIBC_PORT_PLAN.md` for the plan, ordering, and per-file checklist.

## Layout

```
src/syscall/   raw syscall layer, errno, constant tables
src/mem/       allocator, mmap, memcpy, memset, memory utilities
src/str/       string views, buffers, conversion, formatting
src/io/        file descriptors, read/write, printf
src/io/bio/    buffered I/O, FILE
src/proc/      process control, env, signals, exec, fork
src/fs/        paths, stat
src/time/      clocks, sleep
src/math/      math primitives
tests/         unit tests
meta/          plans and specifications
```

## Related repositories

| Repository | What it is |
|---|---|
| `nitpick-native` | the compiler this library serves |
| `ARCHIVE/libn` | the original source being ported here — **reference copy, never edited** |
| `nitpick-libc` | ⚠️ **not related** — contains a full `musl-1.2.6` tree and is C. It is precisely what this library replaces. |

The name `nlibc` was chosen over reusing `libn`, which already exists as an
archived GitHub repository.

## License

Apache 2.0 — see `LICENSE`.
