# The exec family

Step 2 of `VARIADIC_COLLAPSE.md`. The plan was to collapse `execl0`…`execl8` and
`execlp1`…`execlp8` — 17 functions — into two variadic functions.

**They should be deleted instead, not collapsed. 17 → 0.** The reasoning, a
safety defect found along the way, and one language question the collapse forces
into the open.

---

## 1. Why `execl` stops earning its place

`execl` exists in C for exactly one reason: C has no array literal, so
`execv(path, argv)` requires a named array declared on a previous line, and
`execl(path, "ls", "-l", NULL)` does not.

Nitpick has `ArrayLiteralExpr`. That reason is gone.

Worse, the `..*` homogeneous rest marker **lowers to building the same slice**
`execv` already takes (`FORMAL_DRAFT` 06 §6.1.3). So the collapsed `execl` would
not be a different operation from `execv` — it would be the same operation with a
second spelling:

```nitpick
execl(path, argv0, "-l", "-a")      // variadic form
execv(path, argv0, ["-l", "-a"])    // slice form — identical lowering
```

Two spellings for one operation is the thing the blueprint philosophy rejects,
and here the redundant one is also **strictly the weaker of the two**: a variadic
argument list is fixed at the call site, so any code written with `execl` that
later needs a runtime-computed argument list must be rewritten to `execv`. The
reverse never happens. Keeping `execl` adds a form whose only distinguishing
property is that it cannot express something its twin can.

`execv`, `execvp`, `execve`, and `execvpe` all stay. Those four are a genuine 2×2
of {explicit path | `PATH` search} × {inherit environment | explicit
environment} — four distinct operations with four names, not arity padding.

---

## 2. `execl0` is the PwnKit primitive, exported as a convenience

`src/proc/exec.npk:241`:

```nitpick
pub func:execl0 = int64(int64:path) {
    stack int64[1]:argv;
    argv[0] = 0i64;
    return execve(path, cast_unchecked<int64>(@argv), environ);
};
```

`argv = {NULL}`, so the exec'd program starts with **`argc == 0`**.

POSIX requires `argv[0]` to be the program name, and essentially all real
programs assume `argv[0]` exists. `argc == 0` is not a curiosity — it is the
entry condition for **CVE-2021-4034 (PwnKit)**, where `pkexec` indexed `argv[1]`
without checking `argc`, read past the end of `argv` into `envp`, and turned a
local account into root. The caller-side ability to launch a program with an
empty `argv` is the primitive that attack needs, and libn hands it out as a
zero-argument convenience wrapper.

Three supporting problems in the same block:

- **The header comment describes a different function than the code.** Line 22
  says `execl0(path, arg0) — execute path with argv = {arg0, NULL}`. The actual
  `execl0` takes no `arg0` at all. Every `execlN` is off by one against its own
  documentation.
- **The two families have different arity floors.** `execl` starts at 0,
  `execlp` starts at 1, with no stated reason. `execlp0` — the same defect for
  the `PATH`-searching path — is absent by accident rather than by decision.
- **The file header advertises `execle`**, which does not exist anywhere in the
  file.

### The fix is structural, not a bounds check

Adding "reject an empty argv" as a runtime test inside `execv` would repeat the
mistake `SYSCALL_LAYER_REMOVAL.md` documents: a constraint invisible at the call
site, enforced by an `if` one layer down.

Make `argv[0]` a **separate mandatory parameter** instead:

```nitpick
pub func:execv   = NIL(char8[]:path, char8[]:argv0, char8[][]:args);
pub func:execvp  = NIL(char8[]:name, char8[]:argv0, char8[][]:args);
pub func:execve  = NIL(char8[]:path, char8[]:argv0, char8[][]:args, char8[][]:envp);
pub func:execvpe = NIL(char8[]:name, char8[]:argv0, char8[][]:args, char8[][]:envp);
```

`argc == 0` is now **unrepresentable** — there is no argument list to leave
empty, because slot 0 is not part of the list. `args` may be empty, which is the
ordinary `argc == 1` case.

This keeps the one legitimate use of a hand-set `argv[0]` — a login shell
launched as `-bash`, `busybox` dispatching on its own name — which a design that
silently derived `argv[0]` from `path` would have removed.

It also removes the repetition that makes the POSIX convention error-prone.
`execl("/bin/ls", "ls", "-l")` requires writing the program name twice and
silently misparses if you forget; `execv(path, argv0, args)` makes the second
name a parameter the compiler requires.

### Return type

`NIL`, per `SIGNATURE_LEDGER.md`. On success these do not return at all; failure
travels in `Result.error`. The current `int64` return is the STATUS case — a
value that is `-1` or nothing.

---

## 3. The question this forces: `char8[]` does not guarantee termination

`execve` hands the kernel a pointer it will read until it finds a NUL. So the
type of `path`, `argv0`, and every element of `args` has to be something
NUL-terminated.

It cannot be `string`. Per `TYPE_REFERENCE.md` §3.1 a Nitpick `string` is
`{ptr, len, cap}` — 24 bytes, length-carrying, **not NUL-terminated**. Passing
one to `execve` reads off the end of the buffer.

The spec's answer is `as_cstring(string) → char8[]`, which produces a
NUL-terminated `char8[]`. So `char8[]` is the type used above. But **`char8[]`
does not carry that guarantee** — it is an ordinary char array, and one built by
hand need not end in `0u8`. The guarantee lives in `as_cstring`'s behaviour, not
in the type, so nothing stops a caller passing an unterminated array and nothing
in the type system flags it.

This is not an exec-specific problem. Every kernel-bound string in nlibc has it:
`open`, `stat`, `getenv`, `chdir`, `mkdir`, `readlink`, `unlink`. It is also
exactly the unbounded-read pattern static analysers cannot discharge, so it will
surface during the Astrée run whether or not it is settled first.

**Recommendation: add a `cstring` type whose only constructor is `as_cstring`.**
Same representation as `char8[]`, distinct name, guarantee by construction:

```nitpick
pub func:execv = NIL(cstring:path, cstring:argv0, cstring[]:args);
```

This is the `fd` and `pid` argument from D-042 applied to strings. An `fd` is
always valid because `-1` is not representable; a `cstring` is always terminated
because there is no way to make an unterminated one. The alternative — a runtime
check of `arr[len-1] == 0u8` in every wrapper — is a check at every boundary
instead of a proof at one, and is what D-042 already rejected once.

**This needs a decision before the signatures are final**, since it changes the
type of roughly every path and name parameter in the library. It is recorded
here rather than assumed; the signatures above use `char8[]` and become `cstring`
if the recommendation is taken.

---

## 4. Effect on the ledgers

| | Plan before | After this finding |
|---|---:|---:|
| step 2 result | 17 → 2 | **17 → 0** |
| public functions after collapse | 467 | **465** |

`execv`, `execvp`, `execve`, and `execvpe` were already counted; they change
signature, not existence. Remove all 17 `execl*` / `execlp*` names from
`src/all.npk:36`, and correct the `execlp1` example in `src/proc/wait.npk:21`,
which is the only remaining reference to the family in the tree.

---

## 5. Verification value

- 17 near-duplicate functions gone, each of which an analyser had to prove the
  same properties about.
- **The `NULL` terminator convention disappears from `argv` and `envp`.** A slice
  carries its length, so the "scan forward until NULL" loop — unbounded, and hard
  to discharge — is replaced by a bounded iteration over a known count.
- **`argc == 0` becomes unrepresentable** rather than merely unused. That is a
  property proven by the signature, requiring no analysis at all.
