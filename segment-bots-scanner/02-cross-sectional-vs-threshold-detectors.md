# Cross-sectional ranking vs per-symbol absolute thresholds

**Researched 2026-08-26 by a `sonnet` subagent (Rule 1). The Alpha101 operator
definitions and formulas below were extracted from the arXiv PDF with `pdftotext`,
not from a summarizer. Qlib class names were read from raw GitHub source.**

## Why ten of eleven detectors went silent — structural, not a tuning problem

A per-symbol z-score threshold assumes each symbol's distribution is independently
well-behaved and stationary enough that "3 sigma from its own history" is a meaningful
rarity. In a ~1,500-symbol perpetual universe dominated by one common factor, most
symbols spend most of their time moving *together* — so a symbol's own z-score rarely
detaches from where the whole market is, because the whole market moved with it.

**The threshold is not wrong; it is measuring "is this symbol unusual relative to
itself" when the tradeable fact is usually "is this symbol unusual relative to its
peers right now."**

## But "always trade the top decile" is a liquidity guarantee, not an edge guarantee

This is the honest read, and it is the finding that should change the plan:

Ranking guarantees that *some* symbols are in the top decile every period — it engineers
away the zero-signal problem **definitionally**, because rank is defined over whatever
universe is fed to it. It does **not** guarantee the top decile was separated from the
bottom by anything economically real.

If 80% of the universe is correlated junk, the top decile on any given tick is
disproportionately populated by whichever few names have the **thinnest order books**
and hence the noisiest short-horizon returns — not by names carrying genuine
idiosyncratic information.

## Alpha101 — the exact operator vocabulary (verbatim, Appendix A.2)

```
rank(x) = cross-sectional rank
delay(x, d) = value of x d days ago
correlation(x, y, d) = time-serial correlation of x and y for the past d days
covariance(x, y, d) = time-serial covariance of x and y for the past d days
scale(x, a) = rescaled x such that sum(abs(x)) = a (the default is a = 1)
delta(x, d) = today's value of x minus the value of x d days ago
signedpower(x, a) = x^a
decay_linear(x, d) = weighted moving average over the past d days with linearly decaying
                     weights d, d-1, ..., 1 (rescaled to sum up to 1)
indneutralize(x, g) = x cross-sectionally neutralized against groups g (subindustries,
                     industries, sectors, etc.), i.e., x is cross-sectionally demeaned
                     within each group g
ts_{O}(x, d) = operator O applied across the time-series for the past d days
ts_min(x, d), ts_max(x, d), ts_argmax(x, d), ts_argmin(x, d), ts_rank(x, d)
sum(x, d), product(x, d), stddev(x, d)
```
Inputs: `returns, open, close, high, low, volume, vwap, cap, adv{d}`, `IndClass.*`.

Representative formulas, verbatim:
```
Alpha#1:   (rank(Ts_ArgMax(SignedPower(((returns < 0) ? stddev(returns, 20) : close), 2.), 5)) - 0.5)
Alpha#2:   (-1 * correlation(rank(delta(log(volume), 2)), rank(((close - open) / open)), 6))
Alpha#3:   (-1 * correlation(rank(open), rank(volume), 10))
Alpha#4:   (-1 * Ts_Rank(rank(low), 9))
Alpha#101: ((close - open) / ((high - low) + .001))
```

**The structural lesson: Alpha101 is a hybrid, not a cross-sectional replacement.**
`ts_rank`, `delta`, `correlation`, `stddev` are per-symbol time-series operators; only
`rank()` is cross-sectional, and it is applied **last**, wrapping the time-series
feature to convert it to a universe percentile.

So the fix is **not** "replace time-series detectors with cross-sectional ones." It is
**"rank the output of the existing time-series detectors across the universe before
thresholding."** That preserves every detector already written.

## Qlib — usable as a reference, not installable here

`qlib/data/ops.py` operators (read from source): `Abs Sign Log Not Mask ChangeInstrument`;
`Power Add Sub Mul Div Greater Less Gt Ge Lt Le Eq Ne And Or`; `If`; rolling
`Ref Mean Sum Std Var Skew Kurt Max Min Med Mad IdxMax IdxMin Rank Count Delta Slope
Rsquare Resi Quantile WMA EMA`; pairwise rolling `Corr Cov`; `TResample`.

**Correction worth keeping: Qlib's `Rank` is a *rolling* (time-series) percentile, NOT
a cross-sectional rank.** Same name, different operator from WorldQuant's `rank()`.
Qlib does cross-sectional work in `CSRankNorm`/`CSZScoreNorm` processors, a different
layer. (UNVERIFIED — the processor file was not fetched this session.)

`pyqlib` on PyPI: `requires_python=">=3.8.0"`, wheels only to **cp312**, ships compiled
extensions. **Not installable on cp314 with no compiler.** `qlib/data/ops.py` is itself
pure Python and MIT — vendorable as a design reference, which is a different thing from
installing the package.

## Crypto-specific evidence

| Source | Type | Confirmed claim |
|---|---|---|
| Liu, Tsyvinski & Wu, *Common Risk Factors in Cryptocurrency*, J. Finance 2022 / NBER WP 25882 | Peer-reviewed | 3-factor (market, size, momentum) model explains cross-sectional returns of 10 characteristic-sorted long-short crypto strategies |
| Dobrynskaya, *Cryptocurrency Momentum and Reversal*, SSRN/HSE | Working paper, UNVERIFIED beyond abstract | Momentum to 2-4 weeks, reversal past ~1 month, 2wk/2wk sort ~70%/yr, 2,000 spot cryptos 2014-2020 |

**Both are daily-to-multi-week, spot, not perps.** No primary source was found studying
cross-sectional signal generation at tick frequency on perpetuals. Treat the literature
as evidence that cross-sectional structure *exists* in crypto, not that it survives at
this operating frequency.

## The honest counter-case — read this before implementing

- **Crypto perps are close to a single-factor market.** BTC-altcoin correlations
  reported in the 0.7-0.9 range, spiking toward 0.9 in stress (secondary sources; treat
  the exact figure as approximate). If ~90% of a symbol's return variance is the common
  factor, a raw cross-sectional rank of *returns* mostly ranks symbols **by their beta** —
  it rediscovers "which altcoins are high-beta today," not dislocations.
  **Beta-neutralise before ranking: regress out the common factor and rank the residual.**
  Skipping this is the single most likely way "adopt cross-sectional ranking" produces
  trades with *worse* Sharpe than the current all-zero detectors — zero signals cost
  nothing, a beta-driven top decile costs slippage and correlated drawdowns.
- **Top-decile-by-liquidity distortion.** Noise has fatter tails in thin names, so a
  naive rank over-selects them. Apply a liquidity/ADV filter **before** ranking, not after.
- **Junk symbols pollute a percentile rank** — they do not get excluded, they shift where
  the cutoffs fall.
- **Regime dependence does not disappear**, it moves upstream: in a crash, correlations go
  to ~1 and cross-sectional dispersion *collapses*, so a cross-sectional detector goes
  quiet at exactly the moment an absolute one does, for the same underlying reason.
- **Turnover.** Recomputing decile membership every tick over 1,500 symbols without a
  rebalance interval and hysteresis is a cost-generation machine.

## What not to bother with

- Do not hunt for a published Alpha101-on-crypto backtest. It does not exist in citable form.
- Do not try to `pip install pyqlib` on this box.
- **Do not treat `rank()` as sufficient.** It fixes the *volume* problem trivially and does
  nothing for the *edge* problem without factor-neutralisation. Cross-sectional ranking
  without neutralisation in a one-factor universe is a documented way to build a leveraged
  bet on the common factor.
- Do not assume the 70%/year daily-rebalance number transfers to tick frequency.

## UNVERIFIED / low-confidence

1. Dobrynskaya's numbers (70%/yr, 2/2 sort/hold) — from abstract via WebSearch, full PDF
   not read.
2. BTC-altcoin correlation 0.7-0.9 — secondary sources, no rigorous perp-specific estimate.
3. Talyxion arXiv:2511.13239 using Alpha101 on crypto — snippet only, paper not read.
4. "`rank()` appears in >90 of 101 formulas" — from a secondary summary, not tallied.
5. Qlib's `CSRankNorm`/`CSZScoreNorm` as the cross-sectional layer — stated from general
   knowledge, `processor.py` not fetched.
6. **Whether this project's ten silent detectors would fire under a beta-neutralised
   cross-sectional reformulation is a claim about our own data that no external source can
   settle. It requires running the detectors' metrics through a rank/residual transform on
   the captured tape and measuring the fire rate. That verification has not been run.**

Sources: arXiv:1601.00991 (PDF, pdftotext); microsoft/qlib `data/ops.py`,
`contrib/data/handler.py`, `contrib/data/loader.py` (raw); pypi.org/pypi/pyqlib/json;
Dobrynskaya SSRN 3913263 + HSE PDF; Liu/Tsyvinski/Wu J.Finance 10.1111/jofi.13119, NBER 25882.
