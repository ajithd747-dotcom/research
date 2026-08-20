# The real numeric scaling ceiling on this box — and why the first measurement lied

Researched 2026-08-20 by a `sonnet` agent (Rule 1). Report only. The hardware
findings were re-verified locally with `lscpu` and the GCP metadata service.

## Verdict

**Mostly not worth optimising — with one cheap exception.** At megabyte-scale
intraday bar data the working set fits in cache; Polars' gigabyte-scale wins,
Numba's 1M-point break-even and hand-rolled cache blocking all answer a data-volume
problem this project does not have. The one thing worth doing is **pinning BLAS
thread counts**, which is mechanical and demonstrated.

## What this machine actually is — verified locally, not assumed

```
machine-type: e2-custom-12-30720          (GCP metadata service)
cpu-platform: AMD Rome / AMD EPYC 7B12
Socket(s): 1   Core(s) per socket: 6   Thread(s) per core: 2
  -> 12 logical CPUs = 6 PHYSICAL cores with SMT2
NUMA node(s): 1        cpu0 thread siblings: 0,6
L1d 192 KiB (6 inst)   L2 3 MiB (6 inst)   L3 32 MiB (2 instances of 16 MiB)
```

**This corrects a number used throughout the runtime design: there are 6 physical
cores, not 12.** "8 threads should give 8x" was never valid here. The honest
ceiling from core count alone is ~6x for compute-bound work, less for anything SMT-
or bandwidth-sensitive.

E2 is GCP's cost-optimised family: vCPU-to-physical-core mapping is **not** dedicated
the way N2D/C2 is, and performance is documented as more variable under load, so
there is host-level noisy-neighbour risk on top. (E2 tenancy internals beyond public
docs: **UNVERIFIED** — the hypervisor is not visible from inside.)

## The first measurement was oversubscription, not bandwidth

Agent's measurement on a comparable environment — 600×600 float64 matmul, **single
Python thread**, varying only one environment variable:

| `OPENBLAS_NUM_THREADS` | time/call |
|---|---|
| unset (default) | **1.96 ms** |
| `1` | **10.26 ms** |
| `6` (physical cores) | 2.27 ms |
| `12` (all logical) | 2.24 ms |

**5x difference from one env var, with no Python-level threading at all.** The
default already spawns a multithreaded BLAS call inside one Python thread. So a
"1 thread vs 8 threads" comparison without pinning is really *1 internally-parallel
call* versus *8 Python threads × N BLAS threads each*, oversubscribing 6 physical
cores — which produces exactly the observed collapse.

Confirmed locally on our own stack: pinning BLAS to 1 moved scaling from
**1.9x to 4.4x of 8**, and raised aggregate throughput ~1.5x.

### Exact control variables for the wheel we actually have

- `numpy.show_config()` here reports **scipy-openblas**, a bundled package-private
  OpenBLAS (agent saw 0.3.33.112.0; our venv reports 0.3.34.0.0).
- **PyPI OpenBLAS wheels are built with pthreads, not OpenMP** (conda-forge uses
  OpenMP; PyPI does not). So the authoritative lever is **`OPENBLAS_NUM_THREADS`**.
  `OMP_NUM_THREADS` is what numpy's docs mention generically but is imprecise for
  this wheel — a fallback, not the primary lever.
- Full set to pin **before `import numpy`**: `OPENBLAS_NUM_THREADS`,
  `MKL_NUM_THREADS`, `OMP_NUM_THREADS`, `VECLIB_MAXIMUM_THREADS`,
  `NUMEXPR_NUM_THREADS`.
- **`threadpoolctl` is not installed**, and `numpy.show_runtime()` prints a warning
  saying it cannot report BLAS thread state without it. **Until it is installed,
  in-process BLAS thread counts cannot be verified at all.**

### The correct experiment is a 2×2, not one line

| | `OPENBLAS_NUM_THREADS=1` | unset |
|---|---|---|
| 1 Python thread | true single-core baseline | BLAS-internal parallelism only |
| 8 Python threads | Python-thread parallelism, BLAS not competing | both layers fighting |

Discard the first call (warm caches), repeat, and choose the numerator deliberately.

## Roofline — and why the matmul benchmark was a trap

Arithmetic intensity = FLOPs ÷ bytes moved from DRAM (Williams, Waterman, Patterson,
CACM 2009). Ridge point on typical x86 server chips is ~5–15 FLOPs/byte for float64.

- **Rolling-window statistics are bandwidth-bound almost by construction** — low
  arithmetic intensity whether naive or incremental (Welford). Independent of scale.
- **600×600 matmul never touched DRAM.** Three float64 600×600 matrices ≈ **8.2 MiB**,
  which fits inside *one* of this chip's two 16 MiB L3 instances. It is **L3-bound,
  a third regime a two-line roofline misses.** L3 bandwidth is 5–10x DRAM on this
  class of chip — and that is also why threads stopped helping past ~6 (2.27 ms at 6
  threads vs 2.24 ms at 12: saturated the L3 port, more threads only add scheduling).

**Measuring achievable bandwidth without perf counters or root:** STREAM is just
wall-clock timing of large array copy/scale/add/triad sized well beyond cache — no
privilege needed. No compiler is available here, so the practical version is numpy
itself: allocate arrays well past 32 MiB (200+ MB is affordable on 30 GB), run
`c[:] = a + b` in a loop with `perf_counter()`, compute GB/s from known bytes moved.

## What helps at megabyte scale — honestly

| Technique | Verdict at our data size |
|---|---|
| **Polars vs pandas** | db-benchmark's 5–13x wins are measured at **0.5 / 5 / 50 GB**. At MB scale most of pandas' cost is fixed per-call Python overhead. **Adopt for correctness and ergonomics, not throughput** |
| **Numba** | Wins show at **1M+ points**; below that JIT compile time (hundreds of ms) can exceed the whole naive runtime. In a long-running bot it amortises — worth it **for a profiled hot inner loop only, with `cache=True`** |
| **Cython** | Ongoing build and iteration cost, nothing here hot enough. **Skip** |
| **numexpr** | Real modest win for chained elementwise expressions (avoids materialising intermediates) once operands exceed L1/L2. Needs `NUMEXPR_NUM_THREADS` pinned or it becomes another oversubscription source |
| **Arrow zero-copy** | An I/O/interop optimisation, not compute. Relevant at the ingestion/replay boundary only |
| **Cache blocking** | OpenBLAS already does it internally and will beat hand-rolled. The free win is keeping own loops over contiguous memory |

## Threads versus processes for numeric work

- numpy's C ufunc loops **already release the GIL** (`NPY_BEGIN_THREADS`), so on a
  normal build multiple Python threads already get real parallelism for array math.
  **Free-threading adds nothing there.** Documented exception: `dtype=object` arrays
  do not release the GIL.
- Free-threading helps the **Python-level glue** — bar loops, dict building,
  object-heavy pandas internals — which is common in a bot's non-vectorised control flow.
- Compatibility (py-free-threading.github.io/tracking): **numpy since 2.1.0, pandas
  since 2.2.3, both tested in CI.** Real, not aspirational.
- **Caveat that matters:** any C extension not ported silently re-enables the GIL for
  the whole process on import. **Check `sys._is_gil_enabled()` after all imports** —
  every time an exchange SDK is added.
- **Subinterpreters are unrelated to free-threading** (PEP 554/684 vs PEP 703). Our
  measured `numpy._core._multiarray_umath does not support loading in subinterpreters`
  is numpy lacking multi-phase init (PEP 489). **Free-threading does not fix it.**
- **Processes remain the safer default for CPU-bound numeric fan-out** — each gets its
  own isolated BLAS pool, so cross-thread oversubscription is impossible by construction.

## Bandwidth-bound classes — for the governor's part classification

Cap these at ~2–4 threads regardless of how many cores look idle, and **never
co-schedule two of them** — they divide one fixed pipe rather than adding throughput:

- elementwise array ops on data exceeding cache
- large array copies and dtype casts
- reductions (sum/mean/std/min/max) over large 1-D data
- **rolling-window statistics** — the workload this project runs most
- CSV/text parsing (byte-bound, not FLOP-bound)
- pandas groupby dominated by hashing and intermediate copies

**Single NUMA node, one memory controller:** every thread in every process on this VM
shares one bandwidth ceiling — and on E2 that ceiling may be shared with other tenants.

## UNVERIFIED

- The agent could not find our exact stack (Python 3.14.6t + numpy 2.5.2 +
  pandas 3.0.5) installed anywhere on the box, so its BLAS demonstration ran on
  numpy 2.5.1 under a standard GIL build. The mechanism is a numpy/OpenBLAS property,
  not a GIL-build property, so it transfers — and we reproduced the effect
  independently on our own venv (1.9x → 4.4x).
- GCP E2 scheduling and tenancy internals beyond public docs.
- `threadpoolctl` not installed, so no in-process cross-check of live thread counts.
- Williams/Waterman/Patterson roofline cited as standard methodology, not re-fetched.

## Sources

- https://docs.cloud.google.com/compute/docs/general-purpose-machines · .../machine-resource
- https://pythonspeed.com/articles/concurrency-control/
- https://pypackaging-native.github.io/key-issues/native-dependencies/blas_openmp/
- https://numpy.org/doc/stable/reference/global_state.html
- https://numpy.org/doc/2.4/reference/thread_safety.html · https://numpy.org/devdocs/reference/c-api/ufunc.html
- https://py-free-threading.github.io/tracking/
- https://duckdblabs.github.io/db-benchmark/
- https://pandas.pydata.org/docs/user_guide/enhancingperf.html
- https://www.cs.virginia.edu/stream/ref.html
- https://superfastpython.com/numpy-number-blas-threads/
