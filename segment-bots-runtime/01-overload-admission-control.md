# Overload, admission control and graceful degradation — on THIS box

Researched 2026-08-20 by a `sonnet` agent (Rule 1) for the ajit-segment-bots
part runtime. Kernel docs, SRE book and Netflix source read directly; several
claims live-verified on this VM. Report only — nothing was installed or changed.

## Verdict first

**No technique escapes the arithmetic.** Demand above capacity must wait, be
rejected, or be served at reduced quality. Every mechanism below only chooses
*which of the three*, for *which work*, on *which measured signal*. That is
physics, not an engineering gap.

"No queueing" read literally = load shedding by another name. The achievable and
correct goal is **no unbounded or hidden queueing**: an explicit, fast, measured
admit / reject-now / degrade-now decision instead of a queue that silently grows.

## Real-time theory is inapplicable — two independent reasons

Liu & Layland RM bound `U = Σ(Cᵢ/Tᵢ) ≤ n(2^(1/n)−1)` → ln2 ≈ 0.693; EDF necessary
and sufficient at `U ≤ 1`; Linux SCHED_DEADLINE admits at
`Σ(runtimeᵢ/periodᵢ) < M × (sched_rt_runtime_us / sched_rt_period_us)`, default
95% × cores.

1. **Needs root.** `sched(7)`: *"A thread must be privileged (CAP_SYS_NICE) in
   order to set or modify a SCHED_DEADLINE policy."* No passwordless sudo here.
2. **Needs a periodic task model we do not have.** The parts are event-driven with
   unknown arrival rates — there is no Cᵢ or Tᵢ to put in the inequality.
   Applying it is a category error, not merely overkill.

Kernel doc's own words on overload: *"clearly, if the system is overloaded this
guarantee cannot be respected."*

## What IS available unprivileged — live-verified on this VM

| Mechanism | Verified detail |
|---|---|
| `cpu.max` | two values `$MAX $PERIOD` in µs, default `max 100000`. `CPUQuota=25%` read back as `25000 100000`. **Hard ceiling**, CFS bandwidth throttling, no root |
| `cpu.weight` | range **[1, 10000]**, default **100**. Proportional only — matters solely under contention |
| `cpu.weight.nice` | same knob as nice value, [-20, 19], default 0 |
| `cpu.pressure` (PSI) | **world-readable, mode 0666, no capability needed to read.** Only poll()-trigger setup was ever gated |
| `/proc/pressure/cpu` | system-wide `full` line is **always zero by definition** — backward compatibility |
| per-cgroup `cpu.pressure` | `full` line IS meaningful and non-zero — every task in that cgroup stalled on CPU at once. **This is the real oversubscription signal** |
| `pids.max` | default `max`; enforcement is `fork()` returning `-EAGAIN`. Cheap fork-bomb breaker, free when unused |
| `memory.pressure` | read-only |

**Implementation friction found empirically, not in any doc:** raw `mkdir` under
the systemd-owned session scope is refused (`Permission denied`) even though
`cpu memory pids` are delegated — systemd owns that subtree exclusively.
**Launch each part via `systemd-run --user --scope -p MemoryMax=… -p CPUWeight=…
[-p CPUQuota=…]`. Do not hand-roll cgroupfs.**

Polling PSI by plain file read every few seconds sidesteps the trigger-privilege
question entirely — no poll() API needed for this workload.

## Shed versus degrade — the one piece of theory that transfers

Google SRE book, *Addressing Cascading Failures* (fetched directly):

- **Load shedding = reject outright.** *"return an HTTP 503 … when there are more
  than a given number of client requests in flight."* Also names LIFO-over-FIFO,
  CoDel, and priority selectivity.
- **Graceful degradation = keep serving, do less work.** *"search a subset of data
  stored in an in-memory cache rather than the full on-disk database, or use a
  less-accurate (but faster) ranking algorithm."*

Cost: shedding costs completeness; degradation costs accuracy AND requires a
second cheaper code path built and tested in advance. Degradation is strictly
more engineering.

## Backpressure algorithms worth porting, not installing

- **CoDel (RFC 8289)** — bounds queue *sojourn time* (~5 ms target), not depth;
  a standing queue is pure waste. **Not a pip package.** Reimplement the ~30-line
  idea only if a real queue with growing sojourn time exists.
- **Netflix concurrency-limits** (Java, source read): `VegasLimit` grows by
  α = max(3, 10% of limit), shrinks by β = max(6, 20% of limit);
  `Gradient2Limit` scales by `gradient ≈ min_RTT / current_RTT` with long-term
  smoothing. Admission on **measured latency gradient, not queue depth**. A few
  dozen lines to port to Python.
- **Brownout (Klein et al., ICSE 2014)** — a continuously-tuned "dimmer" giving
  optional code paths an execution probability 0–1 under a feedback controller.
  Real cost: you must pre-identify safely-optional computations and thread the
  dimmed-out case through the whole application. Not worth it until 2–3 specific
  expensive-and-optional computations exist.

## Recommended stack, in build order

1. Poll `cpu.pressure` / `memory.pressure` per part-cgroup every few seconds — the
   single oversubscription signal. **Replaces all RM/EDF admission math.** PSI
   measures "runnable but not running", which is the actual definition; CPU%
   conflates "busy and fine" with "busy and starved".
2. `systemd-run --user --scope` with `MemoryMax` + `CPUWeight`; add `CPUQuota`
   only where a hard ceiling is needed (runaway training jobs).
3. `pids.max` per part as a fork-bomb breaker.
4. The governor decides shed-vs-degrade-vs-throttle per part type. **This is the
   only piece of custom logic worth writing** — everything upstream is measurement.

**Skip:** SCHED_DEADLINE/CBS (root + wrong model), CoDel as a library (does not
exist), brownout's full controller (unearned at this stage).

## UNVERIFIED

- **AWS Builders' Library load-shedding article** — WebFetch returned only page
  chrome; content not independently confirmed. Only the consistently-reported
  claim "shed early, shed cheap, shed in layers" should be relied on.
- **Anytime algorithms (Zilberstein)** — named as the correct academic term for
  self-degrading CPU-bound parts, but not fetched from a primary source here.
- **CoDel outside networking** — only research papers found (e.g. LSTFCoDel); no
  maintained general-purpose task-queue implementation.
- Liu & Layland formulas cross-checked against secondary academic sources rather
  than the 1973 paper itself; standard and uncontested in the field.

## Sources

- https://docs.kernel.org/scheduler/sched-deadline.html (direct)
- https://man7.org/linux/man-pages/man7/sched.7.html
- https://www.kernel.org/doc/Documentation/admin-guide/cgroup-v2.rst (direct)
- https://docs.kernel.org/accounting/psi.html (direct) + live-verified here
- https://lkml.iu.edu/hypermail/linux/kernel/2303.3/08412.html (unprivileged PSI polling)
- https://sre.google/sre-book/addressing-cascading-failures/ (direct)
- https://people.cs.umu.se/cklein/publications/icse2014-preprint.pdf (brownout)
- https://www.rfc-editor.org/rfc/rfc8289.html (CoDel)
- https://github.com/Netflix/concurrency-limits (VegasLimit.java, Gradient2Limit.java, read directly)
