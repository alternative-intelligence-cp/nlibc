# Variadic Collapse

The first porting action, per `PORT_PLAN.md`. **153 hand-expanded functions
become 18.**

Every subsequent pass — the D-012 pointer classification, the D-042 identifier
types, the D-021 cast rewrites — then runs over roughly half the surface. Doing
this after those passes would mean carefully rewriting 858 parameters that are
about to be deleted.

| | Before | After |
|---|---:|---:|
| public functions | 608 | **473** |
| parameters | 1,710 | **852** |

---

## Two mechanisms

**Homogeneous** families take the `..*` rest marker over a typed slice
(`FORMAL_DRAFT` 06 §6.1.3). 27 functions collapse to 4.

**Format-directed** families take a `fmt` parameter — inhabited only by
compile-time string literals — and the compiler checks each specifier against the
corresponding argument's type (D-045). 126 functions collapse to 14.

The second is not merely a collapse. `libn` currently erases every variadic
argument to `int64`, so an argument may be a number or a pointer and the format
string decides which. Collapsing to an erased `..*int64[]` would carry that
hazard across intact.

---

## Homogeneous — 27 → 4

Signatures below are shown **after** the D-012 and D-042 passes, so the intended
end state is visible in one place. The collapse itself only changes the arity.

```nitpick
// proc/exec.npk        execl0 … execl8   (9)  ->  1
pub func:execl  = NIL(wild int8->:path, ..*wild int8->[]:args);

// proc/exec.npk        execlp1 … execlp8 (8)  ->  1
pub func:execlp = NIL(wild int8->:name, ..*wild int8->[]:args);

// syscall/syscall.npk  sys1 … sys5, sys_full1 … sys_full5  (10)  ->  ZERO
//
// These do NOT collapse — they are DELETED (D-047). libn's wrappers are built on
// sys!!!, the raw tier D-001 removed, and the language builtins `sys` and
// `sys_full` now return Result<int64> directly. err_from_syscall goes with them.
// Callers use the builtins.
//
// Before deleting: libn's `sys_safe` holds the curated whitelist that
// BUILTIN_REFERENCE §3 leaves unspecified, including per-argument filtering
// (SYS_IOCTL is admitted only for three specific request codes). That policy is
// extracted into the builtin's specification first.
```

Notes:

- `execl` / `execlp` return **`NIL`** — on success they do not return at all, and
  failure travels in `Result.error`. The current `int64` return is the STATUS case
  from `SIGNATURE_LEDGER.md`.
- The C convention of a trailing `NULL` sentinel **disappears**: a slice carries
  its own length. That removes a real failure mode — a forgotten terminator in
  C reads past the end of the argument list.
- `sys` / `sys_full` keep `int64` returns as COUNT — the syscall result is a
  genuine value, with failure moving to `Result.error`.

---

## Format-directed — 126 → 14

```nitpick
// io/bio/fprintf.npk
pub func:printf         = int64(fmt:f, ..*);
pub func:fprintf        = int64(FILE->:fp, fmt:f, ..*);
pub func:eprintf        = int64(fmt:f, ..*);
pub func:asprintf       = int64(int64->:out_len, fmt:f, ..*);

// io/printf.npk
pub func:io_printf      = int64(fmt:f, ..*);
pub func:io_fprintf     = int64(fd:d, fmt:f, ..*);
pub func:io_eprintf     = int64(fmt:f, ..*);
pub func:io_dprintf     = int64(fd:d, fmt:f, ..*);
pub func:io_sprintf     = int64(wild int8->:buf, int64:buf_size, fmt:f, ..*);

// io/bio/fscanf.npk
pub func:scanf          = int64(fmt:f, ..*);
pub func:fscanf         = int64(FILE->:fp, fmt:f, ..*);
pub func:sscanf         = int64(wild int8->:str, fmt:f, ..*);

// str/strfmt.npk
pub func:str_snprintf   = int64(wild int8->:buf, int64:buf_size, fmt:f, ..*);

// str/strbuf.npk
pub func:strbuf_appendf = int64(StrBuf->:sb, fmt:f, ..*);
```

All return `int64` as COUNT — characters written or fields matched — with failure
in `Result.error`. The POSIX `-1` disappears.

### What the compiler must check

For each call, having parsed the literal `fmt`:

1. **Arity** — specifier count equals argument count. Too few or too many is a
   compile error, not a read past the end of an argument list.
2. **Type per specifier** — `%d` requires an integer type, `%s` requires a string
   or `char8` pointer, `%f` a float or fixed-point type. A `%s` paired with an
   integer is rejected at compile time rather than dereferencing it as an address.
3. **Width and length modifiers** agree with the argument's width.
4. **`scanf` additionally**: every argument is a **pointer** of the type the
   specifier writes, and is writable. This matters more than the `printf` case —
   `scanf` writes through caller pointers, so a mismatch corrupts memory rather
   than printing nonsense.

### Consequence, intended

A format string cannot be computed, stored, or received as data — `fmt` is
inhabited only by literals. Code needing dynamic output composes it with
`StrBuf`, which is already present and type-safe.

---

## Order of work

1. **`sys` / `sys_full`** — **deleted, not collapsed** (D-047). Extract the
   `sys_safe` whitelist policy into the builtin's specification first, then remove
   all 10 wrappers plus `err_from_syscall` and rewrite call sites to the builtins.
2. **`execl` / `execlp`** — 17 to 2, no dependants inside `libn`.
3. **`str_snprintf`, `strbuf_appendf`** — the string layer, needed by the io
   families.
4. **`io_*printf`** — the unbuffered io family.
5. **`printf` / `fprintf` / `eprintf` / `asprintf`** — the buffered layer.
6. **`scanf` / `fscanf` / `sscanf`** — last, and the most careful: the pointer
   checking in step 4 of the previous section is where the safety is won.

Steps 4–6 depend on the compiler implementing `fmt` checking, so the signatures
can be written now while the bodies wait on the frontend.

---

## Verification value

This collapse is worth more to the Astrée run than the line count suggests.

- **135 fewer functions** to analyse, and 858 fewer parameters.
- The eliminated functions were **near-duplicates**, so an analyser had to prove
  the same properties nine times over for each family.
- **Format-string handling stops being a runtime concern at all.** Rather than
  proving that no call site can mismatch a specifier, the property holds by
  construction and there is nothing to prove.
- The `NULL`-terminator convention disappears from `execl`, removing an
  unbounded-read pattern that static analysers flag and that is genuinely hard to
  discharge.
