# Volatility estimation, forecasting, and regime conditions

**Researched by:** a `sonnet` research-web subagent, **2026-08-28**.
**Constraints given:** no IV surface for most symbols (so anything IV-based is BTC/ETH only);
~6 days of tape; 1m bars, trades, partial-depth books, quotes, mark/index/funding.

**The agent checked every recommended window length against the 6 days we actually have, and
several standard windows do not fit yet. That is the most immediately useful thing in here.**

---

## 1. Which volatility estimator

**Verdict: for 1-minute crypto bars, skip Garman-Klass and Parkinson as the primary estimator.**

- **Realized variance (use this intraday):** `RV_t = Σ r²_{t,i}`, log returns of consecutive
  closes. Annualise as `√(RV_t · periods_per_year)`. This is the discretized quadratic
  variation every realized-vol/HAR paper builds on.
- **Parkinson (1980):** `σ²_P = 1/(4ln2) · (1/n) Σ (ln(H_i/L_i))²` — high-low only, assumes
  zero drift and no opening jump.
- **Garman-Klass (1980):** `σ²_GK = (1/n) Σ [0.5(ln(H/L))² − (2ln2−1)(ln(C/O))²]`.
- **Rogers-Satchell (1991):** `σ²_RS = (1/n) Σ [ln(H/C)ln(H/O) + ln(L/C)ln(L/O)]` —
  drift-independent, still not jump-robust.
- **Yang-Zhang (2000):** `σ²_YZ = σ²_O + k·σ²_C + (1−k)·σ²_RS`, with
  `k = 0.34 / (1.34 + (n+1)/(n−1))`. The only one both drift-independent **and** jump-robust —
  but its overnight term is meaningless for a 24/7 perp unless "overnight" is redefined (across
  the funding timestamp, or a UTC day boundary).

**Why not the range estimators at 1m:** the dominant contaminant at that frequency is
**bid-ask bounce / tick discreteness**, not drift or an overnight gap. The signature-plot
effect (Zhang, Mykland, Aït-Sahalia 2005, *A Tale of Two Time Scales*, JASA) means RV from raw
1-second/1-minute data is **biased upward** by microstructure noise and increases with sampling
frequency instead of converging. Range estimators are **more** noise-sensitive than realized
variance at high frequency (Hansen & Lunde), and on exchange-supplied 1m candles the bar's
high/low is itself a noisy quantity.

**What to build:**
1. **5-minute close-to-close RV** as the live intraday vol measure — not 1-minute.
2. **Yang-Zhang only at daily aggregation**, built from our own 1m data, as the jump-aware
   daily series feeding HAR/GARCH later.

Skip Parkinson and Garman-Klass as production estimators — they exist mainly because RS and YZ
postdate them.

**Minimum sample:** a single day's RV from 1m bars uses ~1440 observations and is already fine —
**RV precision comes from within-day return count, not calendar days, so this is usable today.**
What needs accumulation is anything modelling the *time series* of daily RV.

**Practical break conditions:** stale/duplicate OHLC ticks on lower-volume symbols (check for
duplicate timestamps before trusting any range estimator); funding-settlement wicks that are not
real traded volume; liquidation-cascade range spikes that are real but unrepresentative.

---

## 2. Forecasting: HAR-RV, GARCH, EWMA

**HAR-RV (Corsi 2009, *J. Financial Econometrics*) — the model to build first, once we can:**

    RV_{t+1d} = β₀ + β_D·RV_t + β_W·RV_{t|t−4} + β_M·RV_{t|t−21} + ε

with `RV_{t|t−4}` the 5-day average and `RV_{t|t−21}` the 22-day average. A constrained AR(22):
three parameters instead of 22, exploiting long memory that AR(1) and GARCH(1,1) badly underfit.

**It needs at least 22 days of daily RV to populate the monthly term at all, and the literature
typically uses 250+ days for stable coefficients. We have ~6 days — HAR-RV cannot be estimated
yet, full stop.** What we can do today: start accumulating the daily RV series; run a first cut
at ~30 days; treat anything before ~90-120 days as provisional.

**GARCH(1,1)/EGARCH:** `σ²_t = ω + α·ε²_{t−1} + β·σ²_{t−1}`. Needs 100+ observations for a
stable MLE; small-sample fits are notoriously unstable (boundary solutions with α+β ≈ 1,
non-convergence). **Not usable off 6 days.**

**EWMA:** `σ̂²_t = λ·σ̂²_{t−1} + (1−λ)·r²_{t−1}`. **λ = 0.94 is a RiskMetrics commercial
convention, not a number derived from our data — under RL-061 either estimate λ from our own
tape or cite RiskMetrics explicitly as its provenance. Do not hardcode it as if derived.**

**What the evidence says, and where it conflicts.** Two Bitcoin-specific strands that are
**not apples-to-apples** and should not be cited against each other:
- GARCH-family **on returns**: PLOS ONE, *predictive capacity of GARCH-type models* — EGARCH(1,1)
  beats GARCH(1,1) and EWMA in and out of sample. The arXiv "Horserace" paper (arXiv:2010.07402)
  agrees GARCH/EGARCH beat historical/EMA models, then builds a GARCH-vs-implied-vol spread trade
  with delta hedging — **needs an options market, BTC-only, out of scope for us.**
- HAR **on realized volatility**: multiple search hits report **HAR beating GARCH for Bitcoin RV
  forecasting**. **The magnitude of HAR's edge is UNVERIFIED — only the directional claim is
  corroborated, from summaries, not from a table the agent read.**

A notable substantive finding: the EGARCH asymmetry term came out **positive and insignificant**
in those samples, meaning **Bitcoin does not show the equity-typical leverage effect** — bad news
does not spike vol more than good news.

**Do not import anyone's fitted crypto GARCH/HAR coefficients.** The published ones are from
2017-2021 samples that do not represent 2025-2026 market structure. Re-estimate on our own tape.

---

## 3. Regime classification — and the lag problem

**The honest fact to design around:** an HMM (or any smoothing-based classifier) used live
reports a **filtered** state, which by construction only becomes reliable a few observations
*after* the true regime changed. "The model correctly identified the crisis regime" in a backtest
is very often measuring identification *after* the crisis moved price.

Three approaches, most to least defensible for us:

1. **Threshold on rolling vol percentile — build this first.** RV (or EWMA vol) against its own
   trailing distribution: below the 33rd percentile = compressed, 33rd-80th = normal, above the
   80th = expansion/crisis. **Fully mechanical, known lag (the window width), zero fitting,
   works from day one.** Trending-vs-ranging additionally needs a direction signal (high vol with
   signed-return autocorrelation near zero = chop; elevated vol with strong same-sign
   autocorrelation = trending) — that crosses into the momentum area, so keep the vol output as a
   gate/feature.
2. **Markov-switching / HMM**, 2-4 states, EM/Baum-Welch. Conservatively 10-20 free parameters;
   numerically fittable on 6 days of 1m data, but the **regime-persistence estimates will be
   unreliable until multiple real transitions have been observed** — weeks to months.
   *All three sources found for this were vendor/blog, not peer-reviewed.*
3. **Rolling Hurst exponent** — H > 0.5 trending, H < 0.5 mean-reverting. Needs 100+ observations
   for stability, noisy and slow-adapting. A candidate feature, not a primary classifier.

**No crypto-specific paper quantifying HMM regime-detection lag was found.** The agent
deliberately attached **no borrowed number** to it. **The only honest way to know our
classifier's lag is to backtest strictly causally — filtered probabilities only, refit on past
data at each point — and measure the gap between the true state change and the threshold
crossing.** That measured gap is what belongs in our own docs.

**Where it breaks hardest:** a genuinely new regime (a newly-listed symbol's first flash crash)
has no prior state to be classified into; the model force-fits it into the nearest existing
state, or needs a refit that is too slow to matter live.

---

## 4. Vol-of-vol and compression/expansion — treat with more skepticism than §1-§3

- **Vol-of-vol:** apply the RV formula a second time, to the *series of RV values*:
  `VoV_t = std(RV_{t−k} … RV_t)`.
- **Compression ratio:** `ratio_t = RV_7d / RV_30d`. Well below 1 = compressed (squeeze
  candidate); well above 1 = expansion under way. Source: an Amberdata **data-vendor blog** —
  the ratio construction is mechanically sound, its predictive claims are not verified.
- **BB/KC squeeze:** "squeeze on" when `2σ_20 < 1.5·ATR_20`. **The 1.5 and 2.0 are conventions,
  not derived constants** — under RL-061 they must be estimated and justified against our own
  data, not imported as magic numbers.

**The evidence here is bad and should be named as such.** The circulating backtest figures
("squeeze-to-fire → 2x average daily range within 5 bars ~68% of the time"; "35-40% vs 55-65%
breakout failure rates") come from trading-education vendor sites, tested on S&P 500 components,
not peer-reviewed, with no reproducible methodology. The one crypto item reports Sharpe > 1.0
from a **243-parameter-combination optimization sweep with no out-of-sample holdout** — close to
the textbook definition of overfitting. **No peer-reviewed academic paper on vol-of-vol or
squeeze cycles in crypto perps was found.**

**What is genuinely uncontroversial:** volatility mean-reverts, so abnormally low RV tends to be
followed by reversion toward the mean, which given vol clustering often appears as expansion.
**What is unverified is that a squeeze predicts the direction or magnitude of the ensuing move**
beyond that trivial fact.

**Also:** on a thin symbol a "squeeze" can be an artefact of near-zero trading, and the
"breakout" that follows is a liquidity event — one large order moving an empty book — not a
volatility regime signal.

**Build the compression ratio as a gating feature. Do not build a squeeze entry signal on the
strength of these sources.**

---

## 5. Vol regime as a gate on other strategies — the best-evidenced claim in this report

**Daniel & Moskowitz, *Momentum Crashes*, Journal of Financial Economics 122 (2016) 221-247.**
Momentum crashes are "partly forecastable," occurring in **panic states — following market
declines, when market volatility is high**, and contemporaneous with market rebounds. They build
a dynamic strategy scaling exposure by a forecast of momentum's own conditional mean and
variance, and show it materially reduces crash risk versus static momentum.

**The direct implication, and the single most actionable fact found:** a momentum/trend strategy
should have its size **reduced, not increased, when realised vol is elevated AND the market has
just declined.** The naive intuition — high vol means strong trend, so size up momentum — is
**backwards for exactly the state that produces the worst outcomes.**

| Strategy family | Vol regime that helps | Vol regime that hurts |
|---|---|---|
| Trend / momentum | Moderate, smoothly *rising* vol with directional persistence | Vol spike after a decline (the crash state); also very low/compressed vol (no signal-to-noise) |
| Mean-reversion / range | Low-to-moderate, range-bound | Crisis/expansion — a reversion entry into a real breakout is the textbook way to get run over |
| Breakout | The compression → expansion transition | Already-expanded persistent-high-vol — the move already happened |

**Only the momentum row is sourced.** The other two rows are the agent's structural inference and
must be validated on our own tape.

**Crucially this cannot be one global on/off switch:** the state that protects momentum is exactly
the state a reversal strategy wants to enter into. It has to be a **per-strategy-family gate.**

**"A regime filter is often worth more than a signal" is reasoned inference, not a measured
number** — no crypto paper quantifying how much strategy variance a vol gate removes was found.
The real number comes from backtesting our own gate.

**Horizon caveat:** Daniel-Moskowitz is monthly-rebalance equity momentum. The *mechanism*
(crowded levered directional bets unwind violently in vol spikes) plausibly transfers — perp
funding and liquidation cascades arguably sharpen it — but **the specific thresholds do not.
Import the gate direction, never the numbers.**

---

## What to skip

- All five OHLC estimators — build two (5-min RV intraday, Yang-Zhang daily).
- GARCH and HAR-RV in parallel on day one — no history for either to mean anything. EWMA is the interim.
- A 4-state HMM before observing one real regime transition on our own tape.
- A squeeze-breakout *entry* signal off these sources.
- **Build first: the vol-regime gate on momentum exposure (§5)** — the one claim backed by a
  peer-reviewed, on-point result rather than inference or vendor content.

---

## UNVERIFIED

1. Yang-Zhang's "~8x more efficient" figure — secondary source, not re-derived from the 2000 original.
2. **No crypto-specific head-to-head estimator comparison found** — the recommendation against
   range estimators at 1m is inference from the equities/FX noise literature.
3. The "5-minute rule" for optimal sampling — carried over from equities/FX; no crypto-specific study.
4. Exact HAR-vs-GARCH numbers on Bitcoin (R², QLIKE, MSE) — PDF fetch failed; only the abstract and
   search summaries.
5. HMM regime lag in bars/days for crypto — no paper found; no borrowed number attached, deliberately.
6. All squeeze/BB-KC statistics (68% hit rate, 35-65% failure rates, Sharpe > 1.0 from a 243-combination
   sweep) — vendor blogs, equities except the one crypto item, which shows overfitting signs.
7. The 7d/30d RV ratio as a regime signal — data-vendor blog; construction sound, predictive claims unverified.
8. Magnitude of the momentum-crash effect transferred to crypto perps at minutes-to-days —
   mechanism reasoned to transfer, no crypto replication found.
9. *Market volatility, momentum, and reversal: a switching strategy* (J. Asset Management 2024) —
   search summary only.
10. "Range estimators are fairly robust to microstructure noise" — search synthesis, contradicts the
    better-supported Hansen-Lunde finding; resolved by recommending against them at 1m regardless.

**Quota note:** native WebSearch throughout, no quota limit hit; one PDF fetch failed and the agent
fell back to the abstract page rather than to memory.

## Sources
- Corsi (2009), *A Simple Approximate Long-Memory Model of Realized Volatility*, J. Financial Econometrics
- Daniel & Moskowitz (2016), *Momentum Crashes*, JFE 122:221-247 — <https://www.nber.org/papers/w20439>
- Andersen, Bollerslev, Diebold, Labys (2001), *The Distribution of Realized Exchange Rate Volatility*, JASA
- Zhang, Mykland, Aït-Sahalia (2005), *A Tale of Two Time Scales*, JASA
- Garman & Klass (1980) — <https://www.cmegroup.com/trading/fx/files/a_estimation_of_security_price.pdf>
- Shu & Zhang (2006), *Testing range estimators of historical volatility*, J. Futures Markets
- PLOS ONE, *predictive capacity of GARCH-type models* — <https://journals.plos.org/plosone/article?id=10.1371/journal.pone.0245904>
- arXiv:2010.07402 (GARCH/EGARCH horserace, BTC spot and options)
- arXiv:2205.11122 (Hurst + Q-learning; abstract level only)
