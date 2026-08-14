# Removing libn's syscall layer

Step 1 of `VARIADIC_COLLAPSE.md`. D-047 and D-048 delete `sys1`…`sys5`,
`sys_full1`…`sys_full5`, `sys_safe`, the 7-arg `sys_full`, and
`err_from_syscall` — 14 functions — in favour of the single `sys` builtin.

The collapse plan attached one precondition: `sys_safe` performs per-argument
checks that the builtin does not, so before deleting it, confirm the typed `io_*`
API leaves no way to pass an argument the check would have rejected.

**It does not, and the checks turn out to be worth less than they look.** The
findings below are measured against `ARCHIVE/libn` at its final state.

---

## What `sys_safe` actually checks

Three argument-level checks, at `src/syscall/syscall.npk:111-135`:

| Syscall | Constraint |
|---|---|
| `SYS_IOCTL` | request must be `21523` (`TIOCGWINSZ`), `21531` (`FIONREAD`), or `TCGETS` |
| `SYS_FCNTL` | cmd must be one of 14 named `F_*` values |
| `SYS_MMAP` / `SYS_MPROTECT` | `PROT_EXEC` in prot ⇒ `-EPERM` |

Everything else on the whitelist is passed straight through.

The `PROT_EXEC` check does not survive contact with `wildx`: `libn_mprotect_exec`
already exists and routes around `sys_safe` through `sys_full`, because W^X
memory is a supported regime (D-036). The check restates a policy the language
enforces at the type level, one layer down and by a different mechanism.

---

## The ioctl and fcntl checks: three separate problems

### 1. Both general-purpose wrappers have zero callers

`libn_ioctl(fd, request, arg)` and `io_fcntl(fd, cmd, arg)` are the only public
functions that accept an arbitrary request/cmd. Searched across all 58 source
files:

```
rg -n 'libn_ioctl\(|io_fcntl\(' src -g '*.npk'   →   no matches
```

They are pure escape hatches that nothing escaped through. Deleting them costs
nothing and removes the only route by which an unchecked request code could
reach the kernel.

### 2. Exactly one ioctl request is used in the entire library

`TCGETS`, at `src/io/bio/stdfiles.npk:63`, inside `isatty` — the terminal probe
that decides whether `stdout` is line-buffered or fully buffered.

`TIOCGWINSZ` and `FIONREAD` are admitted by the check but are **not defined as
named constants anywhere in libn**; they appear only as the bare literals
`21523i64` and `21531i64` inside the check itself. The whitelist permits two
requests that no caller can name and no caller uses.

The doc comment directly above the function (`syscall.npk:64`) lists the safe
subset as "`TIOCGWINSZ, FIONREAD`" — omitting `TCGETS`, the only one anything
actually calls. Comment and code have drifted apart in the direction that
matters.

### 3. The fcntl whitelist silently breaks two public functions

`io_fcntl_add_seals` and `io_fcntl_get_seals` (`src/io/fcntl.npk:218-226`) issue
`F_ADD_SEALS` (1033) and `F_GET_SEALS` (1034). Neither value appears in
`sys_safe`'s 14-command list, so both calls take the reject branch and return
`-EINVAL` **unconditionally**. Two public API functions that can never succeed.

`io_fcntl`'s own documentation compounds it:

> Use this only for `F_*` commands not wrapped by the above functions.
> Examples: `F_SEAL_*`, `F_ADD_SEALS`, `F_GET_PIPE_SZ`, `F_SET_PIPE_SZ`.

Every example the escape hatch offers is blocked by the whitelist. The escape
hatch does not escape.

---

## Why this is the argument for D-048, not against it

libn's own header says it plainly (`syscall.npk:53-55`):

> The whitelist exists not as a security boundary but as an API quality signal:
> if your code needs `sys_full()`, you should stop and think about whether you
> really need that power.

It was never a boundary — `sys_full` sits beside it, unrestricted, in the same
module. What it was instead is a **runtime check invisible from the call site**,
and that is what produced the seals bug: a caller writes
`io_fcntl_add_seals(fd, seals)`, the signature and the doc comment both say it
works, and it returns `EINVAL` forever. Nothing in the type system, the
signature, or the compiler could have caught it, because the constraint lives in
an `if` three layers down.

This is the blueprint philosophy's case exactly. A wrapper whose behaviour
depends on a value the caller passes — where the same call is permitted or
rejected depending on a constant the caller cannot see checked — changes meaning
by context. The typed API does not: `io_set_nonblocking(d)` has no slot for a
wrong command, so there is no wrong call to make and no runtime check to forget.

Deleting `sys_safe` therefore removes a check that has never prevented a real
call and has broken two real ones, and it does so *while* the replacement is
strictly narrower.

---

## Resolution

**The precondition is discharged. Delete the layer, and take these with it.**

| Delete | Reason |
|---|---|
| `sys1` … `sys5`, `sys_full1` … `sys_full5` | D-047 — arity padding for a builtin that takes `..*` |
| `sys_safe`, `sys_full` | D-048 — one syscall form |
| `err_from_syscall` | `sys` returns `Result<int64>` directly |
| `libn_ioctl` | zero callers; the only arbitrary-request route |
| `io_fcntl` | zero callers; the only arbitrary-cmd route |

**Keep, and fix:** every typed `io_*` wrapper. `io_fcntl_add_seals` and
`io_fcntl_get_seals` begin working the moment the whitelist is gone — they were
correct all along and the layer beneath them was not.

**Add one typed wrapper**, replacing the raw `TCGETS` in `isatty`:

```nitpick
// io/term.npk — the only ioctl libn ever needed
pub func:io_isatty = bool(fd:d);
```

`TIOCGWINSZ` and `FIONREAD` get **no** wrapper. Nothing uses them, and adding an
API on speculation is the thing that costs a reverification cycle later. If a
terminal-size query is wanted, it arrives then, typed, with a `WinSize` return —
not as an `int64` request code.

Callers needing an ioctl outside the typed API use the `sys` builtin directly.
That is greppable, bannable with `--extra-picky=no-sys`, and — unlike the
whitelist — visible at the call site.

---

## Effect on the ledgers

`SIGNATURE_LEDGER.md` and `PARAMETER_LEDGER.md` both count `libn_ioctl` and
`io_fcntl`, which the collapse plan did not previously mark for deletion.

| | Plan before | After this finding |
|---|---:|---:|
| functions deleted in step 1 | 14 | **16** |
| public functions after collapse | 468 | **467** |

467, not 466: `io_isatty` is added. Its 1 parameter replaces the 6 of the
`libn_ioctl` / `io_fcntl` pair.
