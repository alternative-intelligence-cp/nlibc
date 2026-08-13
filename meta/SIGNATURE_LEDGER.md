# `nlibc` Signature Ledger

Classification of every public function in `ARCHIVE/libn/src` against **D-012**
(the `int64`-as-address convention) and the universal `Result<T>` rule.

**608 public signatures across 58 files.** Generated mechanically, then reviewed.

> **Purpose:** settle the convention *once*, here, before porting begins. D-012
> notes that the pointer question touches nearly every public signature and is far
> cheaper to decide in one pass than to discover mid-port. This is that pass.

## Summary

| Category | Count | Becomes |
|---|---:|---|
| **ADDRESS** | 66 | `wild int8->` or a typed pointer |
| **STATUS** | 102 | `NIL` |
| **PREDICATE** | 5 | `bool` |
| **HANDLE** | 4 | `FILE->` |
| **FD** | 19 | `int32` — file descriptor |
| **PID** | 26 | `int32` — pid / uid / gid |
| **COUNT** | 227 | `int64` — count, with failure moving to `Result.error` |
| **NUMERIC** | 65 | `int64` — unchanged |
| **KEEP** | 94 | unchanged |
| | **608** | |

### The three that actually change a signature

| | Count | Why it matters |
|---|---:|---|
| **ADDRESS** | 66 | `int64` addresses defeat leak checking, escape analysis, and Z3 pointer reasoning |
| **STATUS** | 102 | double-encodes failure — once in the `int64`, once in `Result.error` |
| **PREDICATE** | 5 | a truth value returned as an integer permits arithmetic on it |

**COUNT (227) and NUMERIC (65) need no signature change at all** — together 48% of
the surface. For COUNT the `-1` simply moves into `Result.error`; the `int64`
success value stays. That is a much smaller job than D-012's "514 of ~560
signatures" first suggested.


---

## ADDRESS — 66 functions

**Target:** `wild int8->` or a typed pointer

Returns an allocation, or a pointer into an existing buffer. **This is the category that matters** — an `int64` address is invisible to leak checking, escape analysis, and Z3 pointer reasoning (D-012).


**`fs/path.npk`**
- `path_basename`
- `path_dirname`
- `path_ext`
- `path_join`
- `path_normalize`
- `path_realpath`
- `path_strip_trailing_slash`

**`io/bio/file.npk`**
- `bio_alloc_buf`
- `bio_alloc_file`

**`io/bio/strerror.npk`**
- `strerror`
- `strerror_r`

**`io/bio/tmpfile.npk`**
- `mkdtemp`
- `tmpnam`

**`mem/alloc.npk`**
- `libn_mem_malloc`
- `libn_slab_alloc`
- `libn_slab_alloc_zero`
- `mem_alloc_guard_pages`
- `mem_alloc_pages`
- `mem_calloc`
- `mem_free_guard_pages`
- `mem_free_pages`
- `mem_malloc_map_size`
- `mem_malloc_user_size`
- `mem_memdup`
- `mem_realloc`
- `slab_free`
- `slab_realloc`

**`mem/memcpy.npk`**
- `mem_memcpy`
- `mem_memmove`
- `mem_mempcpy`

**`mem/memset.npk`**
- `mem_memset`

**`mem/memutil.npk`**
- `mem_align_ptr`
- `mem_memchr`
- `mem_memmem`
- `mem_memrchr`

**`mem/mmap.npk`**
- `mem_mmap_raw`
- `mem_mremap_raw`

**`proc/env.npk`**
- `getenv`

**`proc/exec.npk`**
- `proc_getenv_from`

**`str/strbuf.npk`**
- `strbuf_finish`
- `strbuf_new`
- `strbuf_ptr`
- `strbuf_to_str`

**`str/strchr.npk`**
- `str_strcasestr`
- `str_strchr`
- `str_strchrnul`
- `str_strpbrk`
- `str_strrchr`
- `str_strstr`

**`str/strconv.npk`**
- `str_itoa`
- `str_itoa_hex`
- `str_utoa`

**`str/strcpy.npk`**
- `str_stpcpy`
- `str_stpncpy`
- `str_strcpy`
- `str_strdup`
- `str_strncpy`
- `str_strndup`

**`str/strtok.npk`**
- `str_strsep`
- `str_strtok_r`

**`str/strview.npk`**
- `strview_ptr`
- `strview_to_str`
- `strview_to_strbuf`

**`syscall/errno.npk`**
- `err_str`

**`syscall/syscall.npk`**
- `libn_mmap`
- `libn_mremap`

---

## STATUS — 102 functions

**Target:** `NIL`

The `int64` carries no information beyond success/failure. Returning `NIL` means `Result<NIL>` — failure lives in `Result.error`, and callers use the same `?` / `?!` / `drop` vocabulary as everywhere else. This is the **double-encoding** case D-012 flagged.


**`fs/stat.npk`**
- `libn_fstat`
- `libn_lstat`
- `libn_stat`

**`io/bio/file.npk`**
- `bio_flush_write_buf`

**`io/bio/fio.npk`**
- `fflush`

**`io/bio/fopen.npk`**
- `fclose`

**`io/bio/fseek.npk`**
- `fgetpos`
- `fsetpos`

**`io/bio/fstate.npk`**
- `setvbuf`

**`io/bio/stdfiles.npk`**
- `bio_flush_all`
- `bio_flush_stderr`
- `bio_flush_stdout`

**`io/dup.npk`**
- `io_redirect`

**`io/fcntl.npk`**
- `io_clear_append`
- `io_set_append`
- `io_set_blocking`
- `io_set_cloexec`
- `io_set_nonblocking`
- `io_set_status_flags`

**`io/open.npk`**
- `io_close`
- `io_close_nointr`

**`io/pipe.npk`**
- `io_pipe`
- `io_pipe_nonblock`
- `io_pipe_raw`
- `io_pipe_read_fd`
- `io_pipe_write_fd`
- `io_socketpair_pipe`

**`io/printf.npk`**
- `io_perror`

**`io/seek.npk`**
- `io_rewind`

**`mem/alloc.npk`**
- `mem_free`

**`mem/mmap.npk`**
- `mem_madvise_raw`
- `mem_mprotect_raw`
- `mem_msync_raw`
- `mem_munmap_raw`

**`proc/env.npk`**
- `clearenv`
- `putenv`
- `setenv`
- `unsetenv`

**`proc/exec.npk`**
- `execl0`
- `execl1`
- `execl2`
- `execl3`
- `execl4`
- `execl5`
- `execl6`
- `execl7`
- `execl8`
- `execlp1`
- `execlp2`
- `execlp3`
- `execlp4`
- `execlp5`
- `execlp6`
- `execlp7`
- `execlp8`
- `execv`
- `execve`
- `execvp`
- `execvpe`

**`proc/exit.npk`**
- `at_quick_exit`
- `atexit`

**`proc/fork.npk`**
- `setpgid`
- `setsid`

**`proc/signal.npk`**
- `kill`
- `killpg`
- `raise`
- `sig_default`
- `sig_ignore`
- `sigaction`
- `sigaddset`
- `sigdelset`
- `sigemptyset`
- `sigfillset`
- `signal`
- `sigprocmask`

**`proc/wait.npk`**
- `proc_wait_all`

**`syscall/syscall.npk`**
- `libn_arch_prctl`
- `libn_chdir`
- `libn_clock_getres`
- `libn_clock_gettime`
- `libn_close`
- `libn_fchdir`
- `libn_kill`
- `libn_madvise`
- `libn_mprotect`
- `libn_mprotect_exec`
- `libn_msync`
- `libn_munmap`
- `libn_nanosleep`
- `libn_nanosleep_retry`
- `libn_pipe2`
- `libn_sched_yield`
- `libn_set_tid_address`
- `libn_statx`
- `libn_tgkill`

**`time/clock.npk`**
- `clock_getres`
- `clock_gettime`
- `clock_nanosleep`
- `clock_settime`

**`time/sleep.npk`**
- `nanosleep`

**`time/timeofday.npk`**
- `gettimeofday`
- `settimeofday`

---

## PREDICATE — 5 functions

**Target:** `bool`

Returns a yes/no answer as `int64`. Should be `bool` — D-005: a truth value is not an integer, and `bool` permits no arithmetic.


**`fs/path.npk`**
- `path_has_trailing_slash`
- `path_is_abs`

**`io/bio/fstate.npk`**
- `feof`
- `ferror`

**`io/bio/stdfiles.npk`**
- `isatty`

---

## HANDLE — 4 functions

**Target:** `FILE->`

Returns a `FILE` handle. Should be a typed pointer, not an integer.


**`io/bio/fopen.npk`**
- `fdopen`
- `fopen`
- `freopen`

**`io/bio/tmpfile.npk`**
- `tmpfile`

---

## FD — 19 functions

**Target:** `int32` — file descriptor

A kernel handle. Same argument as PID: semantically distinct from a number, and a candidate for a distinct type.


**`io/bio/fopen.npk`**
- `fileno`

**`io/bio/tmpfile.npk`**
- `mkostemp`
- `mkstemp`

**`io/dup.npk`**
- `io_dup`
- `io_dup2`
- `io_dup3`
- `io_dup_cloexec`

**`io/fcntl.npk`**
- `io_dupfd`
- `io_dupfd_cloexec`

**`io/open.npk`**
- `io_creat`
- `io_open`
- `io_open_inheritable`
- `io_open_tmp`
- `io_openat`

**`syscall/syscall.npk`**
- `libn_dup`
- `libn_dup2`
- `libn_dup3`
- `libn_open`
- `libn_openat`

---

## PID — 26 functions

**Target:** `int32` — pid / uid / gid

A process or user identifier. Narrower than `int64` and semantically distinct from a count — a candidate for a distinct type rather than a bare integer (D-005).


**`proc/fork.npk`**
- `fork`
- `getegid`
- `geteuid`
- `getgid`
- `getpgid`
- `getpgrp`
- `getpid`
- `getppid`
- `getsid`
- `getuid`
- `vfork`

**`proc/wait.npk`**
- `proc_run`
- `wait`
- `wait3`
- `wait_exit_code`
- `wait_stopsig`
- `wait_termsig`
- `waitpid`

**`syscall/syscall.npk`**
- `libn_fork`
- `libn_getegid`
- `libn_geteuid`
- `libn_getgid`
- `libn_getpid`
- `libn_getppid`
- `libn_gettid`
- `libn_getuid`

---

## COUNT — 227 functions

**Target:** `int64` — count, with failure moving to `Result.error`

POSIX-shaped "returns a count, or `-1` on failure". Under the universal `Result<T>` rule the count stays as the success value and the `-1` becomes `Result.error`. **The signature does not change**; only the error path moves.


**`io/bio/fchar.npk`**
- `fgetc`
- `fputc`
- `getc`
- `putc`
- `ungetc`

**`io/bio/file.npk`**
- `bio_discard_read_buf`
- `bio_parse_mode`
- `bio_refill_read_buf`

**`io/bio/fio.npk`**
- `fread`
- `fwrite`

**`io/bio/fprintf.npk`**
- `asprintf0`
- `asprintf1`
- `asprintf2`
- `asprintf3`
- `asprintf4`
- `asprintf5`
- `asprintf6`
- `asprintf7`
- `asprintf8`
- `eprintf0`
- `eprintf1`
- `eprintf2`
- `eprintf3`
- `eprintf4`
- `eprintf5`
- `eprintf6`
- `eprintf7`
- `eprintf8`
- `fprintf0`
- `fprintf1`
- `fprintf2`
- `fprintf3`
- `fprintf4`
- `fprintf5`
- `fprintf6`
- `fprintf7`
- `fprintf8`
- `printf0`
- `printf1`
- `printf2`
- `printf3`
- `printf4`
- `printf5`
- `printf6`
- `printf7`
- `printf8`

**`io/bio/fscanf.npk`**
- `fscanf0`
- `fscanf1`
- `fscanf2`
- `fscanf3`
- `fscanf4`
- `fscanf5`
- `fscanf6`
- `fscanf7`
- `fscanf8`
- `scanf0`
- `scanf1`
- `scanf2`
- `scanf3`
- `scanf4`
- `scanf5`
- `scanf6`
- `scanf7`
- `scanf8`
- `sscanf0`
- `sscanf1`
- `sscanf2`
- `sscanf3`
- `sscanf4`
- `sscanf5`
- `sscanf6`
- `sscanf7`
- `sscanf8`

**`io/bio/fseek.npk`**
- `fseek`
- `ftell`

**`io/bio/fstr.npk`**
- `fgets`
- `fputs`

**`io/bio/stdfiles.npk`**
- `getchar`
- `putchar`
- `puts`

**`io/bio/tmpfile.npk`**
- `tmpfile_get_entropy`

**`io/fcntl.npk`**
- `io_fcntl`
- `io_fcntl_add_seals`
- `io_fcntl_get_seals`
- `io_fcntl_getown`
- `io_fcntl_ofd_getlk`
- `io_fcntl_ofd_setlk`
- `io_fcntl_ofd_setlkw`
- `io_fcntl_setown`

**`io/printf.npk`**
- `io_dprintf0`
- `io_dprintf1`
- `io_dprintf2`
- `io_dprintf3`
- `io_dprintf4`
- `io_dprintf5`
- `io_dprintf6`
- `io_dprintf7`
- `io_dprintf8`
- `io_eprintf0`
- `io_eprintf1`
- `io_eprintf2`
- `io_eprintf3`
- `io_eprintf4`
- `io_eprintf5`
- `io_eprintf6`
- `io_eprintf7`
- `io_eprintf8`
- `io_fprintf0`
- `io_fprintf1`
- `io_fprintf2`
- `io_fprintf3`
- `io_fprintf4`
- `io_fprintf5`
- `io_fprintf6`
- `io_fprintf7`
- `io_fprintf8`
- `io_printf0`
- `io_printf1`
- `io_printf2`
- `io_printf3`
- `io_printf4`
- `io_printf5`
- `io_printf6`
- `io_printf7`
- `io_printf8`
- `io_sprintf0`
- `io_sprintf1`
- `io_sprintf2`
- `io_sprintf3`
- `io_sprintf4`
- `io_sprintf5`
- `io_sprintf6`
- `io_sprintf7`
- `io_sprintf8`

**`io/read.npk`**
- `io_read_char`
- `io_read_exact`
- `io_read_line`
- `io_read_n`
- `io_read_some`
- `io_read_until`

**`io/seek.npk`**
- `io_lseek`
- `io_seek`
- `io_seek_cur`
- `io_seek_end`
- `io_tell`

**`io/write.npk`**
- `io_eprint`
- `io_eprintln`
- `io_print`
- `io_println`
- `io_write_char`
- `io_write_cstr_n`
- `io_write_n`
- `io_write_newline`
- `io_write_nonblock`
- `io_write_str`
- `io_write_strln`

**`str/strbuf.npk`**
- `strbuf_appendf0`
- `strbuf_appendf1`
- `strbuf_appendf2`
- `strbuf_appendf3`
- `strbuf_appendf4`
- `strbuf_appendf5`
- `strbuf_appendf6`
- `strbuf_appendf7`
- `strbuf_appendf8`
- `strbuf_write_to_fd`

**`str/strconv.npk`**
- `str_atoi`
- `str_parse_i64`
- `str_parse_i64_base`
- `str_parse_u64`
- `str_strtod_q32`
- `str_strtol`
- `str_strtoul`

**`str/strfmt.npk`**
- `str_format_args`
- `str_snprintf0`
- `str_snprintf1`
- `str_snprintf2`
- `str_snprintf3`
- `str_snprintf4`
- `str_snprintf5`
- `str_snprintf6`
- `str_snprintf7`
- `str_snprintf8`

**`str/strlcpy.npk`**
- `str_strlcat`
- `str_strlcpy`
- `str_strlcpy_chk`
- `str_strscpy`
- `str_strscpy_pad`

**`str/strtok.npk`**
- `str_split_fields`
- `str_split_lines`

**`str/strview.npk`**
- `strview_copy_to`
- `strview_copy_to_z`
- `strview_find`
- `strview_find_char`
- `strview_find_str`
- `strview_parse_i64`
- `strview_parse_u64`
- `strview_rfind_char`
- `strview_write_to_fd`
- `strview_write_to_fp`

**`syscall/syscall.npk`**
- `libn_fcntl`
- `libn_futex_wait`
- `libn_futex_wake`
- `libn_getcwd`
- `libn_getdents64`
- `libn_getrandom`
- `libn_ioctl`
- `libn_lseek`
- `libn_poll`
- `libn_pread`
- `libn_pwrite`
- `libn_read`
- `libn_read_retry`
- `libn_write`
- `libn_write_all`
- `sys1`
- `sys2`
- `sys3`
- `sys4`
- `sys5`
- `sys_full`
- `sys_full1`
- `sys_full2`
- `sys_full3`
- `sys_full4`
- `sys_full5`
- `sys_safe`

---

## NUMERIC — 65 functions

**Target:** `int64` — unchanged

A genuine number: length, count, comparison result, alignment, error code. Cannot fail. No change.


**`fs/path.npk`**
- `path_depth`

**`io/fcntl.npk`**
- `io_get_status_flags`

**`io/seek.npk`**
- `io_file_size`

**`math/math.npk`**
- `math_abs_i64`
- `math_align_down`
- `math_align_up`
- `math_clamp_i64`
- `math_clz`
- `math_ctz`
- `math_div_ceil_i64`
- `math_div_floor_i64`
- `math_div_round_i64`
- `math_gcd`
- `math_isqrt`
- `math_isqrt_ceil`
- `math_lcm`
- `math_log2_ceil`
- `math_log2_floor`
- `math_max_i64`
- `math_max_u64`
- `math_min_i64`
- `math_min_u64`
- `math_mod_pos_i64`
- `math_next_pow2`
- `math_parity`
- `math_popcount`
- `math_pow_i64`
- `math_pow_u64`
- `math_prev_pow2`
- `math_reverse_bits`
- `math_rol`
- `math_ror`
- `math_sat_add_i64`
- `math_sat_add_u64`
- `math_sat_mul_i64`
- `math_sat_mul_u64`
- `math_sat_sub_i64`
- `math_sat_sub_u64`
- `math_sign_i64`

**`mem/memutil.npk`**
- `mem_memcmp`
- `mem_memcmp_ct`

**`mem/mmap.npk`**
- `page_align_down`
- `page_align_up`
- `size_align_up`

**`str/strbuf.npk`**
- `strbuf_cap`
- `strbuf_len`
- `strbuf_new_cap`

**`str/strchr.npk`**
- `str_strcspn`
- `str_strspn`

**`str/strcmp.npk`**
- `str_strcasecmp`
- `str_strcmp`
- `str_strncasecmp`
- `str_strncmp`

**`str/strlen.npk`**
- `str_strlen`
- `str_strlen_safe`
- `str_strnlen`

**`str/strview.npk`**
- `strview_byte_at`
- `strview_cmp`
- `strview_cmp_str`
- `strview_first`
- `strview_last`
- `strview_len`
- `strviewiter_remaining`

**`syscall/errno.npk`**
- `err_from_syscall`
- `libn_errno_get`

---

## KEEP — 94 functions

**Target:** unchanged

Already returns something other than `int64` — `NIL`, `bool`, or a typed value. No classification needed.


**`failsafe.npk`**
- `failsafe`

**`io/bio/file.npk`**
- `bio_free_buf`
- `bio_free_file`

**`io/bio/fseek.npk`**
- `rewind`

**`io/bio/fstate.npk`**
- `clearerr`
- `setbuf`
- `setlinebuf`

**`io/bio/stdfiles.npk`**
- `bio_ensure_std_init`

**`io/bio/strerror.npk`**
- `perror`

**`io/fcntl.npk`**
- `io_get_cloexec`
- `io_is_nonblocking`

**`math/math.npk`**
- `math_is_aligned`
- `math_is_pow2`

**`mem/memcpy.npk`**
- `__intrinsic_memcpy`

**`mem/memset.npk`**
- `mem_bzero`
- `mem_bzero_explicit`
- `mem_memset_explicit`
- `mem_memset_pattern4`
- `mem_poison`

**`mem/memutil.npk`**
- `mem_equal`
- `mem_is_aligned`
- `mem_is_zero`
- `mem_reverse`
- `mem_swap`

**`mem/mmap.npk`**
- `is_page_aligned`

**`proc/env.npk`**
- `proc_env_init`

**`proc/exit.npk`**
- `_exit`
- `abort`
- `libn_exit`
- `quick_exit`

**`proc/signal.npk`**
- `sigismember`

**`proc/wait.npk`**
- `wait_continued`
- `wait_coredump`
- `wait_exited`
- `wait_signaled`
- `wait_stopped`

**`str/strbuf.npk`**
- `strbuf_append_bytes`
- `strbuf_append_char`
- `strbuf_append_repeat`
- `strbuf_append_str`
- `strbuf_append_strbuf`
- `strbuf_free`
- `strbuf_is_empty`
- `strbuf_reset`
- `strbuf_strip_trailing_newline`
- `strbuf_truncate`

**`str/strchr.npk`**
- `charset_build`
- `charset_test`

**`str/strcmp.npk`**
- `str_casecmp_prefix`
- `str_strcmp_prefix`
- `to_lower_ascii`

**`str/strconv.npk`**
- `str_to_fix256`
- `str_to_tfp64`

**`str/strtok.npk`**
- `str_field_equals`

**`str/strview.npk`**
- `strview_contains`
- `strview_contains_char`
- `strview_contains_str`
- `strview_drop`
- `strview_drop_prefix`
- `strview_drop_suffix`
- `strview_empty`
- `strview_ends_with`
- `strview_ends_with_str`
- `strview_eq`
- `strview_eq_str`
- `strview_is_empty`
- `strview_next_line`
- `strview_next_token`
- `strview_of`
- `strview_of_bytes`
- `strview_of_strbuf`
- `strview_slice`
- `strview_split_once`
- `strview_split_once_char`
- `strview_starts_with`
- `strview_starts_with_str`
- `strview_strip_prefix_str`
- `strview_strip_suffix_str`
- `strview_substr`
- `strview_take`
- `strview_trim`
- `strview_trim_left`
- `strview_trim_right`
- `strviewiter_next_line`
- `strviewiter_next_token`
- `strviewiter_of`
- `strviewiter_of_str`

**`syscall/errno.npk`**
- `errno_clear`
- `errno_is_fatal`
- `errno_is_retriable`
- `errno_set`
- `libn_errno_set`

**`syscall/syscall.npk`**
- `libn_exit_group`
- `libn_exit_thread`

---

## Review notes

### Where the heuristics could be wrong

The classification is name-driven, so a handful warrant a second look before the
edits are applied mechanically:

- **`getenv`, `putenv`** — classified ADDRESS. `getenv` returns a pointer into the
  environment block, so that is right; `putenv` is arguably STATUS.
- **`fgetc` / `getc` / `getchar`** — classified COUNT. They return a character *or*
  `EOF`, which under `Result<T>` is cleaner as `Result<char8>` with EOF as the
  error. Worth treating as a small category of its own.
- **`str_strcmp` family** — classified NUMERIC. Correct, but the return is a
  three-way comparison, so `<=>`'s `int32` result may be the better type.
- **`mem_memcmp_ct`** — the `_ct` suffix means constant-time. It must **not** be
  rewritten in a way that introduces early return, which would destroy the
  timing property. Flag for manual handling.

### Two categories worth a type rather than an integer

`FD` (19) and `PID` (26) are both classified as `int32` above, but by D-005's
principle — a semantic concept is not its representation — both are candidates
for distinct types. A file descriptor supports `close` and `read`; it does not
support multiplication. The same argument that separated `char8` from `uint8`
applies.

Deferred rather than decided: it is a language-surface question, not a porting
one, and 45 signatures is a small enough surface to revisit later.

### Not covered here

**Parameters.** This ledger classifies *return* types. The same treatment is
needed for parameters — `int64:ptr`, `int64:dst`, `int64:src`, `int64:buf` are
addresses and become typed pointers, while `int64:n`, `int64:size`, `int64:len`
stay numeric. That pass should run over the same 608 signatures before editing
begins.

**Conformance beyond D-012.** Separate from the signature work, and measured
elsewhere: `cast_unchecked<T>` → `=>!` (D-021), integer→pointer construction →
`#wild_ptr<T>` in `wild` context (D-019), `?!` arity (D-009), and the `fix256_t`
/ `tfp64_t` return types found in this scan, which carry both the obsolete `fix`
name (D-036) and a C-style `_t` suffix.
