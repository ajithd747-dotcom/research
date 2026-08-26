# Does moving the scanner "to RAM" make it fast? Measured on this box.

**Researched 2026-08-26 by a `sonnet` subagent (Rule 1). Every number below was
measured on THIS machine (12 cores, CPython 3.14.4 standard build) in a scratch `uv`
venv — not estimated, not taken from a blog. No project files were touched.**

## The answer: the data is already in RAM. The win is LAYOUT, not residency.

1,500 symbols, 300-sample window, rolling z-score, one full sweep:

| Approach | Time/sweep | Speedup |
|---|---|---|
| A — dict of Python `RollingWindow` objects (deque + Python-level mean/std) — **what the detectors do today** | **157.1 ms** | 1x |
| B — SoA numpy ring buffer, full `mean()`/`std()` recompute each tick | **1.09 ms** | **144x** |
| C — SoA numpy, O(1) running sum / sum-of-squares (no window recompute) | **0.054 ms** | **2,922x** |

Both A and B/C are 100% in RAM already. Nothing moved between disk and memory. The
144-2,900x gap is cache locality, contiguous homogeneous dtype memory, and vectorised C
loops replacing per-object Python attribute access.

**"Move it to RAM" is the wrong framing — it was never anywhere else.** The correct
framing is "stop storing it as 1,500 Python objects and store it as one 2D array."

**And the honest scale answer: at 1,500 symbols a rolling z-score in plain numpy is
sub-millisecond. There is no wall here.** Approach C does 1,500 symbols in 54
microseconds. No case exists for numba, Cython, shared memory, or a columnar database to
make this computation faster. Plain SoA numpy is the entire fix, with zero new dependencies.

## Ring-buffer traps (both hit while writing the benchmark)

Canonical layout is `array[N_symbols, W]` with `wi = tick_count % W`. ~10 lines, normally
hand-rolled; there is no canonical library.

- **Wraparound.** `axis=1` reductions over the whole row are correct regardless of write
  position, because the *set* of values is what matters. But the moment you go to O(1)
  running sums you **must subtract the value being overwritten** (`ring[:, wi]`) before
  adding the new one, or the running sum silently accumulates every tick ever seen instead
  of a rolling window. **This is the most common bug in a hand-rolled ring buffer and it
  fails silently — the numbers stay plausible and are wrong.**
- **Partial windows.** A ring initialised with zeros computes a z-score against fake data
  until it fills. Either keep a per-symbol fill counter and use `n = min(ticks_seen, W)`,
  or initialise to NaN and use `np.nanmean`/`np.nanstd` (2-4x slower, UNVERIFIED multiple).
  The fill-counter approach matches this codebase's existing checkpoint idiom better.

## Sharing across processes

**`multiprocessing.shared_memory`** — stdlib, no wheel question. Verified live on CPython
3.14.4: a 3.6 MB `(1500, 300)` float64 array created, attached from a second handle by
name, **144.4 microseconds** for a full-array `sum()` read.

**Correction to the framing:** `shared_memory` on Linux backs onto **`/dev/shm`, not
`/tmp`**. Confirmed by `mount`: both are separate tmpfs mounts, 15 GB each on this box,
`/tmp` already holding 709 MB. So "memmap under /tmp" and "shared_memory" are two
different RAM-backed mechanisms on two different tmpfs mounts, and **neither writes to
disk**. A `/tmp` memmap is a legitimate footgun: nothing distinguishes "a file that
survives a reboot" from "RAM that a reboot erases." Prefer the `shared_memory` API — it
is self-documenting about being ephemeral.

**What breaks with a writer and N readers:** no locking, no atomicity across the array, no
versioning. Aligned 8-byte float64 writes are atomic on x86-64, so single-cell tears do not
happen — but a **row** update is not atomic, so a reader sweeping the whole array while a
writer updates 1,500 rows can see a half-updated snapshot. Mitigation: single writer,
readers copy-then-compute (144 microseconds, cheap enough every tick), or double-buffer
with an atomic index flip. UNVERIFIED — no concurrent stress harness was actually built.

**Arrow Plasma is dead.** Removed in Arrow 12.0.0 (May 2023), apache/arrow#33243. Does not
exist in pyarrow 25.0.1. Do not design around it.

## cp314 wheel availability — resolved and INSTALLED live, not just file-listed

| Package | Version | cp314 path |
|---|---|---|
| numpy | 2.5.2 | `cp314-cp314-manylinux_2_27_x86_64` |
| pyarrow | 25.0.1 | `cp314-cp314-manylinux_2_28_x86_64` |
| duckdb | 1.5.5 | `cp314-cp314-manylinux_2_26_x86_64` |
| polars | 1.44.1 | metapackage (`py3-none-any`) pulling `polars-runtime-32`, which ships `cp310-abi3` — installs on 3.14 |
| **numba** | 0.67.0 | `cp314-cp314-manylinux2014_x86_64` — **JIT compiled and ran, zero system compiler used** |
| **llvmlite** | 0.49.0 | `cp314-cp314-manylinux2014_x86_64`, **bundles prebuilt LLVM 22 in the wheel** |

## Polars / DuckDB for streaming rolling computation — measured, and the answer is no

DuckDB: **1.26 ms per single-row INSERT**; 1,500 rows via `executemany` still took
**2.5 seconds**. Compare 0.054 ms for numpy's *entire 1,500-symbol sweep* — DuckDB's
per-row insert alone is ~23x slower than numpy's whole answer. Its `GROUP BY` over 1,500
rows was fast (4.6 ms) — that is the batch strength — but ingestion is the bottleneck.

**Bluntly: DuckDB and Polars are the wrong tool for updating a rolling window once a
second per symbol.** They are the right tool for periodic columnar dumps of the tape and
ad-hoc analytical/backtest scans — a different access pattern from a live detector's hot
path. Polars' streaming append was not benchmarked directly (UNVERIFIED) but has the same
architectural mismatch.

## numba / Cython with no compiler

- **numba works, verified live.** First call 0.96s (JIT compile), second 5.2 microseconds.
  `llvmlite` ships LLVM *inside the wheel* — it is not bindings needing a system LLVM.
  The missing-compiler constraint does not block it.
- **Cython does not and should not be attempted.** It emits `.c` and needs a real C
  compiler to build a `.so`. That step cannot be skipped for your own code (unlike numba,
  a JIT that never touches a C compiler).
- **Neither is needed.** numba pays off for Python loops that *cannot* be vectorised.
  A rolling z-score is not one.

## What this recommends

Rewrite the detectors' per-symbol dict-of-objects into 2D numpy SoA ring buffers with
O(1) running stats and a fill counter, **inside the same process they already run in**.
~3,000x reduction in sweep cost, no new library, no wheel risk, no cross-process sharing.

Cross-process sharing only matters if a *second* part must read the same rolling state
without recomputing it — and under T-4 (a part names data, never other parts) the more
idiomatic fit here is probably still "each part computes its own state from the bus."

## UNVERIFIED / low-confidence

1. NaN-aware reduction overhead (2-4x) — not benchmarked on this box.
2. Concurrent writer + N readers torn-row race — described from how the mechanism works,
   no multi-process stress test was run to observe a failure.
3. Polars lazy/streaming incremental append — not benchmarked; DuckDB used as the analogous case.
4. The "10-100x numpy" blog citations are corroboration only; the load-bearing numbers are
   the ones measured here.
5. `polars-runtime-32` abi3 on 3.14 — install and import succeeded live; Polars' own test
   suite was not verified against 3.14 upstream.

Sources: apache/arrow#33243; Arrow 12.0.0 release notes; pypi.org/pypi/<pkg>/json fetched
raw with curl for numpy, numba, llvmlite, polars, polars-runtime-32, pyarrow, duckdb.
Benchmark scripts: scratchpad/{bench_soa,numba_smoke,shm_smoke,duckdb_smoke}.py
