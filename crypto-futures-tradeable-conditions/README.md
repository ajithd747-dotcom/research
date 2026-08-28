# When is a crypto futures symbol tradeable? — six-agent research sweep

**Run on 2026-08-28** for the `ajit-segment-bots` project, in answer to: *every condition or
opportunity a crypto futures symbol can present that, when met, means it can be traded for profit* —
plus a follow-up on candle/price-action conditions across 1m/5m/15m/30m/1h.

**Six `sonnet` research agents, non-overlapping areas, all instructed to report only, to cite
exactly, to mark what they could not verify, and to say plainly what is not worth building.**

| File | Area |
|---|---|
| [01](01-funding-basis-oi-liquidations-crowding.md) | Funding, basis, open interest, liquidations, crowding |
| [02](02-order-book-microstructure.md) | Order book and order flow microstructure |
| [03](03-volatility-and-regime.md) | Volatility estimation, forecasting, regime classification |
| [04](04-momentum-breakout-mean-reversion.md) | Momentum, breakout, mean reversion, regime detection |
| [05](05-relative-value-events-and-liquidity-screening.md) | Cross-venue, cointegration, events, liquidity screening |
| [06](06-candles-price-action-and-timeframes.md) | Candles, price action, multi-timeframe |

Synthesis below is the main thread's, not an agent's.

---

## The four things worth acting on

### 1. Three agents, three different topics, independently concluded the *gate* is worth more than the *signal*

This is the strongest finding in the sweep, precisely because nobody was asked to look for it:

- **File 03** — the best-evidenced claim any agent found, from a peer-reviewed on-point paper
  (Daniel & Moskowitz, *Momentum Crashes*, JFE 2016): momentum's worst losses are **concentrated in
  an identifiable state** (elevated vol *following a decline*), and conditioning exposure on that
  state materially reduces crash risk. The naive intuition — high vol means strong trend, size up —
  **is backwards for exactly the state that produces the worst outcomes.**
- **File 01** — crowding is **a filter, not an edge**. Do not build a standalone crowding entry.
- **File 05** — liquidity screening **gates everything else**: on our weaker symbols the cost floor
  is 30-100 bps+, which eats most of the statistical edges in files 01-04 before the strategy's own
  gross edge is counted.

**Implication for this project:** effort spent on regime gates and tradeability screens is better
spent than effort on new detectors. We have nine detectors and a scanner; we do not have a
measured, per-strategy-family regime gate.

**And one caution the same file raises:** the gate cannot be one global switch. The state that
protects momentum is exactly the state a reversal strategy wants to enter into.

### 2. Multiple testing is the recurring failure mode, across every topic

Three agents reached it independently from different directions:

- **File 05:** scanning ~100 symbols pairwise is ~1,225-4,950 tests; **at a naive 5% threshold expect
  ~60-250 "cointegrated" pairs by chance even if none are real.**
- **File 06:** every candlestick-positive result reports the best of a handful of tested patterns with
  no bootstrap null or deflated Sharpe. The **one** study using a proper bootstrap null found no
  significant edge.
- **File 04:** a systematic falsification study with walk-forward splits, permutation testing and
  positive controls found **all 14 retail signal families failed** — and its most common failure mode
  was "one strong year masking flat or negative other years."

**The tools named for this: White's Reality Check, the Deflated Sharpe Ratio (Bailey & López de
Prado), Benjamini-Hochberg FDR across a scan, and Harvey/Liu/Zhu's t > 3.0 rather than t > 2.0.**

**This contradicts something the running system currently does.** `cointegration-pair-finder`
reports **263 cointegrated pairs out of 2,211 correlated pairs**. Against the arithmetic above,
that count is exactly the shape a multiple-testing artefact produces, and the part does not appear
to apply an FDR correction or a held-out re-validation. **Check that before any of those verdicts
is acted on.**

### 3. Order flow is where two independent agents converged

- **File 02** (microstructure): if forced to keep one signal, **OFI (Cont-Kukanov-Stoikov)** — best
  evidenced, formula-level, computable from the depth feed alone without trade prints, and its own
  result (fit *improves* at longer aggregation) works in a non-colocated participant's favour, which
  is unusual for this literature.
- **File 06** (candles): of all bar-level statistics, **order-flow imbalance at bar resolution is the
  best-supported**, because it has a real mechanism (informed-trader price impact) that named
  candlestick patterns never had.

**And we have a data advantage the equities literature lacks:** both venues label the taker side on
every trade, so no Lee-Ready tick-rule inference is needed. VPIN's bulk volume classification exists
precisely to work around not having that.

### 4. Two free, public data feeds are missing entirely

**File 01's most concrete finding.** Open interest (`/fapi/v1/openInterest`, `/v5/market/open-interest`)
and the liquidation tape (`!forceOrder@arr`, Bybit's `liquidation` topic) are **public, keyless, and
captured by nothing in this system.** Which means:

- OI build-up / divergence / flush **cannot be computed at all today**.
- Liquidation-cascade work rests on a proxy for a proxy.

History accrues only in real time. If these are ever wanted, the capture should start before the
analysis is designed — the same argument that started the trade tape on 2026-08-22.

---

## Contradiction worth resolving, and it lands on code we are changing today

**File 01 and file 05 appear to disagree about funding and basis** — 01 calls them the only two
topics with real peer-reviewed evidence; 05 calls the trade decayed to a negative Sharpe by 2025.
**They do not actually conflict: the phenomenon is real, the edge is arbitraged.** Both should be
read as "use as a regime/sizing input, not as a P&L source."

**But file 01 raises something that directly contradicts a design assumption in the tailgater.**
`tail-crowding-detector` requires **2 of 3 sources** (book imbalance, funding deviation, sentiment)
before it will report a crowding reading, on the stated reasoning that "one source is a guess about
the crowd." File 01's finding is that **funding, basis and crowding are one underlying fact —
"leverage is one-sided" — measured three ways, not three independent confirmations.** And file 06
makes the same argument for nested timeframes.

So "two of three sources agreed" is **partly an illusion of independence**, in the same way
"three timeframes agree" is. The two sources that are genuinely independent of each other here are
**the book** (microstructure) and **funding** (positioning). Sentiment would be a third. That is
worth knowing before raising or lowering `tail_crowding_minimum_sources`.

---

## What every agent said to skip

- **Named candlestick patterns.** The only study with a proper null found nothing.
- **Retail breakout rules** (opening-range, volume-confirmed) — falsified as a family under realistic costs.
- **ADX as a regime classifier**, and **static full-sample Hurst** as a live signal — folklore.
- **Cross-venue latency arbitrage** between our two venues — dead for anyone non-colocated.
- **Event-driven listing/delisting trading** — we have no announcement feed, and the abnormal return
  happens *before* a listing appears in our data.
- **Any microstructure signal as a standalone taker trigger** against the ~10-11 bps round-trip floor.
- **Trusting any cointegration output on 6 days of tape.** Build the pipeline; do not trade it.

## What the sweep says we cannot do yet, for want of history

**~6 days of tape rules out, by the agents' own arithmetic:**

| Method | Needs |
|---|---|
| HAR-RV | ≥22 days for the monthly term; 250+ for stable coefficients |
| GARCH/EGARCH | 100+ observations for a stable MLE |
| A 2-4 state HMM regime model | enough tape to observe **multiple real regime transitions** |
| Cointegration scanning | weeks-to-months, plus FDR control and walk-forward re-validation |
| Liquidation-reversal rules | enough captured cascades — they are rare tail events |

**Usable today:** realized variance from 1m bars (precision comes from within-day return count, not
calendar days), EWMA vol, the mechanical rolling-percentile vol regime, order-flow imbalance, and
the liquidity screen.

**The single highest-value thing to start now that costs nothing but time: accumulate the daily
realized-vol series, and start capturing OI and liquidations.** None of it can be back-filled.

---

## Reading these files

Every one has an **UNVERIFIED section at the end, preserved rather than smoothed over.** Several
load-bearing numbers in the sweep — the multi-timeframe win rates, the squeeze backtest statistics,
the carry Sharpe decay figures, the liquidity thresholds — are secondary-source or vendor-blog tier
and are marked as such. **Two agents flagged that a fetch failed and they fell back to an abstract;
neither fell back to memory.** Where an agent gave its own reasoning rather than a citation, it said
so. Read those sections before writing code against any number here.
