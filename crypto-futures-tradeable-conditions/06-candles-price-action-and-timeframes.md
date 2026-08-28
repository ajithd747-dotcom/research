# Candles, price action, and multi-timeframe conditions

**Researched by:** a `sonnet` research-web subagent, **2026-08-28**, in answer to the question
"are there conditions on candle charts / price movement across 1m, 5m, 15m, 30m and 1h worth
checking?"

**Constraints given:** trades available so candles can be built at any interval; taker fees
4-5.5 bps/side, so a 1m signal must clear roughly 10 bps round trip.

**This is the area with the most folklore per unit of evidence in all of trading, and the report
was asked to say so plainly. It does.**

---

## Headline

**Named candlestick patterns (engulfing, hammer, doji, pin bar) do not survive rigorous testing as
standalone signals, in equities or in crypto.** What does show something real is:

1. **Bar-shape statistics used as continuous features**, not binary pattern flags.
2. **A few specific intraday crypto regularities that are not candlestick patterns at all** —
   turn-of-the-candle timing, funding-driven basis reversion.
3. **Round-number order clustering** — which is a fact about *where orders sit*, not a fact about
   future returns.

**Multi-timeframe "confirmation" as commonly practised is mostly correlated redundancy dressed up
as independent evidence. Nothing found clears ~10 bps round trip at 1m from candle shape.**

---

## 1. Candlestick patterns

**The definition problem comes first, because it is the crux.** There is no single definition.
"Engulfing" varies by body-only vs body+wick containment, and by whether it requires a preceding
trend (Nison's original rules do). "Doji" thresholds range from near-zero to 5-10% of range across
papers, none standardised. **This is not a minor detail — positive and negative results on the same
nominal pattern differ partly because of definition choice.** And it means **any reported win rate
is conditional on an arbitrary threshold you can tune post-hoc: a multiple-testing trap by
construction, not by accident.**

**The reference negative result:** **Marshall, Young & Rose (2006), *Market Timing with Candlestick
Technical Analysis*** (SSRN 980583; *Pacific-Basin Finance Journal*), DJIA components 1992-2002.
Crucially it used **bootstrap-generated random OHLC series as the null** — not buy-and-hold — which
is a far stronger test. **Result: candlestick strategies produced no statistically significant
excess returns versus the bootstrap null.** Their Japanese-equity follow-up reached the same
conclusion.

**The pattern in the literature is the most important fact here:** the negative results come from
the strongest tests (DJIA + bootstrap null); the positive results (2017 Thailand SET, 2022 SSE50
China) come from **weaker nulls (buy-and-hold) in smaller, less liquid, less efficient markets.**

**Crypto:** one Bitcoin study (2012–Jul 2020) found **only classical Engulfing** among three tested
patterns showed a positive profit factor (3.54); the other two did not. Single market, no described
out-of-sample split, no multiple-comparison correction. **Testing three patterns and reporting the
one that worked is itself a small multiple-testing exercise.**

**The load-bearing gap: no crypto study using the Marshall/Young/Rose bootstrap-null methodology
could be found — the one methodology strong enough to separate real edge from pattern-frequency
artefacts has never been run on crypto candlesticks.**

**Decay:** named patterns are the most publicly taught, most-coded-into-every-charting-platform
signals in existence. Any edge from slow price discovery is the single most likely to be gone. No
post-2023 crypto out-of-sample test showing persistence was found.

**Verdict: do not build decision code around named-pattern detection.**

---

## 2. Bar-level statistics (not patterns)

These are **continuous features**, and the distinction matters statistically: a continuous feature
in a walk-forward-validated model is not subject to the "pick the pattern that worked" trap.

    body_ratio = |close − open| / (high − low)                        ∈ [0,1]
    CLV        = [(close − low) − (high − close)] / (high − low)      ∈ [−1,1]
    upper_wick = (high − max(open,close)) / (high − low)
    run_length = consecutive same-direction closes

CLV is the standard input to Accumulation/Distribution and Chaikin Money Flow — an established
primitive, not folklore-only.

**Volume-at-price within the bar** needs trade-level data (which we have): the volume delta between
up-tick and down-tick trades composing the bar — i.e. **order-flow imbalance at bar resolution.**

**Honest state of the evidence:** the agent found **no rigorous academic paper isolating CLV or
body-ratio as a standalone predictor with an out-of-sample Sharpe**, crypto or equity. What is
better supported is **order-flow imbalance**, because it has a real mechanism behind it (informed
trader price impact — Kyle's lambda, and the imbalance-bar literature) that named candlestick
patterns never had beyond "traders believe in them."

**So: is "bar statistics beat named patterns" true? Partially, and for a specific reason** — they
avoid the definitional-arbitrariness problem and order-flow imbalance has a mechanism. But
**UNVERIFIED as a standalone cited crypto backtest. Mechanistically more plausible, empirically
thinner than one would like to claim outright.**

---

## 3. Multi-timeframe alignment — the statistics are against the folk practice

**Confirmed, and it is a mathematical fact rather than a caveat:** nested timeframes computed from
the *same* price series with the *same* formula (RSI on 1m/15m/1h) are **not three independent
confirmations.** A 1h bar is a deterministic aggregation of sixty 1m bars, so the 1h condition is a
function of the same data. **Requiring "1m AND 15m AND 1h" of the same indicator builds one slower,
smoother version of the 1m signal and reports inflated confidence by treating correlated evidence
as independent.** It is the multiple-testing error inverted — double-counting one test as three.

**The one paper directly on point** — Goswami, *Multi-Timeframe Signal Confirmation in Algorithmic
Cryptocurrency Trading: A Backtest Study on ETH/USDT* (SSRN 6683818) — **could not be fetched
(HTTP 403).** Its circulating numbers ("67% win rate vs 56%", "filters 30-40% of false signals")
are from a secondary summary; methodology, fee treatment and sample are all unconfirmed.
**UNVERIFIED — treat as not-evidence.**

**The design constraint that does follow, from first principles:** genuine independence across
timeframes requires **different signal types per timeframe** — e.g. a funding/basis regime on 1h
and order-flow imbalance on 1m — **not the same indicator restated at three resolutions.**

**And a diagnostic worth keeping:** if adding timeframe confirmation improves Sharpe, check whether
it is genuinely filtering noise or just producing **fewer, bigger-edge trades.** Those look the same
in a headline number and are different claims.

---

## 4. Which timeframe to trade — the concrete crypto measurements

**Turn-of-the-candle effect in Bitcoin** (Aharon et al., peer-reviewed, PMC10015199): using
**1-minute** returns, returns are positive and concentrated **at the open minute of each 15-minute
candle** (minutes 0, 15, 30, 45) at **+0.58 bps/minute average**, while other minutes average
negative. Sample 2013-2021 across **7 exchanges**, t-stats > 9.0 in 2021. **The effect only clearly
emerges from mid-to-late 2020 — it is a recent-regime phenomenon, not a stable multi-year one.**
Reported to survive costs: **74.18% annualised net vs 60.27% buy-and-hold, net of fees and spread.**

**This is the single most concrete "1m carries real, fee-surviving signal" result found — but it is
about candle-boundary *timing*, not candle *shape*.** It is an order-clustering effect (algos
executing at round clock times), structurally closer to §5 than to candlestick patterns.

**Wen, Bouri, Xu, Zhao (2022)** (also in file 04) — Bitcoin, 5-min aggregated to hourly, 2013-2020:
**both intraday momentum and reversal coexist**, sign and magnitude depending on regime (jumps,
FOMC, liquidity, COVID). **The paper's own title — "momentum, reversal, or both" — is an admission
that the sign flips.**

**No source gave a clean Sharpe-by-bar-interval comparison (1m/5m/15m/30m/1h) for crypto perps net
of realistic fees. That is a genuine gap, flagged as an absence rather than guessed at.**

**Honest synthesis: treat 1m as an execution/entry-timing timeframe, not a standalone
signal-generation timeframe**, absent a specific mechanism (like turn-of-candle) with its own
fee-survival evidence.

---

## 5. Support/resistance, round numbers, session opens, volume profile — in a 24/7 market

**Round numbers — the precise finding matters.** **Urquhart (2017), *Price Clustering in Bitcoin*,
*Economics Letters* 159:145-148** (peer-reviewed), Bitstamp daily closes 2012-2017: **found
significant clustering** ("00" endings over-represented) — **but explicitly found NO significant
pattern of returns following a round number.**

> **Clustering (an order-placement fact) ≠ predictability (a returns fact).**

Follow-up work confirms clustering intraday and across Litecoin/Ether/Ripple, and that it
**strengthens on higher timeframes**. **No paper testing round numbers as return-predictive
support/resistance in crypto perps was found.**

**24/7 is a genuine structural difference.** Essentially all session-open, prior-day-high/low and
volume-profile (POC / value-area) evidence comes from **regulated futures with a defined session**
(the cited 62% win rate / 800 trades / 4.2 ES points figure is CME ES, practitioner blog tier).
**Nothing found tests volume-profile POC on a 24/7, fragmented, multi-venue crypto perp market —
and the practitioner source itself flags the reason: futures volume is centrally reported, while
crypto volume is exchange-specific.** The mechanism that makes POC work on CME (everyone sees the
same complete tape) is **weaker or absent** when Binance and Bybit each see only their own flow.

**"Session opens"** do not map to 24/7. The closest analogues are **funding-interval boundaries
(every 8h)** and **UTC 00:00 rollover** — neither found directly tested as support/resistance.

**Verdict: use round numbers and prior highs/lows for *execution* — stop placement, limit
placement — not as directional signals.**

---

## 6. Bar aggregation alternatives

Definitions from **López de Prado, *Advances in Financial Machine Learning* (2018), Ch. 2**:
tick bars (every N trades), volume bars (every V units), **dollar bars** (every $D notional —
recommended as most robust because a fixed volume threshold goes stale as price moves), and
**imbalance bars** (bar closes when signed order-flow imbalance exceeds an EWMA-estimated threshold).

**Crypto evidence:**
- arXiv:2608.26158, BTCUSDT perp on Binance, Jan 2020–Dec 2025 (**non-peer-reviewed preprint**):
  tick-based bars substantially reduce lag-1 autocorrelation vs 1-minute bars (|AC(1)| 0.013 vs
  0.044) and show smaller variance-ratio deviation from a random walk. **But when frequency-matched
  to the same bar count, statistical independence is largely recovered — the paper's own conclusion
  is that the advantage comes mainly from finer resolution, not from the sampling scheme.**
  **It reports no trading-performance results at all.**
- *Algorithmic crypto trading using information-driven bars, triple barrier labeling and deep
  learning*, **Financial Innovation 2025 (peer-reviewed)** — compares information-driven bars
  against time bars for BTC/ETH label quality, tick data 2018-2023. **Full text not fetched;
  magnitudes UNVERIFIED.**

**Honest synthesis:** the statistical case (better-behaved returns, closer to i.i.d.) is real and
mechanistically sound — match sampling to information arrival, not the clock. **Whether it improves
actual trading performance in crypto perps is not demonstrated by anything found.** Since we
already ingest individual trades, **dollar/volume bars are cheap to try; imbalance bars cost more
(EWMA threshold estimation) for a benefit evidenced only as "cleaner statistics", not "more money".**

---

## Cross-cutting: the multiple-testing tools to actually use

- **White's Reality Check (2000)** and **Bailey & López de Prado's Deflated Sharpe Ratio (2014)** are
  the standard corrections for exactly our risk: testing many pattern/threshold/timeframe
  combinations and reporting the best.
- **Harvey, Liu & Zhu's "factor zoo" recommendation: require t-stat > 3.0, not 2.0**, given how much
  of finance has already been mined.
- **Every candlestick-positive result found reports the best of a handful of tested patterns with no
  deflated-Sharpe or bootstrap-null correction. That is the textbook shape of a multiple-testing
  artefact** — not proof it is fake, but proof the number cannot be taken at face value.

---

## What to build, in order of evidence strength

1. **Order-flow imbalance at bar resolution** — we have trade-level data and the mechanism is
   established microstructure theory, even without a clean isolating crypto backtest.
2. **Round-number and prior high/low proximity as *execution* inputs** (stop and limit placement),
   never as directional signals — Urquhart's own test found no return edge.
3. **Dollar/volume bars as the base sampling unit** instead of time bars — cheap given we have ticks,
   with real if modest and statistical-only evidence.
4. **Multi-timeframe features built from *different measurement types* per timeframe**, not one
   indicator restated three times.

**Do not build named candlestick pattern detection.**

---

## UNVERIFIED

1. **SSRN 6683818 multi-timeframe figures** (67% vs 56%, 30-40% filtering) — HTTP 403; numbers from a
   secondary summary, methodology/fees/sample unconfirmed, cannot even confirm they are from that paper.
2. Crypto Engulfing PF 3.54 — ResearchGate listing/summaries only; methodology unconfirmed.
3. Forex hammer/hanging-man "consistently profitable" — secondary summary only.
4. Hammer 59.86% / Bullish Engulfing 54.35% / Bearish Engulfing 42.39% accuracy — blog aggregation,
   source paper not identified.
5. Turn-of-the-candle "74.18% vs 60.27%" — confirmed via the PMC page, but **the exact trading rule
   producing it was not reproducible** from what was retrieved.
6. *Financial Innovation* 2025 information-driven-bars result magnitudes — existence confirmed, full text not fetched.
7. arXiv:2608.26158 — non-peer-reviewed preprint; numbers fetched directly (higher confidence) but
   scope is statistical properties only, no P&L.
8. Volume-profile "62% win rate, 800 trades, 4.2 ES points" — practitioner blog, CME ES, not crypto.
9. **No source tested Sharpe-by-bar-interval systematically net of realistic crypto perp fees.**
   Flagged as an absence, not guessed.

## Sources
- Marshall, Young & Rose (2006), *Market Timing with Candlestick Technical Analysis*, SSRN 980583 / Pacific-Basin Finance Journal
- Urquhart (2017), *Price Clustering in Bitcoin*, Economics Letters 159:145-148
- Aharon et al., *Turn-of-the-candle effect in bitcoin returns* — <https://pmc.ncbi.nlm.nih.gov/articles/PMC10015199/>
- Wen, Bouri, Xu, Zhao (2022), NAJEF
- *Intraday patterns of price clustering in Bitcoin*, Financial Innovation (Springer)
- López de Prado (2018), *Advances in Financial Machine Learning*, Ch. 2
- arXiv:2608.26158, *A Frequency-Controlled Comparison of Tick- and Minute-Based Information Bars*
- *Algorithmic crypto trading using information-driven bars, triple barrier labeling and deep learning*, Financial Innovation 2025
- Bailey, Borwein, López de Prado, Zhu, *The Probability of Backtest Overfitting*
- White (2000), *A Reality Check for Data Snooping*
- Goswami, SSRN 6683818 (**could not fetch — 403**)
