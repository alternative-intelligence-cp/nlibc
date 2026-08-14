# The format layer

> **Outcome: the format layer is deleted, not collapsed (D-053).**
>
> This document is the audit that produced that decision, and the findings below
> are the evidence for it — so it is kept intact rather than rewritten. What
> changed: the twelve functions this collapse was converging on are themselves
> removed. `printf`, `scanf`, and every relative leave `nlibc`; output is
> `io_print` / `io_println` / `io_write_str` / `fputs`, each taking one `string`,
> and formatting happens beforehand as ordinary functions returning `string`,
> spliced by `&{ }` interpolation.
>
> The reasoning is in D-053 and is short: every defect catalogued here is a
> format string's fused type/width/bound/count assertion disagreeing with the
> arguments. Separating what to print from how to print it leaves nothing to
> disagree. See §9.

Step 3 of `VARIADIC_COLLAPSE.md` — the string layer the io families depend on.
`str_snprintf0`…`str_snprintf8` and `strbuf_appendf0`…`strbuf_appendf8`, 18
functions, collapse to 2.

This step also settles **how** the format-directed collapse is implemented, so it
governs steps 4–6 (`io_*printf`, the `printf` family, and `scanf`) as well.

```nitpick
// str/strfmt.npk
pub func:str_snprintf   = int64(wild char8->:buf, int64:buf_size, fmt:f, ..*);

// str/strbuf.npk
pub func:strbuf_appendf = int64(StrBuf->:sb, fmt:f, ..*);
```

---

## 1. The engine is careful about arity, and cannot be careful about types

`str_format_args` (`src/str/strfmt.npk:297`) is the core formatter every variant
funnels into. It deserves credit before criticism: **it bounds-checks the
argument index correctly.** All three reads are guarded —

| Line | Read | Guard |
|---|---|---|
| 346 | `width = args[ai]` (`%*`) | `if (ai < nargs)` |
| 368 | `precision = args[ai]` (`%.*`) | `if (ai < nargs)` |
| 398 | `arg_val = args[ai]` | `if (ai < nargs)`, default `0i64` |

More specifiers than arguments therefore yields zeros, not an out-of-bounds read.
`%s` with a missing argument prints `(null)` (line 411). That is the failure mode
handled properly, and most hand-written formatters get it wrong.

**There is also no `%n`.** The classic format-string *write* primitive — which
stores the running character count through a pointer argument, and is the reason
`printf` is a code-execution vector rather than merely an information leak — is
not implemented. Whether by decision or by omission, it is correct, and it should
become an explicit permanent prohibition rather than an accident: **`%n` is never
added to Nitpick's format language, at any point, for any caller.**

### What bounds checking cannot reach

Every argument arrives as `int64`. The types are gone before the engine sees
them, and no amount of index checking recovers them:

```nitpick
str_snprintf1(buf, size, "%s", 42i64)     // well-formed; passes every guard
```

At line 415 the engine executes `str_strlen(sptr)` where `sptr` is `42`. It
dereferences the integer 42 as a pointer and scans forward until it finds a zero
byte. **That is an arbitrary-read primitive**, reachable from a call that is
type-correct in the current signature, with every bounds check passing.

The inverse leaks: `%d` against a pointer argument prints an address, defeating
ASLR. `%f` against an integer reinterprets the bits as a double.

None of this is a defect in `str_format_args`. The information needed to reject
it was destroyed at the call site, by the erasure the arity variants perform. The
fix has to happen before erasure, which is what D-045's `fmt` does.

---

## 2. Don't check at compile time — *lower* at compile time

D-045 requires the compiler to check each specifier against the corresponding
argument's type. Since `fmt` is inhabited only by literals, the compiler holds
the entire format string, and it can do considerably more than check it.

**Recommendation: parse the format string at compile time and emit a
straight-line sequence of typed calls. No runtime format parsing at all.**

```nitpick
str_snprintf(buf, size, "x=%d y=%s\n", x, name)
```

lowers to:

```
fmt_lit(st, "x=", 2)
fmt_i64(st, x, <flags, width, precision as constants>)
fmt_lit(st, " y=", 3)
fmt_str(st, name)
fmt_lit(st, "\n", 1)
```

Each emitter is monomorphic and takes its formatting parameters as compile-time
constants. What this removes:

- **The erasure is gone.** `fmt_str` accepts a `string` or `cstring` and nothing
  else. `str_snprintf(buf, size, "%s", 42i64)` is a compile error, not an
  arbitrary read.
- **The runtime format parser is gone** — the `while` loop over `f[fi]`, the flag
  parsing, the specifier dispatch, roughly 300 lines of state machine. It is not
  in the binary at all, so there is nothing for Astrée to analyse and no path
  through it to prove safe.
- **Arity mismatch is unrepresentable**, rather than degrading to zeros. There is
  no `nargs` because there is no argument vector.
- **`fmt_str` uses the carried length** (D-049), so the `str_strlen` scan at line
  415 disappears. Combined with `cstring`, no format operation performs an
  unbounded read.

`%*` width and `%.*` precision still work — the width becomes a runtime `int64`
parameter to the emitter, type-checked as an integer like any other argument.
Truncation semantics are preserved: each emitter returns what it would have
written, and the sum is `snprintf`'s return value.

### The tradeoff, stated

Straight-line lowering trades a shared runtime parser for per-call-site code. A
call with four specifiers becomes five calls instead of one.

For this project that is the right side of the trade. Format strings are short,
the emitters are shared (only the *sequence* is per-site), and the alternative is
carrying a 300-line interpreter through formal verification and proving that no
call site can drive it into type confusion. The engine is code we would have to
prove correct; the lowering is code that cannot be wrong in that way.

If a binary ever proves genuinely size-constrained, the emitters can be
out-of-lined further. The parser does not come back.

---

## 3. The erased engine cannot survive as a public bypass

`str_format_args` is `pub` and exported from `src/all.npk:23`. Leaving it public
while giving the wrappers `fmt` checking would repeat the `sys_safe` mistake
exactly: a typed API with an untyped bypass sitting beside it, where the
constraint holds only for callers who choose the checked spelling.

It has **no callers outside `strfmt.npk`'s own arity wrappers**, so this is free:

```
rg -n 'str_format_args' src -g '*.npk'   →   only strfmt.npk:532-601 and all.npk:23
```

**`str_format_args` is deleted**, along with the wrappers that call it. Under
compile-time lowering there is no runtime engine for it to be the entry point of.
Remove it from `src/all.npk:23`.

---

## 4. Effect on the ledgers

The 18 → 2 collapse was already counted in the 465 figure. The new delta is the
engine:

| | Before | After |
|---|---:|---:|
| public functions | 465 | **464** |

`str_format_args` is counted at `SIGNATURE_LEDGER.md:648`.

---

## 5. Step 4 — the unbuffered io family: 45 → 3

`io/printf.npk` holds five families of nine: `io_fprintf`, `io_printf`,
`io_eprintf`, `io_dprintf`, `io_sprintf`. The plan was 45 → 5. **Two of the five
are pure aliases**, so it is 45 → 3.

| Family | Outcome | Why |
|---|---|---|
| `io_fprintf0…8` | → 1 | the real implementation |
| `io_printf0…8` | → 1 | `io_fprintf(STDOUT_FD, …)` — a genuine convenience |
| `io_eprintf0…8` | → 1 | `io_fprintf(STDERR_FD, …)` — likewise |
| `io_dprintf0…8` | **→ 0** | every one is `return io_fprintf(fd, …)` |
| `io_sprintf0…8` | **→ 0** | every one is `return str_snprintf(buf, size, …)` |

`io_dprintf` exists in C because `dprintf` writes to a descriptor and `fprintf`
writes to a `FILE*`. Here **both take `int64:fd`** — the distinction that
justifies two names does not exist in this module, since the buffered `FILE*`
version lives in `io/bio/fprintf.npk` and is step 5. Nine functions, one
behaviour.

`io_sprintf`'s header admits the same thing outright:

> These are provided for completeness / naming consistency. For most uses, call
> `str_snprintfN` directly.

Neither family has a single caller anywhere in `libn` — nor does the rest of the
file — so deleting them costs nothing.

`io_perror` is not variadic and stays.

### The double format

Every `io_fprintf` formats its output **twice**:

```nitpick
str_snprintf0(0i64, 0i64, fmt);              // pass 1 — measure, writes nothing
...
str_snprintf0(buf_ptr, len + 1i64, fmt);     // pass 2 — render
```

then writes, and on the long path allocates and frees around it:

```nitpick
if (len < PRINTF_BUF_SIZE) { stack uint8[4096]:buf; … }
else { libn_slab_alloc(len + 1i64); … io_write_n(…); mem_free(p); }
```

So a single print costs two full renders, a 4096-byte stack frame, and — past
4096 bytes of output — a heap allocation and free. **`printf` currently has an
`ENOMEM` failure path.**

One thing to record as correct rather than suspect: `PRINTF_BUF_SIZE` is 4096 and
the buffer is `uint8[4096]`, with the guard `len < PRINTF_BUF_SIZE`, so the
largest `len` is 4095 and the `len + 1` write exactly fits. It is right. It is
also two constants coupled by hand, written separately, repeated across nine
functions — correct today and a one-byte overflow the first time someone edits
one of them. Under lowering the pairing disappears rather than needing a comment.

### What lowering removes here

D-052 removes the runtime parser. In this family it removes the **buffering
strategy** as well, which is the larger win:

- **No measure pass.** Every emitter's output length is either a compile-time
  constant (literals; the digit bound for a numeric specifier) or a value already
  in hand (`string`/`cstring` carry `len`, per D-049). The total is a **sum**,
  computed before emitting — not a second render.
- **No heap.** With the bound known up front, output goes into one stack buffer
  sized from that bound; output larger than the buffer flushes in chunks. So
  **`printf` never allocates, and its `ENOMEM` path ceases to exist.** For a
  library where the failure path of a diagnostic print matters, removing a
  failure mode from the thing you call *to report failures* is worth more than
  the cycles.
- **No hand-coupled constants**, and no 4096-byte frame on call sites whose
  output is provably short.

The one thing lowering must not do is emit a `write` per emitter — that would
turn `printf("x=%d\n", n)` into three syscalls and break output atomicity
between concurrent writers. Accumulate into the sized buffer, then one write.
Buffered streams (step 5) accumulate into the `FILE` buffer instead, which is the
same shape with a different destination.

### Ledger

45 → 3 rather than 45 → 5, so two below plan: **464 → 462**.

---

## 6. Step 5 — the buffered `FILE` family: 36 → 4

`io/bio/fprintf.npk` holds four families of nine: `fprintf`, `printf`, `eprintf`,
`asprintf`. Given step 4's hit rate the first thing to check was whether this
family is an alias of the unbuffered one. **It is not.**

`fprintf0` ends in `bio_fprint_rendered(fp, buf_ptr, len)` — the buffered path —
where `io_fprintf0` ends in `io_write_n(fd, buf_ptr, len)`. Different sinks, and
the difference is semantic: flush timing, write atomicity, and ordering between
the two are genuinely observable. `printf` (buffered, `stdout_fp`) and
`io_printf` (unbuffered, `STDOUT_FD`) coexisting is legitimate, not the
`io_dprintf` situation.

So the plan holds: **36 → 4**, and the ledger is unchanged at 462.

`printf`/`eprintf` are `fprintf(stdout_fp, …)` / `fprintf(stderr_fp, …)`, the
same genuine convenience as their unbuffered counterparts. The double-format,
4096-byte stack buffer, and slab-alloc fallback are identical to step 4 and get
the identical treatment.

One redundancy worth removing while here: `printf0` calls `bio_ensure_std_init()`
and then calls `fprintf0`, which calls `bio_ensure_std_init()` again. It is
idempotent so the behaviour is correct, but every buffered print through the
convenience wrappers initialises twice.

### `asprintf`'s header documents an implementation that does not exist

The comment above `asprintf` (lines 413–420) describes this:

> It renders into the 4096-uint8 stack buffer first; if the output fits, it
> allocates exactly (len+1) bytes and copies. If the format output exceeds 4095
> bytes, the result is truncated (same as snprintf — a known limitation
>
> Returns a heap pointer (as int64) that the caller must free with mem_free().
> … Returns 0 on allocation failure.

The code does none of it. `asprintf0` measures, calls `libn_slab_alloc(len + 1)`,
and renders into the allocation. **There is no stack buffer in any `asprintf`
variant** — all nine `stack uint8[4096]` declarations in the file belong to
`fprintf*` — and consequently **no truncation at 4095 bytes**. On allocation
failure it `fail`s with `ENOMEM`; it does not return 0.

Three disagreements, and the sentence describing the truncation is itself cut off
mid-clause with an unclosed parenthesis.

The dangerous direction is not the stale text but what a reader does with it. The
comment describes a *worse* implementation than the one present, so someone
trusting it either adds a workaround for a limitation that was already fixed, or
"repairs" the code to match the documentation and reintroduces the truncation.
And a caller following the documented `0` convention would test for a failure
value the function never returns, while the real `ENOMEM` propagates past an
unprepared call site.

### `asprintf` becomes `format`, returning `string`

The signature is the C convention Nitpick's type system exists to eliminate — a
raw heap pointer returned as `int64`, the length passed back through an
out-parameter, and a manual `mem_free` obligation on the caller:

```nitpick
pub func:asprintf0 = int64(int64:out_len, int64:fmt);   // before
pub func:format    = string(fmt:f, ..*);                 // after
```

A `string` is `{ptr, len, cap}` and RAII-managed under D-003, so it carries its
own length — `out_len` has nothing to report — and frees itself, so the leak that
the current contract makes possible is not expressible. Under D-052 the size is
known before emitting, so it allocates exactly once with no measuring render.

The name changes because the contract does. `asprintf` means "allocating
`sprintf` returning `char*`"; a function returning a managed `string` is not that
function, and a familiar name promising C semantics while delivering different
ones is worse than an unfamiliar one. This is the same call already made for
`execl` and the syscall wrappers: `nlibc` keeps POSIX *behaviour* where it is
right and drops POSIX *conventions* that the type system supersedes.

`format` is the only member of this family that allocates. It is also the only
one that should — the other three write to an existing sink.

---

## 7. Step 6 — `scanf`: 27 → 3, and why this step mattered most

`io/bio/fscanf.npk` — `fscanf`, `sscanf`, `scanf`, nine variants each — collapses
to 3 as planned. The count is unremarkable; the engine is not.

### What `libn` got right

`bio_scan_str` **traps** when `%s` is used without a width:

```nitpick
if (width <= 0i64 && buf != 0i64) {
    !!! 22i32; // EINVAL: strictly require width bounds on string parsing to prevent buffer overflow
}
```

That closes C's most notorious hole — `scanf("%s", buf)`, an unbounded write into
a caller buffer — and closes it with a trap rather than a recoverable error,
correctly treating it as a programming defect rather than a runtime condition.
Three more things are right: the store is guarded against a null destination
(line 209), running out of arguments stops cleanly rather than reading past the
vector (line 358), and a malformed format string cannot advance the parser past
its NUL (line 339).

### Defect 1 — every integer store is 8 bytes, whatever the destination is

Length modifiers are parsed and then thrown away. The comment says so:

```nitpick
// Skip length modifiers (l, h, hh, ll, z, j, t) — all use int64 anyway
```

and every integer conversion ends at the same store:

```nitpick
(cast_unchecked<int64->>(out))[0] = value;
```

So `%hhd`, `%hd`, `%d`, and `%ld` all write **8 bytes**. In C, `%d` means `int` —
32 bits. A caller doing the C-correct thing:

```nitpick
int32:n;
sscanf1(str, "%d", cast_unchecked<int64>(@n));   // writes 8 bytes into 4
```

overflows the destination by four bytes. `%hhd` into an `int8` overflows by
seven. This is not misuse — it is the conventionally correct spelling, and the
engine cannot detect it because the parameter type is `int64` and the modifier it
would need was discarded three lines earlier.

"All use int64 anyway" is an assumption about the caller that the signature gives
the caller no way to honour or violate visibly.

### Defect 2 — the `%s` width is buffer-size-minus-one

`bio_scan_str` fills to `max = width` and then writes the terminator past it:

```nitpick
while (count < max) { … buf[count] = c; count = count + 1i64; }
if (buf != 0i64) { buf[count] = 0u8; }        // count may equal width
```

`width + 1` bytes total. That is correct C semantics — `%Ns` means N characters
*plus* a NUL — but it means the number in the format string is the buffer size
**minus one**, and `char8[32]` paired with `%32s` overflows by exactly one byte.
`libn` closed the unbounded case and inherited the off-by-one-prone convention
whole.

### Defect 3 — `%n` exists here

`bio_scan_engine` implements `%n` (line 366), storing the consumed count through
a caller pointer. D-052 prohibits `%n`; that prohibition covers this one too, and
the replacement is the same argument: the consumed count should be a **return
value or a typed accessor**, not a pseudo-conversion that writes through a
pointer. `%n` exists in C because there is no other way to get the number out.
Under lowering there is.

### Why lowering is worth more here than anywhere else

Every defect above is a **write** through a caller pointer, and every one of them
becomes a compile error under D-052 — not because the check is stricter, but
because the information arrives intact:

| Defect | Under lowering |
|---|---|
| 8-byte store into a 4-byte destination | the reader is `scan_i32` **because the destination is `int32->`**; the store width comes from the type, and `%d` against an `int8->` is a compile error |
| `%32s` into a `char8[32]` | the bound comes from **the destination's own length**, not a number transcribed into a string; the width specifier becomes redundant, and must agree if written |
| `%n` writing through a pointer | deleted; the count is returned |

The general shape: **C's format string carries a description of the arguments,
and the arguments carry no description of themselves.** Every scanf defect above
is that description being wrong, absent, or discarded. Lowering deletes the
description and reads the arguments directly, so there is nothing left to
disagree.

For `printf` that turns an arbitrary *read* into a compile error. For `scanf` it
turns three classes of arbitrary *write* into compile errors, which is why this
step was ordered last and given the most care.

---

## 8. The collapse, as measured

| Step | Family | Before | After collapse |
|---|---|---:|---:|
| 1 | syscall wrappers | 15 | 1 (`io_isatty` added) |
| 2 | `execl` / `execlp` | 17 | 0 |
| 3 | `str_snprintf`, `strbuf_appendf`, engine | 19 | 2 |
| 4 | `io_*printf` | 45 | 3 |
| 5 | buffered `printf` / `format` | 36 | 4 |
| 6 | `scanf` | 27 | 3 |

**608 → 462 public functions** at the end of the six steps, ~1,710 → ~825
parameters.

Four families turned out to be deletions rather than collapses — the syscall
wrappers, `execl`, `io_dprintf`, and `io_sprintf` — and the audits found an
exported `argc == 0` primitive, an arbitrary-read primitive behind `%s`, an
`ENOMEM` path in `printf`, a header documenting a truncation that does not exist,
and an 8-byte store into whatever the caller supplied.

None of those were the point of the exercise. They were found because collapsing
a family requires reading all nine variants of it.

---

## 9. D-053: the twelve survivors go too

Reading all six steps together made the pattern visible. Every defect above is
the same defect:

| Defect | The assertion that disagreed |
|---|---|
| `%s` against an integer → arbitrary read | type |
| length modifiers discarded → 8 bytes into 4 | width |
| `%32s` into a `char8[32]` → one-byte overflow | bound |
| more specifiers than arguments | count |

A format string fuses the output text, the layout, and a set of assertions about
arguments that sit elsewhere in the call. All four rows are that fusion coming
apart. Compile-time lowering (D-052) catches each one — but only by re-deriving
from the arguments what the format string already claimed, which raises the
question of why the claim is there at all.

**D-053 removes it.** Formatting becomes ordinary functions returning `string`,
and interpolation splices the result:

```nitpick
println(`Total: &{x.prec(2).pad_left(8)}`);
```

### What this deletes from `nlibc`

The twelve functions the six steps converged on:

| Module | Removed |
|---|---|
| `str/strfmt.npk` | `str_snprintf` |
| `str/strbuf.npk` | `strbuf_appendf` (the rest of `StrBuf` stays) |
| `io/printf.npk` | `io_printf`, `io_fprintf`, `io_eprintf` |
| `io/bio/fprintf.npk` | `printf`, `fprintf`, `eprintf`, `format` |
| `io/bio/fscanf.npk` | `scanf`, `fscanf`, `sscanf` |

**Nothing is added.** The string-taking output API already exists and is already
counted: `io_print`, `io_println`, `io_write_str`, `io_write_strln`
(`io/write.npk`) and `fputs` (`io/bio/fstr.npk`).

`scanf` needs no replacement invented either — `str/strconv.npk` already holds
typed parse functions, which is what `sscanf` was approximating badly.

**462 → 450.**

### Where the formatting vocabulary lives

Not in `nlibc`. By D-051's layering, `nlibc` is the POSIX syscall surface;
`prec`, `pad_left`, `pad_right`, `hex`, and their siblings are portable code and
belong in the **stdlib above it**.

The prototype already wrote them. `nitpick/stdlib/fmt.npk` has `fmt_int`,
`fmt_bool`, `fmt_hex`, `fmt_float(x, decimals)`, `fmt_pad_left`, `fmt_pad_right`,
and `fmt_repeat` — each taking a typed value and returning a `string` — and its
header records **no extern calls, only compiler builtins**. It ports cleanly.

Its `fmt` / `fmt2` / `fmt3` / `fmt4` placeholder helpers do **not** come across:
they are a template mini-language, arity-expanded, and interpolation replaces
them exactly.

For composition too complex to read inline, the answer is a builder rather than a
richer format string. `StrBuf` is C-free and stays. The prototype's
`stdlib/string_builder.npk` is superseded — it depends on an
`extern "nitpick_libc_string"` block of three C functions.
