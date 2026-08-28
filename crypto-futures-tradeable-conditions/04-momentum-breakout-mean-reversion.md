# Momentum, breakout, mean reversion — and how to tell which regime you are in

**Researched by:** a `sonnet` research-web subagent, **2026-08-28**.
**Constraints given:** single-symbol directional bot, minutes-to-days, taker fees 4-5.5 bps/side
(a 1m signal must clear ~10 bps round trip). Cross-symbol pairs excluded (covered in file 05).

---

## The scope mismatch that changes every answer

**Almost all rigorous, citable crypto momentum evidence is cross-sectional** — long winners /
short losers across hundreds-to-thousands of coins, weekly rebalanced. **Our bot is
single-symbol, directional, minutes-to-days.** Most of what is "proven" in this literature is
**not directly actionable in our architecture**, and the agent flagged where each finding
transfers and where it does not.

---

## 1. Time-series momentum (single symbol)

**Dobrynskaya (2021), *Cryptocurrency Momentum and Reversal*** — J/K sorts (Jegadeesh-Titman) on
~2,000 coins, weekly, 2014-2020. Primary PDF read directly:

| Sort J / hold K | Result |
|---|---|
| J=1 week, K=1 week | 40% p.a., **statistically insignificant** |
| **J=2, K=2 (best)** | **70% p.a., significant** |
| up to ~2-4 weeks | positive and significant |
| ~4-6 weeks | **crosses to zero** |
| beyond ~8-13 weeks | **significant large negative — reversal**, >100% p.a. at 6-8 week rebalancing, reaching ≈ −1,200% p.a. in the most extreme combination |

**These are gross returns. The agent read the full paper and found no transaction-cost-adjusted
table.** Treat "momentum works in crypto" from this paper as **UNVERIFIED net of costs.**

**CTREND (Fieberg, Liedtke, Poddig, Walker, Zaremba, *JFQA* 2025)** — primary source read.
Aggregates 28 technical indicators (7 SMAs from 3d to 200d, MACD, RSI, stochastics, Bollinger,
volume signals) into one cross-sectional signal; weekly rebalance, 2015-2022, >3,000 coins,
min $1M market cap. **≈ 3.87%/week long-short, Sharpe ≈ 2.0 annualised** (CMOM momentum factor
alone ≈ 1.2, market ≈ 1.2). Directly quoted: *"Despite the short-term nature of the CTREND
trading signal and significant portfolio turnover, the profits generated are resilient to
transaction costs."* **This is the best net-of-cost crypto evidence found for anything in the
trend family — and it is a broad cross-sectional signal, not a single-symbol rule.**

**Horizon that actually shows up: 1-2 weeks continuation, flipping to reversal by roughly one
month.** Much faster than equities/futures TSMOM (12-month lookback, ~1-year reversal) —
Dobrynskaya frames this as crypto's "faster metabolism."

**What invalidates it:** holding past ~4-6 weeks flips the sign. **A signal on a 1-4 week
lookback that does not enforce a hold-period cap will bleed into the reversal regime.**

**Decay:** the Bitcoin market-microstructure maturation study (ScienceDirect, 2012-2025, three
eras) found via variance-ratio and Hurst tests that Bitcoin's return process has moved
**progressively closer to a random walk** across 2012→2025 — the exploitable serial correlation
this whole family depends on has been shrinking as the market institutionalised.

**Verdict: do not build a single-symbol trend/MA-crossover rule expecting the cited Sharpes —
they were not measured on that object.**

---

## 2. Cross-sectional momentum — real, but architecturally not ours

Summarised only for completeness. Liu & Tsyvinski (2021, *RFS*) and Liu, Tsyvinski & Wu (2022,
*Journal of Finance*) establish the market/size/momentum 3-factor "LTW" model, 1-4 week
formation, ~3% weekly excess on long-short momentum. **CTREND beats it and "renders the momentum
effect insignificant while not being subsumed by any other factors"** — the technical-indicator
composite dominates raw past-return momentum in a horse race.

**Skip:** needs basket trading and breadth (hundreds+ of names) we do not have at 30-50 symbols.

---

## 3. Breakout conditions — the evidence is bad, and one rigorous test says so directly

**Mesfin (2026), *Structural Limits of OHLCV-Based Intraday Signals in MNQ Futures: A Systematic
Falsification Study*, arXiv:2605.04004** — independent researcher, **not peer-reviewed**, but
methodologically the most rigorous thing found: walk-forward train/test split, permutation
testing, positive controls, 2-point (~$4) round-trip friction, 947 trading days 2021-2025.
Primary PDF read directly.

- **Opening range breakout (long and short, immediate and delayed): FAIL on every variant.**
  Best case (15-bar delayed long) T=1.50 against a 2.0 threshold. Broken out by year:
  **2022 = −1.42, 2023 = +2.43, 2024 = +7.04 net — "one strong year masking flat or negative
  other years… the most common failure mode across the whole study."**
- **Pullback entry was worse: 80.7% stop-out rate.** *"Most apparent breakouts just fail and
  reverse. The pullback entry captures that reversal, not a continuation."*
- **Volume-spike confirmation: T-stats near zero on N=2,000-2,400 each — "precise null results,
  not inconclusive ones."** Bar-level volume magnitude did not predict next-bar direction.
- **Core finding: across all 14 signal families, gross edge before friction was 0.07-1.5 points
  per trade against a 2.0-point friction floor. Every family failed** — not from bad luck but
  because "what they measure is too small to survive realistic execution costs." **Two positive
  controls passed cleanly, confirming the methodology can detect real edge when it exists.**

Index futures, not crypto, and a single independent manuscript — **but it directly falsifies the
exact rule family (ORB, volume-confirmed breaks) using realistic costs and out-of-sample
discipline that almost nothing else in this space has.** Crypto's relative fees are *higher* than
MNQ's effective friction.

**The circulating crypto false-breakout rates (60-70%, ~65% on BTC futures) come only from
trading-education blogs with no methodology, no sample, no cost treatment. Folklore, not
evidence. Do not cite them in our design docs.**

**Verdict: skip retail breakout rules as designed.** If breakout logic is used at all it needs a
volatility-compression precondition (itself only a blog claim) and must be tested with the same
walk-forward + permutation + cost discipline Mesfin uses.

---

## 4. Short-horizon mean reversion after a sharp move — the weakest section

- **Wen, Bouri, Xu & Zhao (2022), *Intraday return predictability in the cryptocurrency markets:
  momentum, reversal, or both*, North American Journal of Economics and Finance** — Bitcoin,
  5-minute aggregated to hourly, 2013-2020. Finds **both** intraday momentum and reversal, the
  pattern shifting around large intraday jumps, FOMC announcements, and liquidity regime.
  **Peer-reviewed and real, but the agent could not get past the abstract — exact reversal
  magnitude and half-life are UNVERIFIED.**
- **Dobrynskaya's reversal operates at weeks-to-months — the wrong timescale for "fade the
  flush."** Flagged explicitly so it is not misapplied.

**What is entirely absent:** any peer-reviewed or rigorously backtested study of reversal
magnitude and time-decay after a liquidation cascade. Everything found (exchange blogs, Medium,
Mudrex, Bitsgap, Amberdata, PrimeXBT) describes the mechanism qualitatively — forced selling
begets forced selling, price overshoots, then reverts once a large passive bidder steps in —
**with no backtested edge, no holding horizon, and no net-of-fee number.**

**This is the one area where we must generate our own evidence.** And 6 days of tape is nowhere
near enough: cascades are rare tail events.

---

## 5. Deciding which regime you are in — the section to get right

**Hurst exponent: mostly folklore for live decisions.** From the academic critique (Springer 2025,
*Quality & Quantity*): **"H ≠ 0.5 for financial returns is perfectly compatible with the random
walk model"**; estimator variance is high and algorithm-dependent (R/S vs DFA); and critically
**"a single full-sample H answers a question nobody trades: was this series mean-reverting on
average over eight years?"** If used at all: **rolling-window DFA, not R/S** (R/S is more
sensitive to short samples and drift), treated as a slow noisy prior, never a trigger.

**Variance ratio (Lo-MacKinlay 1988) — the more rigorous, and the one with crypto evidence:**

    VR(q) = Var(r_t + r_{t−1} + … + r_{t−q+1}) / (q · Var(r_t))

VR > 1 → positive autocorrelation (trending); VR < 1 → mean-reverting; VR = 1 → random walk.
The Bitcoin maturation paper applied **rolling 5-day VR on 1-minute Bitcoin data** and found the
market moving **progressively toward VR ≈ 1** as it institutionalised. **That is the honest
crowding answer: the signal is real and measurable, but its magnitude has been declining, and
2021-2025 is closer to efficient than 2012-2020 was.**

**ADX: no credible academic backing found.** Everything is retail backtest content with
**contradictory results even within the same search** — one source says trend-filtering
"consistently underperformed direct DI crossover," another claims 28% outperformance with no
methodology or costs. **Treat ADX as folklore.**

**Return autocorrelation** is really the same test as VR at lag 1 and is the most defensible thing
to compute live — simple, known null distribution.

**Recommended: build the regime switch on rolling variance-ratio / lag-1 autocorrelation over a
short window, cross-checked by DFA on a slower window. Skip ADX and static-sample Hurst.**

**The caveat that matters:** none of the sources runs a VR-based regime switch *as a trading rule*
net of crypto perp fees at our horizon. The maturation paper measures efficiency as a research
question, not a signal generator. **"VR(q) below X → mean-revert, above Y → trend" is a reasonable
hypothesis grounded in a real statistical test, not a proven strategy. The window is not
principled by anything found — it must be fitted on our own tape, and that must be said.**

---

## Skip outright

- Retail ORB / volume-confirmed breakout rules.
- **ADX as a regime classifier.**
- **Static/full-sample Hurst as a live signal** — the literature says it answers the wrong question.
- Cross-sectional momentum machinery (LTW, CTREND) — real, well-evidenced, architecturally not ours.
- The 60-70% false-breakout figures.

## Worth building, with caveats attached

- A **short-lookback continuation signal with a hard hold-period cap** at the point where the sign
  flips — but the underlying evidence is gross-of-cost and cross-sectional. **Nothing in this
  literature hands us a principled lookback for a single perp; re-derive from our own tape.**
- A **rolling variance-ratio / autocorrelation regime gate** — the one regime tool with real
  methodology and a crypto-specific empirical result. Fit the parameters ourselves and say so.
- **Do not build a liquidation-flush mean-reversion rule yet.** Plausible, widely described,
  unquantified anywhere rigorous, and untestable on 6 days of tape.

---

## UNVERIFIED

1. CTREND Sharpe values (2.0 / 1.8 / 1.2) — **read off a bar chart (Figure 1)**, not a results
   table. Ordering solid, decimals are an eyeball estimate.
2. Wen et al. (2022) intraday magnitudes — abstract level only; no exact numbers, lookbacks, or
   net-of-fee results.
3. Grayscale "10-30 day MA crossover has the highest Sharpe" — secondary/practitioner, not verified.
4. **Liquidation-cascade reversal size and time-decay — no rigorous quantification found at all.**
5. arXiv:2601.06084 (*Who sets the range? Funding mechanics and 4h context*) — PDF extraction failed
   twice; abstract reads as theoretical framing, not a quantified test. **Do not cite its claims.**
6. Retail false-breakout rates — named only to dismiss.
7. Dobrynskaya net-of-cost figures — **stated as an absence, not an inferred number.**
8. Exact VR(q) trajectory 2012-2025 — qualitative finding from summaries; paywalled abstract only.

## Sources
- Fieberg, Liedtke, Poddig, Walker, Zaremba, *A Trend Factor for the Cross Section of Cryptocurrency Returns*, JFQA 2025 — primary PDF read
- Dobrynskaya (2021), *Cryptocurrency Momentum and Reversal*, HSE — primary PDF read
- Mesfin (2026), *Structural Limits of OHLCV-Based Intraday Signals in MNQ Futures*, arXiv:2605.04004 — primary PDF read
- Liu & Tsyvinski, *Risks and Returns of Cryptocurrency*, NBER w24877 / RFS
- Liu, Tsyvinski & Wu, *Common Risk Factors in Cryptocurrency*, NBER w25882 / J. Finance
- Wen, Bouri, Xu, Zhao (2022), NAJEF — <https://www.sciencedirect.com/science/article/abs/pii/S1062940822000833>
- *Maturation of Bitcoin market microstructure (2012-2025)* — <https://www.sciencedirect.com/science/article/pii/S221484502600089X>
- *Random walks, Hurst exponent, and market efficiency*, Quality & Quantity, Springer 2025
- Lo & MacKinlay (1988), variance ratio test
