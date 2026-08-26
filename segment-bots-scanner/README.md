# Research: the universal opportunity scanner

**2026-08-26.** Five non-overlapping `sonnet` sweeps (Rule 1, Rule 5) run before
redesigning `opportunity-scanner`, whose live measurement was: 11 detectors, ~660,000
observations each, **10 of 11 producing zero candidates**, and the universal sweeper
holding **zero watch conditions** so it could never fire at all.

| | |
|---|---|
| [01](01-full-universe-scanner-architectures.md) | How real systems scan a full universe, and what breaks first |
| [02](02-cross-sectional-vs-threshold-detectors.md) | Cross-sectional ranking vs per-symbol absolute thresholds |
| [03](03-in-memory-layout-measured.md) | **Measured on this box** — does "move it to RAM" help? |
| [04](04-automated-rule-discovery.md) | How rule #1 gets born, and validated without self-deception |
| [05](05-perp-edges-and-free-data.md) | Which perp edges are real, and the exact free endpoints |

## The four findings that change the design

1. **"Move it to RAM" is the wrong framing — the data is already in RAM.** The win is
   *layout*: 1,500 symbols cost **157 ms** as a dict of Python objects and **0.054 ms** as
   a numpy structure-of-arrays with O(1) running stats. **2,922x, measured here**, with
   zero new dependencies. At this scale there is no wall — no numba, no shared memory, no
   columnar database is needed.
2. **Exchange stream limits are not the bottleneck at 1,586 symbols** — a handful of
   connections per venue. What breaks first is CPU per message and bus fan-out, which this
   project has already measured happening to itself.
3. **Cross-sectional ranking fixes the zero-signal problem definitionally and the edge
   problem not at all** — unless the common factor is regressed out first. In a
   one-factor market, ranking raw returns ranks *beta*. Alpha101's real shape is a
   per-symbol time-series feature **wrapped** in `rank()`, which means the existing eleven
   detectors are kept, not replaced.
4. **The instruction-writer deadlock is a schema problem, not a statistics problem.**
   Every credible system constrains what a hypothesis may look like *at creation*, rather
   than relaxing the gate that checks it. The writer refusing 479,323 of 479,323 is the
   verification apparatus working correctly and reporting an upstream defect.

## Immediately actionable

- **`liquidation-cluster-mapper` publishes 544,798 empty maps** because it has no margin
  schedule. Bybit's `GET /v5/market/risk-limit` is **public and unauthenticated**; Binance's
  `GET /fapi/v1/leverageBracket` is **signed**. The blocker is Binance-specific.
- **Drop five detectors' data sources outright**: cross-venue arbitrage (below retail
  latency), whale flows (free tier is >$500k and 10 calls/min), long/short ratio
  (rate-limit-disqualified and folklore), options IV (BTC/ETH only), and — pending a
  five-minute doc check — social sentiment.
- **Fold basis into funding.** On the evidence they are the same information twice.
