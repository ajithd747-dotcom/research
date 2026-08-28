# Funding, basis, open interest, liquidations and crowding — which are real edges

**Researched by:** a `sonnet` research-web subagent, dispatched from the segment-bots
project on **2026-08-28**. Question asked: which funding-rate, basis, OI, liquidation
and positioning conditions actually confer a tradeable edge in crypto perpetuals, and
when does each stop working.

**Constraints given to the agent:** Binance USDM + Bybit linear, ~30-50 symbols; data
available = trades, 1m candles, partial-depth books, quotes, mark/index/declared
funding; ~6 days of tape; no on-chain data, no options surface, no paid sentiment
feed, no leaderboard data. Paper trading now.

**Read the UNVERIFIED section at the end before coding against any number here.**

---

## The finding that changes the project's scope

**Two of the five topics are not computable from the data this system captures.**

- **Open interest is not on any wire we have.** Binance publishes it at
  `/fapi/v1/openInterest` and `/futures/data/openInterestHist`; Bybit at
  `/v5/market/open-interest`. Both are public, keyless, separate from trades and
  candles. Nothing in this system captures them, so OI build-up, OI divergence and
  OI flush cannot be computed at all today.
- **Liquidations are not on any wire we have.** Binance publishes the actual
  liquidation tape on the `!forceOrder@arr` websocket; Bybit on its `liquidation`
  topic. What can be built from trades+depth is a *proxy* (large market orders
  through multiple depth levels, sudden one-sided book removal, price velocity
  spikes) which is materially weaker than the venue's own liquidation prints.
- **Long/short account ratios (CoinGlass-style) are exchange-internal account
  aggregates and cannot be derived from public market data at all.**

Both missing feeds are free and public. This is a capture gap, not a research gap.

---

## 1. Funding rate — level, extremes, sign flips, predicted vs declared

Both venues use the same skeleton:

    Premium Index P = [max(0, ImpactBid − IndexPrice) − max(0, IndexPrice − ImpactAsk)] / IndexPrice
    FundingRate F   = clamp(AvgP + clamp(InterestRate − AvgP, −0.05%, +0.05%), lower, upper)

- Binance: interest term default 0.01%/8h (0.03%/day); outer clamp typically ±0.05%,
  but **symbol-tiered**.
- Bybit: identical structure; interest 0.01% per 8h; outer limit computed from
  `min((initial_margin_rate − maintenance_margin_rate) × 0.75, maintenance_margin_rate)`,
  adjustable during volatility.
- **`AvgP` is a TWAP of 1-minute premium samples over the funding interval, not a
  point-in-time reading.** This matters directly for us: predicting the declared rate
  intraperiod needs the *running TWAP*, not the instantaneous premium.

**On "predicted vs declared":** both venues publish a live next-funding estimate that
*is* this running TWAP recomputed continuously. There is no statistical edge beyond
implementing the venue's own formula correctly. Anything claiming to predict funding
ahead of the mechanical TWAP is either replicating the exchange formula or has no edge.

**Evidence**
- He, Manela, Ross, von Wachter, *Fundamentals of Perpetual Futures*, arXiv:2212.06888
  (rev. Aug 2024) — derives no-arbitrage bounds; deviations larger than in traditional
  FX, **comove across currencies, and diminish over time**. Implied arbitrage strategy
  has high Sharpe **in-sample**. <https://arxiv.org/abs/2212.06888>
- Schmeling, Schrimpf, Todorov, *Crypto Carry*, BIS Working Paper 1087 (rev. Oct 2025;
  also *Management Science*) — carry averages **>10% annualised**, sometimes **>40%/yr**,
  large because arbitrage capital is capital- and margin-constrained, **not because
  there is no risk**. <https://www.bis.org/publ/work1087.htm>
- *Exploring risk and return profiles of funding rate arbitrage on CEX and DEX*,
  ScienceDirect 2025 — best configuration 115.9% over 6 months, max drawdown 1.92%.
  "Best-performing" language is a strong sign of selection over many configurations.
  <https://www.sciencedirect.com/science/article/pii/S2096720925000818>
- BitMEX, *9 Years of XBTUSD Funding Rate Analysis* — exchange-published, self-interested,
  useful as primary distributional data. <https://www.bitmex.com/blog/2025q2-derivatives-report>

**Horizon:** carry is one 8h interval to weeks. Funding-extreme fading (outright, not
delta-neutral) is hours to a few days — and has no peer-reviewed backtest at that horizon.

**What invalidates it — this is not free money.** It is short volatility with a fat left
tail. The BIS framing (limited arbitrage capital due to margin/regulatory frictions)
means the carry is *compensation for basis risk*, not an inefficiency. The same paper
documents carry ranging **-50% (Nov 2022, FTX) to +45% (pre spot-ETF, Jan 2024)**: the
same "arbitrage" loses money for months at a time. Delta-neutral is not neutral under
stress — an outage, ADL, or a spot-perp basis blowout during a cascade can force one leg
out while the other remains, at the worst moment.

**Decay (2025-2026): heavy.** *Cryptocurrency as an Investable Asset Class*
(arXiv:2510.14435, 2025) reports the crypto-carry Sharpe falling from **6.45 (2020-2025
full sample) to 4.06 from 2024, and negative in 2025**. This is the most crowded of the
five topics — the first thing every quant shop and every delta-neutral DeFi vault does.
**Do not treat funding-extreme fading as a standalone strategy.** Use funding level and
sign as one input to a crowding/risk filter.

---

## 2. Basis / premium (mark vs index)

**Directly computable from what we now capture** (mark and index per symbol per second,
as of the `venue-premium` wire added 2026-08-28):

    basis_bps = (mark_price − index_price) / index_price × 10_000

**Evidence:** BIS *Crypto Carry* (above) is the primary paper. Ackerer, Hugonnier,
Jermann, *Perpetual Futures Pricing* (Wharton) derives why basis mean-reverts — funding
is the restoring force by construction; it is pricing theory, not an empirical edge test.
<https://finance.wharton.upenn.edu/~jermann/AHJ-main-10.pdf>

**The trap worth writing down:** basis and funding are *mechanically linked* — basis is
literally the input to funding via the premium index. They are not two independent edges;
they are close to the same signal measured twice. **A system that treats a basis-widening
signal and a funding-extreme signal as independent confirming evidence is double-counting
one variable.** Basis also blows out mechanically during a cascade (the perp book empties
faster than the index composite reprices), so trading basis-widening as "carry opportunity"
in those windows is trading into the cascade.

---

## 3. Open interest — build-up, divergence, flush

**Not capturable today** (see top). Beyond that:

**This is the weakest-evidenced topic of the five.** The agent could not find a single
peer-reviewed or exchange-published quantitative OI-divergence study with a stated formula
and an out-of-sample result. Everything found was blog tier. The circulating thresholds
("$80B aggregate OI → 10-20% pullback within 30 days"; "30%+ OI drop → bottom within 1-2
weeks") trace to a single Gate.io wiki article with no methodology, no confidence interval
and no out-of-sample test — single-cycle, BTC-specific, likely fitted.

Also note "rising OI + rising price = healthy" is close to **tautological** in any trend
(more capital chases a trend by definition), so it is only informative at genuine extremes
that the sources never define rigorously.

**Verdict: do not build a strategy on OI divergence or flush as specified anywhere public.**
Capture OI anyway — as a level feeding a crowding/risk filter it has theoretical grounding
that the divergence patterns do not.

---

## 4. Liquidation cascades

**Evidence — and it cuts against easy alpha:**

- *Where does the criticality live? Early-warning signals are event-heterogeneous across
  seven crypto-perpetual liquidation cascades*, arXiv:2607.27070 — **no single variable is
  event-invariant.** Price shows a critical-slowing-down signature in 5 of 7 events and is
  silent in the 2 news-shock events. **A single-signal early-warning system will miss a
  whole category of cascades by construction.** <https://arxiv.org/abs/2607.27070>
- *Measuring the engine of a liquidation cascade: subcritical branching inside a
  first-order transition*, arXiv:2608.03616 — the Oct 2025 event (~$19B, largest on record)
  measured **subcritical, λ≈0.1-0.2 throughout**: each liquidation triggered fewer than one
  further liquidation on average. Evidence *against* the naive self-sustaining-cascade
  model. <https://arxiv.org/html/2608.03616>
- A practitioner backtest claiming a parameter-free contagion-fade at +2.33%/trade found
  **54% of the return was BTC market beta**, and regression alpha was **not statistically
  significant** after controlling for it. Source unfetchable (HTTP 403) so the numbers are
  UNVERIFIED — **but the lesson stands on its own logic: check any liquidation-fade
  backtest for beta contamination before trusting its Sharpe.**

**Horizon:** minutes to a few hours; cascades resolve fast.

**Verdict:** out of scope until the real liquidation stream is captured, and even then run
it as a **regime filter** (reduce size, widen stops) rather than a directional strategy.

---

## 5. Positioning and crowding

The published version (CoinGlass top-trader ratios) is not computable from public data.
What *is* buildable here:

- **Funding level and persistence** as a crowding proxy — best-grounded of the three,
  because funding is *mechanically* the price of being on the crowded side.
- **OI relative to its own trailing distribution** (once captured), combined with funding sign.
- **Taker buy/sell volume imbalance** from the trade tape — the closest available thing to
  "who is initiating" without account data.

**Evidence: none.** No peer-reviewed or exchange-published backtest of a crowding index
built this way. Every source described the *concept* without a formula, threshold, or
out-of-sample result. CoinGlass's own docs document how the ratio is *constructed*, not
that it predicts anything.

**What invalidates it:** crowded is not "about to reverse" — a market can stay one-sided for
weeks. Treating crowding as a timing signal rather than a sizing input is the standard
mistake. And a crowding index built from funding + OI + basis is **largely re-measuring one
underlying fact ("leverage is one-sided") three ways**, not three independent confirmations.

**Verdict: crowding is a filter, not an edge.** Do not build a standalone crowding entry.

---

## Bottom line

- **Funding carry and basis are the only two with BIS/peer-reviewed-grade evidence of a
  genuine phenomenon** — and the same evidence shows Sharpe falling from ~6.45 to negative
  by 2025, with a documented -50% swing. Short-vol compensation, not free money.
- **OI divergence and crowding-as-signal have no rigorous public evidence** behind the
  thresholds circulating online. Fold both into a sizing/risk filter, never a signal generator.
- **Liquidation cascades have real 2025-2026 academic attention and conclusions that cut
  against easy alpha.** Watch, don't trade, and certainly not without the real feed.
- **Two structural facts for the spec:** the OI and liquidation feeds are missing entirely
  and are free to add; and funding, basis and crowding are one fact measured three ways.

---

## UNVERIFIED

1. **Exact Binance/Bybit funding clamps (±0.05%, 0.01%/8h) and outer limits** — from search
   summaries of secondary pages plus search excerpts of Bybit help text. Direct fetches of
   Binance's premium-index FAQ and Bybit's funding-calculation article both failed. **Pull
   these from the live REST endpoints before coding against them** — limits are symbol-tiered
   and change.
2. **Whether "index price" on our feed is a pure spot composite** — not confirmed against
   either venue's raw docs (same fetch failures).
3. **The liquidation-fade backtest's methodology** (+2.33%/trade, 54% beta) — WebFetch 403.
   Numbers unconfirmed; the beta-contamination lesson is sound regardless.
4. **BIS Crypto Carry sub-period Sharpe and drawdown figures** — only the abstract was
   readable; the PDF fetch returned unparseable binary. The ">10% average, >40% peak" and
   "-50% to +45%" range come from the search tool's extraction of the abstract.
5. **The "$80B OI → pullback" and "30% OI drop → bottom" claims** — unsourced, single-source,
   no methodology. Reported only as existing and unreliable.
6. **Whether the public OI and liquidation endpoints are stable and low-latency enough for
   real-time use** — not verified directly; confirm against live API docs before building capture.
