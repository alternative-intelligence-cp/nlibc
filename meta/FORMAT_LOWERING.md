# The format layer

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

## 6. What remains

**Step 5** — `printf` / `fprintf` / `eprintf` / `asprintf` over the buffered
`FILE` layer. Same transformation, emitters targeting the `FILE` buffer. Check
`io_dprintf`'s lesson there too: `fprintf(FILE->, …)` and `io_fprintf(fd, …)` are
genuinely different sinks, but confirm the buffered family has no aliases of its
own before collapsing.

**Step 6 — `scanf`, last and most carefully.** A mismatched `printf` specifier
reads through a bad pointer; a mismatched `scanf` specifier **writes** through
one. The compile-time check must verify that each argument is a pointer of the
type the specifier writes, and that it is writable. Under lowering, each
specifier emits a typed reader taking a typed pointer, so `%d` paired with a
`string` is a compile error rather than a four-byte write into a string's header.
