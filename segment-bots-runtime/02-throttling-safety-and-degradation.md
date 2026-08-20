# When is throttling a part actually safe? — and how degradation is made visible

Researched 2026-08-20 by a `sonnet` agent (Rule 1) for the ajit-segment-bots
part runtime. Report only. The headline finding contradicts the clock-gating
design proposed earlier in that session — see "What this overturns".

## What this overturns

Clock-gating was presented as the drawback-free replacement for eviction. It is
not. **Reducing tick rate is safe only where it changes WHEN an answer arrives,
never WHAT the answer is.** Three independent research tracks converged on this.

The classification axis is not advisory-versus-decision. It is:

- **(a)** does a lower rate change *the number that comes out* (bias/correctness
  risk) or only *when it arrives* (latency risk)?
- **(b)** is there a monotone invariant — sequence completeness, reconciled state —
  that a skipped tick *corrupts* rather than merely *delays*?

Only parts that are latency-risk-only on both axes belong in a full/half/quarter/
idle ladder. Everything else needs a reserved, non-competed compute floor.

## Safe to throttle

Work where slowing down adds latency to an otherwise unchanged, already-durable
computation:

- draining an already-buffered Kafka or Redis stream more slowly — these are
  pull-based **by design** so a slow consumer catches up without loss
  (Confluent consumer design docs)
- polling a REST endpoint no faster than its own update cadence — polling faster
  than the source changes yields no new information anyway
- MCTS-style search run for fewer iterations — a genuinely proven quality curve

Even for a decision-feeding part, provided its output carries an honest
staleness and confidence signal downstream.

## NOT safe to throttle — regardless of advisory status

**1. Recursive (IIR) indicators — the most important finding for this project.**
EMA, RSI, ATR, MACD carry a seed-value bias that decays only with sufficient
history. TA-Lib documents this as the "unstable period"; QuantConnect's forum
puts EMA(20) at roughly 1.3 days of 1-minute bars before the bias is negligible.

> Shrinking the lookback window is **not a coarser answer to the same question —
> it is a different, silently biased answer.**

SMA-family indicators are exact-once-full and undefined before: a readiness gate,
not a quality dial. The only genuinely anytime lever on an indicator is reducing
*update frequency over a fixed, already-warmed window* — never shrinking the
window itself. A biased EMA looks identical to a healthy one on any dashboard not
built specifically to catch it, which is precisely the failure Rule 8 exists for.

**2. Order-book reconstruction and position/balance reconciliation.** Binary
correctness, not a quality scalar. CME MDP 3.0 recovery docs: on any sequence gap
*"it should be assumed that all books maintained in the client system may no
longer have the correct, latest state"* — full resync, no partial credit.
Concrete cautionary case: `kalshi-python-sdk#189`, a book that permanently
desyncs after one dropped sequence number because resync was never implemented.
Throttling here does not give a worse answer; it gives a **wrong** answer that
looks fine until it silently is not.

**3. The socket-read loop itself on a live feed.** Coinbase documents
`ErrSlowConsume`: *"market data is not being consumed fast enough — the
server-side buffer can fill, resulting in delayed and killed connections"*, and
prescribes offloading processing to a separate thread so the read loop never
blocks. Throttling a read loop converts graceful degradation into a **hard
disconnect** — a step function, not a curve.

**So "reduce tick rate" is at least three different operations** with wildly
different risk: slow a durable-backed downstream reader (safe); slow an
already-capped REST poll (near free); slow the socket read itself (dangerous).
**Audit which one each part actually does before trusting the ladder.**

## What still needs a clock — cannot be event-driven

Genuinely event-driven is real, not marketing: asyncio blocks in `epoll_wait()`
via `selectors`, so an idle process burns zero CPU because the kernel parks the
thread. Exchange websockets (Binance, Kraken, Coinbase) push events server-side.

But these inherently need a clock: websocket ping/pong keepalive and 24-hour
forced reconnects; listenKey / auth-token renewal; **staleness and liveness
detection** (definitionally impossible to trigger from an event — the failure
mode *is* the absence of events); periodic REST reconciliation, needed precisely
because streams can silently drop; bar-close boundaries; manufactured heartbeats
against idle-disconnect on quiet symbols; retraining.

## Anytime algorithms — real, narrow, and mostly not applicable here

Russell & Zilberstein (IJCAI-91; AIJ 1996), Zilberstein AI Magazine 1996.

- **Contract** algorithms need the budget in advance and may return nothing if cut
  short. **Interruptible** ones give a usable answer at any point. Every
  interruptible algorithm is a contract algorithm; the converse is false.
- Converting contract → interruptible by restart at doubling budgets carries a
  **proven, tight 4x worst-case penalty** (Zilberstein & Mouaddib, IJCAI-99).
  Anytime-ising something is not free.
- **Composition destroys interruptibility even when every component is
  interruptible.** Chaining anytime parts does not give an anytime pipeline.
- Genuinely anytime: MCTS, EM, AD*/ARA* motion planning (actually fielded —
  DARPA MARS), adaptive quadrature, online aggregation / BlinkDB. Honesty check:
  a follow-up study found BlinkDB-style error bars frequently fail on real
  production workloads (ACM 10.1145/2588555.2593667). "Anytime" ≠ "safe".
- Zilberstein names stock trading as a plausible beneficiary in 1996. That is a
  motivating example, **not a demonstrated deployment**, and nothing public has
  closed that gap since.

## Adaptive sampling, bandits, successive halving — weak production pedigree

- **Event-triggered / Lebesgue sampling** (Åström & Bernhardsson, CDC 2002) and
  **CUSUM variable-sampling-interval charts** (Arnold 2001: 25–50% fewer samples
  with *better* detection) have real control-theory pedigree. **No confirmed
  production trading system doing volatility-triggered polling was found** —
  Coinbase, kdb+/KX and BIS Project Rio all push at full rate and scale the
  consumer instead. Failure modes: near-Zeno unbounded triggering under noise
  (needs a hard minimum-interval floor, which partly defeats the saving);
  frog-boiling (instant triggers miss slow drift); and a stopped-clock gap — a
  part that never triggers still needs an independent heartbeat.
- **Bandits** (UCB1, Thompson) are production-proven at Netflix, Spotify, Alibaba
  — but **only for allocating traffic to users, never compute to components.**
  Large recsys pipelines manage compute with a **static cascade** (cheap model
  narrows, heavy model runs on survivors — Meta ads ranking), not a bandit.
  Directly relevant failure: non-stationarity means a long-quiet arm is not
  "owed" exploration, so **a bandit can starve a part exactly when it becomes
  important.** The literature's own fix is not a cleverer bandit — Conservative
  Bandits (Wu et al., ICML 2016) enforce a hard floor via a non-competed safe
  default; the standing advice is to **exclude safety-critical arms entirely.**
- **Successive Halving / Hyperband / ASHA** are real and production-deployed —
  but **only as offline hyperparameter-tuning infrastructure** (Ray Tune,
  Determined AI), never as a live control loop over deployed components. SH's
  correctness proof requires candidate loss to converge monotonically at a rate
  the fixed schedule reaches; **a market regime flip mid-tournament violates that
  precondition directly.** Permanent elimination cannot tell "consistently bad"
  from "quiet because not needed yet" — a stop-loss part that has not fired looks
  identical to a useless one. **No prior art found: apparently novel, not borrowed.**

**Cross-cutting rule all three tracks converged on independently: keep
safety-critical and risk-monitoring parts out of any adaptive allocation
mechanism. Give them a reserved, non-competed compute floor.**

## Making degradation visible — thin literature, which is itself the finding

- **Brownout** (Klein, Maggio et al., ICSE 2014, doi 10.1145/2568225.2568227): a
  dimmer `Θ ∈ [0,1]` set by a PI controller against a response-time SLO, ridden
  on every response as an `X-Dimmer` header. Confirmed from the reference
  implementation's README: that signal is consumed by a **load balancer, not an
  operator dashboard.** The graded-number-riding-with-every-answer pattern is
  borrowable; the human-facing layer is not demonstrated anywhere.
- **Hystrix** distinctly counts `countSuccess` vs `countFallbackSuccess` — a
  cheap, proven "fraction currently degraded" gauge, directly usable. Netflix's
  load shedding has a `DEGRADED_EXPERIENCE` class, but that is a pre-assigned
  routing priority by request *type*, not a measured quality score.
  **Circuit-breaker literature is almost entirely about availability ("did we
  serve something"), not answer quality ("how good was it").**
- **Google SRE Workbook** gives the strongest citable vocabulary found: a
  **Quality SLI = "the proportion of responses that were served in an undegraded
  state"**, plus freshness/correctness SLO patterns for pipelines ("the oldest
  data is no older than Y minutes"). One org's book; no independent case study found.
- **Practical gotcha that bites Rule 8 directly:** the IETF health-check draft's
  `pass/warn/fail` and .NET's `Healthy/Degraded/Unhealthy` are real shipped
  conventions, but returning `{"status":"degraded"}` inside an **HTTP 200** is
  invisible to anything routing on status code alone. A degradation signal that
  lives only in a JSON field nothing reads is not a status.
- **Prometheus `StaleNaN` does not solve this** — it detects "the metric stopped
  reporting", not "the metric reports fine but the thing it measures degraded".
  An explicit application gauge is required, e.g.
  `part_tick_rate_ratio{part="scanner"} 0.25` plus a staleness timestamp on the
  output itself.

**Bottom line:** the combination actually wanted — a continuous per-part
quality/rate-ratio gauge plus a Google-style Quality SLI framing — **is not
documented anywhere as an assembled pattern.** Building it is original
engineering, not copying a playbook. Under Rule 8 it is not optional scaffolding
around the governor; it is the only thing that makes the governor's central risk
— silent quality loss — checkable at all.

## UNVERIFIED / weakest evidence

- **No confirmed production trading system** using anytime algorithms,
  volatility-adaptive sampling, or bandit-based compute allocation was found in
  this pass. Borrow the failure-mode reasoning; do not assume the pedigree transfers.
- Successive-halving as a live governor: no prior art at all. Novel, unproven.
- Google Quality SLI framing: single-source (one org's book).
- Brownout's operator-facing layer: does not exist in the primary source.

## Sources

- https://docs.python.org/3/library/asyncio-eventloop.html · https://man7.org/linux/man-pages/man2/epoll_wait.2.html
- https://github.com/binance/binance-spot-api-docs/blob/master/user-data-stream.md
- https://docs.kraken.com/api/docs/websocket-v1/owntrades/
- https://docs.cdp.coinbase.com/exchange/websocket-feed/best-practices (ErrSlowConsume)
- https://docs.confluent.io/kafka/design/consumer-design.html
- http://rbr.cs.umass.edu/shlomo/papers/ZRaij96.pdf · http://rbr.cs.umass.edu/shlomo/papers/Zaimag96.pdf
- https://www.ri.cmu.edu/pub_files/pub4/likhachev_maxim_2005_1/likhachev_maxim_2005_1.pdf
- https://dl.acm.org/doi/10.1145/2588555.2593667 (BlinkDB error bars fail in practice)
- https://github.com/TexasCoding/kalshi-python-sdk/issues/189 (order-book desync)
- https://eugeneyan.com/writing/bandits/ · Wu et al., Conservative Bandits, ICML 2016
- https://engineering.fb.com/2026/08/05/ml-applications/from-user-sequences-to-scaling-laws-a-multi-stage-architecture-for-metas-ads-ranking/
- https://doi.org/10.1145/2568225.2568227 (Brownout)
- https://netflixtechblog.com/keeping-netflix-reliable-using-prioritized-load-shedding-6cc827b02f94
- https://sre.google/workbook/implementing-slos/ · https://sre.google/workbook/data-processing/
