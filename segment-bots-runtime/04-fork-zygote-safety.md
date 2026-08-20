# Is a fork-based zygote safe for a 24/7 unattended system?

Researched 2026-08-20 by a `sonnet` agent (Rule 1) from POSIX, glibc/man7, CPython
source and What's New, AOSP, Chromium, Gunicorn, and a live numpy bug. Report only.
Reproducer re-run locally — result below differs from the agent's box.

## Verdict

**A hand-rolled fork zygote is not a sound long-term foundation** — not because
fork is scary in the abstract, but because **the safety condition (single-threaded
at fork time) cannot be verified once and trusted forever.** It must hold at every
fork, for the life of the process, against every library any dependency pulls in.

Recommendation: use `multiprocessing`'s **`forkserver`**, which is CPython's own
default on POSIX from 3.14, and which moves the single-threaded invariant into a
component whose only job is to uphold it.

## The hazard, precisely

POSIX.1-2024 `fork()` rationale:

> "A process shall be created with a single thread. If a multi-threaded process
> calls `fork()`, the new process shall contain a replica of the calling thread and
> its entire address space, possibly including the states of mutexes and other
> resources. Consequently, to avoid errors, the child process may only execute
> async-signal-safe operations until such time as one of the exec functions is called."

Mechanically: fork duplicates the address space but only the *calling* thread. Every
mutex another thread held at that instant is copied **locked**, and the thread that
would unlock it does not exist in the child. Touching such a lock **hangs forever —
it does not crash.**

`pthread_atfork(3)`: *"other threads may have locked mutexes that are visible in the
user-space memory duplicated in the child. Such mutexes would never be unlocked,
since the threads that placed the locks are not duplicated in the child."*

Real Python failure modes: import-lock deadlock; **`logging` handler lock deadlock**
(the most commonly reported real-world case); glibc malloc arena lock — which
CPython hits almost immediately, since object allocation calls malloc; and BLAS
thread-pool deadlock (below).

## CPython's own position — version-pinned

**3.12** — `os.fork()` raises `DeprecationWarning` when threads are detected. From
`whatsnew/3.12`:

> "Even in code that appears to work, it has never been safe to mix threading with
> `os.fork()` on POSIX platforms. The CPython runtime itself has always made API
> calls that are not safe for use in the child process when threads existed in the
> parent (such as malloc and free)."

Exact warning text from `Modules/posixmodule.c`, `warn_about_fork_with_threads`:

```
"This process (pid=%d) is multi-threaded, use of %s() may lead to deadlocks in the child."
```

Detection reads **field 20 of `/proc/self/stat`** — the kernel's real thread count,
so it catches **native C threads such as OpenBLAS workers**, not only `threading`
objects; it falls back to `threading._active` counting only if that read fails.

**3.14** — `forkserver` becomes the default POSIX start method (gh-84559,
Gregory P. Smith). From the multiprocessing docs:

> "fork … Changed in version 3.14: This is no longer the default start method on any
> platform. Code that requires fork must explicitly specify that via `get_context()`
> or `set_start_method()`."

Fork is **not removed** — it is demoted. The core team did not ban it; they made it
opt-in and shipped a runtime detector for this exact bug class.

## The live bug — and what it did on THIS box

numpy/numpy#30092, *"BUG: Binary Builds Deadlock due to OpenBLAS threading issue with
fork"* — open, reproducing on Python 3.13/3.14 with numpy ≥2.3.2 and OpenBLAS 0.3.30:

```python
import numpy as np, os
A = np.random.randn(216, 216)
np.linalg.inv(A)          # first BLAS call spins up OpenBLAS's thread pool
if (pid := os.fork()) != 0:
    np.linalg.inv(A)      # deadlocks in the PARENT
    os.waitpid(pid, 0)
```

Root cause is worse than the generic hazard: OpenBLAS registers a `pthread_atfork()`
handler (`blas_thread_shutdown_`) meant to tear the pool down safely — and **that
handler itself deadlocks** on a `server_lock` mutex, a regression in OpenBLAS 0.3.30
(OpenMathLib/OpenBLAS#5170). It hangs the **parent**, and worsens with higher core
counts — it would pass on a small test box and hang in production on a big one.

**Re-run locally 2026-08-20: this box ships scipy-openblas 0.3.34 and the reproducer
completes, `NO DEADLOCK`, with BLAS threads both default and pinned to 1.** The
regression was introduced by one dependency bump and fixed by another. That is the
argument for pinning exact versions (RL-065), not evidence the hazard is gone.

**Critical timing fact, verified by that bug:** OpenBLAS worker threads are spawned
**lazily at first computational use, not at `import numpy`.** So `import numpy` in a
warm parent is safe; running any BLAS-backed call before forking is not.

## What `register_at_fork` can and cannot do

`os.register_at_fork` gives `before` / `after_in_parent` / `after_in_child` hooks —
the `pthread_atfork` shape. Docs: *"fork() calls made by third-party C code may not
call those functions"*, and *"There is no way to unregister a function."*

It fixes locks **you** wrote. It cannot fix a library whose own atfork handler is
buggy — which is exactly how numpy#30092 happens. man7 is blunt about the limit:

> "The original intention of `pthread_atfork()` was to allow the child process to be
> returned to a consistent state... **In practice, this task is generally too
> difficult to be practicable.**"

And POSIX: *"there is no way for an implementation to determine whether the fork
handlers established by `pthread_atfork()` are async-signal-safe... leading to a
deadlock condition."*

## How real systems do it — the engineering rule, not an API

- **Android Zygote** — `ZygoteInit.preload()` loads thousands of classes, shared
  libs, graphics driver and resources **before** accepting fork requests. Contract in
  the fork API's own javadoc: setuid/setgid happen "after fork()ing and **before
  spawning any threads**". JIT and other threads are held off during preload
  specifically to keep the zygote single-threaded at fork.
- **Chromium zygote** — preloads loader relocations (~60 ms/GHz, ~6 MB/process
  amortised), ICU, NSS, the V8 snapshot, the namespace sandbox. From
  `zygote_linux.cc`: *"Avoid taking the ~10-50ms jank penalty that a multithreaded
  process has to take when expanding its file descriptor table... by allocating an fd
  table with at least 512 entries right away, **while we're still single-threaded**."*
- **Gunicorn** — `spawn_worker()` calls plain `os.fork()` with **zero** fork-safety
  ceremony. It is safe because the arbiter is an event/signal loop that stays
  single-threaded by design. `preload_app` defaults to **False** — the safer posture.
- **uWSGI `lazy-apps`** — loads the app after fork, "slower but safer", recommended
  when there is "potential bad global state".

**The common rule: the forking process is kept single-threaded by architectural
discipline, not by a library trick.** That is the pattern — a constraint on what the
warm parent may do, not a magic API.

## Safe-by-construction checklist

1. **Single-threaded at the instant of fork — verifiably.** Check field 20 of
   `/proc/self/stat` immediately before every fork; refuse and alert if > 1. Turns a
   rare unreproducible hang into a loud logged refusal (Rule 0 / Rule 8).
2. No `logging` handlers with locks live across the fork boundary.
3. No thread pools or background threads in the warm parent — including ones you did
   not start: BLAS pthreads, asyncio loop threads, `ThreadPoolExecutor`,
   connection-pool keepalives.
4. No CUDA/BLAS initialisation that spawns threads before fork.
5. No open sockets before fork; children connect after forking.
6. **Set `OPENBLAS_NUM_THREADS=1` and `OMP_NUM_THREADS=1` before importing numpy**
   (also `MKL_NUM_THREADS` if MKL-backed — **UNVERIFIED** for our wheel but standard).
   Treat "numpy-warm" as *imported but no BLAS call executed* for the strongest
   guarantee.

## Alternatives

- **`posix_spawn`** — "a combined `fork(2)` and `exec(3)`". Always execs. Cannot give
  a warm-numpy child, so it collapses to the measured 158 ms cold-spawn cost.
- **`vfork`** — parent suspended until child execs; child shares the parent's stack
  and must not return or call `exit`. Fork-then-exec optimisation only. **Unusable**
  for long-lived children running arbitrary Python.
- **`forkserver`** — the real answer. Docs: *"The fork server process is single
  threaded unless system libraries or preloaded imports spawn threads as a
  side-effect so it is generally safe for it to use `os.fork()`."*
  `set_forkserver_preload(['numpy'])` preserves the warm-start and COW benefit —
  *"must be called before the forkserver process has been launched"*.

**Measured on this box 2026-08-20 (Python 3.14.6 free-threaded, numpy 2.5.2):**

| | raw fork from warm zygote | forkserver + preload |
|---|---|---|
| start | 3.0 ms | **2.78 ms** |
| PSS per idle child | **0.76 MB** | 3.07 MB |
| 300 parts | 228 MB | 921 MB |
| default start method on 3.14t | — | **already `forkserver`** |
| child inherits numpy import | yes | **yes, verified** |

**Constraint found empirically, not in the agent's report:** forkserver pickles the
target **by qualified name**, so a nested function fails with
`AttributeError: module '__mp_main__' has no attribute 'idle'`. **Every part's entry
point must be module-level.** Raw fork has no such restriction. Also: the forkserver
process itself stays resident holding the preloaded numpy (~30 MB) — correct, but it
means "all parts off" is not literally zero RSS.

## UNVERIFIED

- Exact CPython version where `/proc/self/stat` OS-level thread counting replaced
  threading-module counting — source read was `main`, possibly ahead of 3.12.0.
- **Free-threaded build's interaction with fork: no documented change.** The
  free-threading HOWTO does not mention fork at all. The agent's own reasoned
  caution, explicitly flagged as inference not citation: removing the GIL lets more
  threads be genuinely inside library code holding locks at any instant, which may
  *widen* the fork window. Treat as an open risk, not a mitigated one.
- `MKL_NUM_THREADS` relevance to our wheel (we ship scipy-openblas, not MKL).

## Sources

- https://pubs.opengroup.org/onlinepubs/9699919799/functions/fork.html
- https://man7.org/linux/man-pages/man2/fork.2.html · .../man3/pthread_atfork.3.html
- https://man7.org/linux/man-pages/man3/posix_spawn.3.html · .../man2/vfork.2.html
- https://docs.python.org/3/whatsnew/3.12.html · https://docs.python.org/3/whatsnew/3.14.html
- https://docs.python.org/3/library/multiprocessing.html · .../os.html#os.register_at_fork
- CPython `Modules/posixmodule.c`, `warn_about_fork_with_threads` · gh-84559
- https://github.com/numpy/numpy/issues/30092 · OpenMathLib/OpenBLAS#5170
- AOSP `ZygoteInit.java`, `Zygote.java` · Chromium `docs/linux/zygote.md`, `content/zygote/zygote_linux.cc`
- https://github.com/benoitc/gunicorn (`arbiter.py`, `config.py`) · https://uwsgi-docs.readthedocs.io/en/latest/ThingsToKnow.html
