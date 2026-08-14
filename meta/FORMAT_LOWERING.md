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

## 5. What this sets up for steps 4–6

Steps 4 and 5 (`io_*printf`, then `printf`/`fprintf`/`eprintf`/`asprintf`) are
the same transformation over a different output sink — the emitters write to an
fd or a `FILE` buffer instead of a `char8` buffer, and the lowering is identical.

**Step 6, `scanf`, is where this matters most.** `printf` with a mismatched
specifier reads through a bad pointer; `scanf` *writes* through one. The
compile-time check there must verify not only that each argument is a pointer of
the type the specifier writes, but that it is writable. Under lowering, a
`scanf` specifier emits a typed reader taking a typed pointer, so `%d` paired
with a `string` is a compile error rather than a four-byte write into a string's
header.

That is the step to do last and to spend the most care on, exactly as
`VARIADIC_COLLAPSE.md` orders it.
