# `nlibc` Port Plan

Plan for porting `REPOS/ARCHIVE/libn/src` into this repository. **No code ported
yet** — this is the plan to execute against.

Design decisions referenced here (D-001 … D-014) live in
`../nitpick-native/meta/specs/DECISIONS.md`. The asset assessment this plan is
based on is `../nitpick-native/meta/ASSET_REVIEW.md`.

Ground rules:

- `ARCHIVE/libn` is the **reference copy and is never edited**. A deny rule for
  `REPOS/ARCHIVE/**` is in this repo's settings to enforce that.
- Port `src/` only. The archive's root holds 159 loose `.npk` files, 676 Python
  fix-up scripts, `scratch/`, and `_workspace_cleanup/` — all porting debris.

---

## 1. What is being ported

**58 files, 17,668 lines, 610 public functions.** By layer, in dependency order:

| Layer | Files | Lines | Pub fns | Depends on |
|---|---|---|---|---|
| `syscall/` | 4 | 2,178 | 72 | — (foundation) |
| `mem/` | 5 | 1,776 | 46 | syscall |
| `str/` | 11 | 4,166 | 138 | mem |
| `io/` + `io/bio/` | 20 | 6,247 | 220 | mem, str, syscall |
| `proc/` | 6 | 1,619 | 74 | syscall, str |
| `fs/` | 2 | 770 | 13 | syscall, str |
| `math/` | 1 | 589 | 38 | — |
| `time/` | 4 | 271 | 7 | syscall |
| root (`all.npk`, `failsafe.npk`) | 2 | 52 | 1 | all |

Largest single files: `strview.npk` (813), `strconv.npk` (781), `alloc.npk` (739),
`fs/path.npk` (714), `syscall.npk` (680), `fscanf.npk` (668), `posix_constants.npk`
(661), `strfmt.npk` (602), `math.npk` (589), `fprintf.npk` (546).

---

## 2. Repo scaffolding

`nlibc` currently has `meta/`, `src/`, `test/`, an empty `README.md`, and a
`.gitignore` covering `.internal`.

### 2.1 Proposed `.gitignore`

```gitignore
# Working area — never committed
.internal/

# Build output
build/
*.o
*.a
*.bc
*.ll
*.s
a.out

# Editor / OS
*.swp
.DS_Store
```

The archive's biggest lesson is that build artifacts and scratch work drowned the
real source — 159 loose files and 676 scripts around 58 files of library. The
ignore list should be aggressive from day one.

### 2.2 Proposed directory structure

Mirror the archive's layer split, which is already sound:

```
src/syscall/   src/mem/   src/str/   src/io/   src/io/bio/
src/proc/      src/fs/    src/time/  src/math/
test/          meta/
```

### 2.3 `README.md` — content to cover

- What `nlibc` is: the Nitpick-native libc replacement — I/O, strings, memory,
  process, syscalls — with **no C dependencies**.
- That the `n` prefix asserts verified C-free status.
- Its relationship to `nitpick-native`: `nlibc` supplies the runtime symbols the
  compiler cannot obtain from a system libc (D-011).
- That it supersedes the archived `libn`, and where the archive lives.
- Explicit warning that `REPOS/nitpick-libc` is a musl tree and unrelated.

### 2.4 Manifest

`npkc-native`'s manifest referenced `libn = { path = "../libn" }`. A
`nitpick.toml` for `nlibc` should declare `name = "nlibc"` so dependents can
reference `nlibc = { path = "../nlibc" }`. Confirm the manifest schema against
`nitpick-docs/specs/build_system_specs.txt` before writing it.

---

## 3. The D-012 classification pass

**This is the bulk of the work and it must happen before or during the port of
each file — not after.**

Every one of the 610 public functions needs its return and parameter types sorted:

| Category | Detection | Becomes |
|---|---|---|
| **Address** | returns/accepts a pointer value | `wild int8->` for untyped memory, typed pointer where known |
| **Size / count** | genuinely numeric | stays `int64` |
| **Status / error** | integer encodes success/failure | `NIL`, failure moves to `Result.error` |

The status category is the one that will be missed, because the current code
looks correct. `pub func:mem_free = int64(int64:ptr)` becomes
`pub func:mem_free = NIL(wild int8->:ptr)` — the integer status is a **double
encoding** alongside the implicit `Result` wrapper.

Recommended approach: produce a **complete signature ledger** (all 610, with
current and target types) as a reviewable artifact before editing anything. Sign
that off, then apply mechanically. This avoids relitigating individual signatures
mid-port.

---

## 4. Port order

Follow the dependency chain, not file size.

1. **`syscall/`** — foundation; nothing works without it. Mostly constants and
   thin wrappers, so the classification pass is easy here. Good place to
   establish the conventions.
2. **`mem/`** — **highest priority after syscall.** Contains `memcpy.npk` and
   `memset.npk`, which D-011 identified as symbols LLVM emits behind our back and
   which the compiler itself will need. Also `alloc.npk`, where the D-012 address
   change has the most impact.
3. **`str/`** — large but self-contained once `mem` is up.
4. **`io/` + `io/bio/`** — biggest layer (6,247 lines, 220 functions). Defer;
   nothing else depends on it.
5. **`proc/`, `fs/`, `time/`, `math/`** — independent leaves, any order.
   `math/` has no dependencies and could be done at any point.

### Strategy: demand-driven, targeting the new compiler only

**Port only enough to unblock, then add as needed.** Everything ported targets the
new design; nothing is written against the prototype. This avoids the double
migration that porting-to-prototype-then-migrating would guarantee, and keeps the
unbuildable surface small while the compiler does not yet exist.

### Minimal unblocking slice — 7 files, ~3,480 lines

Derived from actual `use` declarations, not guesswork:

| File | Lines | Depends on |
|---|---|---|
| `mem/memset.npk` | 184 | **nothing — freestanding** |
| `mem/memcpy.npk` | 221 | imports syscall/errno, but barely uses them (2 refs) — likely vestigial, verify and drop |
| `syscall/errno.npk` | 403 | — |
| `syscall/syscall_numbers.npk` | 434 | — (constant table) |
| `syscall/posix_constants.npk` | 661 | — (constant table) |
| `syscall/syscall.npk` | 680 | the above |
| `mem/mmap.npk` | 158 | syscall layer |
| `mem/alloc.npk` | 739 | mmap, memcpy, memset, syscall, math |

**Start with `mem/memset.npk`** — 184 lines, zero imports, exercises the full
D-012 classification (it has both address and status returns) with no dependency
risk whatsoever. It is the ideal file for establishing conventions.

Two of the three constant-heavy syscall files (`syscall_numbers`,
`posix_constants`, 1,095 lines combined) are tables with no public functions —
mechanical to port and low-risk.

`alloc.npk` imports `math/math.npk`; check whether that dependency is real or
whether the few uses can be inlined, to avoid dragging in 589 more lines.

### Two bootstrap gotchas — resolve before writing `memcpy`/`memset`

**1. Self-reference.** LLVM lowers large struct copies and zero-fills into
`memcpy` / `memset` calls (D-011). If `nlibc`'s own `memcpy` implementation
contains a construct that lowers that way, **it calls itself infinitely**. This is
a well-known libc trap, normally handled with freestanding/no-builtin compilation
of exactly these files. The new compiler needs an equivalent mechanism, and it
must exist before these two files are compiled.

**2. Symbol binding.** LLVM emits calls to literally `memcpy` and `memset` with C
ABI. `libn` exports `mem_memcpy`, `mem_memset`, `mem_memmove`, `mem_bzero` — none
of which satisfy that. Something must bind implementation to emitted symbol name.

Precedent exists: `memcpy.npk` already defines
`pub func:__intrinsic_memcpy = bool(int64:dst, int64:src, int64:num_bytes)`,
commented as "Compiler intrinsic hook for inline memcpy." The prototype clearly
had a mechanism here — **investigate how it bound that hook before designing a new
one.** Whatever the answer, it needs spec support (an export-name attribute, or a
compiler-known intrinsic table) and it interacts with D-011's undefined-symbol
build check.

### Full port order, once unblocked

1. **`syscall/`** — foundation; mostly constants, easy classification pass
2. **`mem/`** — the rest of it (`memutil.npk`, 474 lines)
3. **`str/`** — large but self-contained once `mem` is up
4. **`io/` + `io/bio/`** — biggest layer (6,247 lines, 220 functions); defer,
   nothing depends on it
5. **`proc/`, `fs/`, `time/`, `math/`** — independent leaves, any order

---

## 5. Per-file audit checklist

Applied to each file as it is ported:

- [ ] **D-012** — every signature classified (address / size / status)
- [ ] **`wild` qualifiers** added to unmanaged allocation returns
- [ ] **D-008** — `tbb` cast sites reviewed; width changes and `tbb`↔integer
      conversions are no longer straight casts. *(No bitwise-on-`tbb` violations
      were found in the archive.)*
- [ ] **Legacy type names** — `fix256_t` / `tfp64_t` renamed per `SPEC_GAPS` §3
      (`tfp` plain, `dim` for dimensional analysis)
- [ ] **`raw` audit** — 35 files use `raw`; each is a deliberate safety bypass and
      should be justified or removed
- [ ] **`unknown` audit** — 1 file uses it; check against `TYPE_REFERENCE.md` §27's
      narrowed meaning
- [ ] **D-006** — `.` for all member access; `->` only in type position
- [ ] No `extern` blocks introduced (archive `src/` has none — keep it that way)

Not applicable, confirmed by measurement: `gc` (0 uses), `wild`/`wildx` (0),
`$$i`/`$$m` (0), `defer`/`nodrop` (0), `arena`/`Handle` (0), `ok()` (0).

---

## 6. Open questions before starting

**Settled:**

- ~~Target prototype or new compiler?~~ → **New compiler only**, demand-driven
  porting (§4). Avoids a guaranteed double migration.
- ~~Should a library supply `failsafe`?~~ → **No** (D-013). Executables only, one
  per program. `ARCHIVE/libn/src/failsafe.npk` is not ported.

**Still open:**

1. **The two bootstrap gotchas** (§4) — self-referential `memcpy`/`memset`, and
   symbol binding for compiler-emitted calls. Both must be resolved before those
   two files are compiled, and the second needs spec support.
2. **Does `defer` run when trapping to `failsafe`?** (D-013 follow-on.) Affects
   whether `nlibc` can rely on `defer` for cleanup on the trap path. Recommended
   answer is no.
3. **Test strategy.** The archive has `tests/unit` (47 files) outside `src/`.
   Worth assessing separately for reuse; `nlibc/test/` is currently empty. Note
   nothing can actually be *run* until the compiler exists — the tests are
   written now and executed later.
4. **Confirm the manifest schema** against `build_system_specs.txt` before
   writing `nitpick.toml`.
5. **Is `alloc.npk`'s dependency on `math/math.npk` real?** If only a few
   functions are used, inlining them avoids pulling in 589 lines.
