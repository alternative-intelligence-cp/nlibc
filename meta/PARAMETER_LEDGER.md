# `nlibc` Parameter Ledger

Companion to `SIGNATURE_LEDGER.md`, which classified return types. This pass
covers **parameters** across the same 608 public functions.

---

## The headline: a quarter of the library is variadic scaffolding

**153 of 608 public functions (25%) exist only to fake variadic arguments.**
Eighteen families are hand-expanded to nine arities each:

`printfN` · `fprintfN` · `eprintfN` · `asprintfN` · `io_printfN` · `io_fprintfN` ·
`io_eprintfN` · `io_dprintfN` · `io_sprintfN` · `scanfN` · `fscanfN` · `sscanfN` ·
`str_snprintfN` · `strbuf_appendfN` · `execlN` · `execlpN` · `sysN` · `sys_fullN`

With the `..*` rest marker specified (`FORMAL_DRAFT` 06 §6.1.3), **each family
collapses to a single function**:

| | Before | After |
|---|---:|---:|
| public functions | 608 | **473** |
| parameters | 1,710 | **852** |

**135 functions and 858 parameters disappear entirely** — half the parameter
surface — because they were never really distinct functions. That is the single
largest reduction available in the port, and it comes from a language feature
that already exists on paper.

---

## Classification of the remaining 852 parameters

| Category | Count | Share | Becomes |
|---|---:|---:|---|
| **ADDRESS** | 346 | 40% | typed pointer — `wild int8->`, `FILE->`, `StrView->`, … |
| **FD** | 84 | 9% | **`fd`** (D-042) |
| **PID** | 9 | 1% | **`pid`** / **`uid`** / **`gid`** (D-042) |
| **SIGNAL** | 16 | 1% | **`sig`** — *proposed* |
| **SIZE** | 129 | 15% | `int64` — unchanged |
| **FLAGS** | 44 | 5% | `int32` — or a bitflag type |
| **VALUE** | 124 | 14% | unchanged |
| **REVIEW** | 100 | 11% | — |
| | **852** | | |

### What actually changes

**346 address parameters** are the D-012 work — the same argument as the
return types, and the larger half of it. **93 become the D-042 identifier
types.**

**253 parameters (29%) need no change** — genuine counts, lengths, offsets, and
scalar operands.

So the parameter pass is roughly **439 changes**, against 253 left alone.


---

## ADDRESS — 346 parameters

**Target:** typed pointer — `wild int8->`, `FILE->`, `StrView->`, …

The core D-012 work. An `int64` parameter holding an address is invisible to leak checking, escape analysis, and Z3 pointer reasoning, and it lets a caller pass a size where a buffer belongs.

Parameter names in this category:

`sv`×41, `s`×32, `buf`×30, `fp`×29, `path`×24, `out`×21, `ptr`×20, `sb`×17, `src`×16, `addr`×14, `dst`×13, `needle`×8, `delim`×7, `prefix`×6, `envp`×5, `name`×5, `argv`×5, `endptr`×5, `it`×5, `e`×5, `suffix`×4, `statbuf`×3, `tmpl`×3, `arg`×3, `haystack`×3, `status_ptr`×3, `fn_ptr`×2, `sv_ptr`×2, `res`×2, `tv`×2, `data_ptr`×1, `entry`×1, `key`×1, `old_buf`×1, `set_ptr`×1, `old_ptr`×1, `str`×1, `fmt`×1, `fields`×1, `lines`×1, `statxbuf`×1


---

## FD — 84 parameters

**Target:** **`fd`** (D-042)

A file descriptor. Distinct type, comparison only.

Parameter names in this category:

`fd`×67, `fds`×7, `newfd`×4, `lock_ptr`×3, `dirfd`×3


---

## PID — 9 parameters

**Target:** **`pid`** / **`uid`** / **`gid`** (D-042)

A kernel identifier. Distinct type, comparison only.

Parameter names in this category:

`pid`×7, `pgid`×1, `tid`×1


---

## SIGNAL — 16 parameters

**Target:** **`sig`** — *proposed*

Signal numbers and signal sets. Candidate for the same treatment as `fd`; see review notes.

Parameter names in this category:

`signo`×10, `mask`×4, `sig`×2


---

## SIZE — 129 parameters

**Target:** `int64` — unchanged

A genuine count, length, offset, or alignment.

Parameter names in this category:

`n`×40, `length`×11, `offset`×10, `size`×8, `out_sz`×6, `buf_size`×6, `nbytes`×6, `align`×5, `num_bytes`×5, `dst_size`×5, `whence`×3, `buflen`×3, `new_size`×3, `len`×3, `nmemb`×2, `pos`×2, `old_size`×2, `max_len`×2, `input_len`×2, `start`×2, `nr`×2, `count`×1


---

## FLAGS — 44 parameters

**Target:** `int32` — or a bitflag type

Mode bits, open flags, `fcntl` commands. Currently bare integers; an enum or bitflag type would be stricter.

Parameter names in this category:

`flags`×18, `mode`×10, `clockid`×6, `prot`×5, `cmd`×2, `advice`×2, `how`×1


---

## VALUE — 124 parameters

**Target:** unchanged

A scalar operand — the value being written, compared, or converted.

Parameter names in this category:

`a`×30, `b`×29, `v`×23, `c`×19, `status`×8, `code`×6, `base`×5, `lo`×1, `hi`×1, `value`×1, `val`×1


---

## REVIEW — 100 parameters, 72 distinct names

Heuristics inconclusive. Most are one-offs and obvious in context; listed in
full so the review is a check rather than a hunt.

`a1`, `a2`, `a3`, `a4`, `a5`, `a6`, `accept`, `args`, `charset`, `dir_path`, `enable`, `end_idx`, `err`, `errnum`, `exp`, `extra_flags`, `f`, `handle`, `handler`, `hlen`, `i`, `include_prefix`, `init_cap`, `input`, `left_out`, `line_out`, `max_fields`, `max_lines`, `min_fd`, `mode_str`, `n_data_pages`, `n_pages`, `nargs`, `new_addr`, `new_flags`, `new_handler`, `new_mask`, `nfds`, `nlen`, `nwake`, `old_addr`, `options`, `other`, `out_file_mode`, `out_open_flags`, `overwrite`, `pattern32`, `pgrp`, `pipefd`, `reject`, `rem`, `req`, `request`, `ret`, `right_out`, `rmtp`, `rqtp`, `saveptr`, `seals`, `src_fd`, `str_arg`, `stringp`, `table`, `target_fd`, `tgid`, `tidptr`, `timeout`, `token_out`, `tp`, `ts_req`, `tz`, `uaddr`


Suggested readings for the recurring ones:

| Name | Likely | Reasoning |
|---|---|---|
| `old_addr`, `new_addr`, `table`, `input`, `tp` | ADDRESS | plainly pointers |
| `*_out` (`left_out`, `token_out`, `line_out`, …) | ADDRESS | out-parameters |
| `n_pages`, `n_data_pages`, `min_fd`, `exp` | SIZE | counts |
| `errnum`, `options`, `accept` | FLAGS / VALUE | codes and bitmasks |
| `a1`–`a4` outside the variadic families | VALUE | raw syscall arguments |

---

## Review notes

### Should `sig` be a type?

16 parameters carry signal numbers or signal sets. D-042's rule is "a
kernel-assigned identifier is not a number" — a signal number is arguably a
*constant from a fixed enumeration* rather than an identifier, which makes it a
better fit for an enum than a new scalar type.

Recommend an **enum**, not a `sig` scalar: the set is closed and known, and an
enum additionally makes `pick` over signals exhaustive.

### Flags deserve better than `int32`

44 parameters are mode bits, open flags, and `fcntl` commands passed as bare
integers. Nothing stops `O_RDONLY` being passed where `PROT_READ` belongs. A
bitflag type per family would catch that, and it is the same class of error
D-042 eliminates for descriptors.

Deferred — it is a language-surface question and a larger design than this port.

### Ordering

Do the **variadic collapse first**. It removes 135 functions and 858 parameters,
so every other pass runs over roughly half the surface. Doing it after the
address rewrite would mean rewriting parameters that are about to be deleted.
