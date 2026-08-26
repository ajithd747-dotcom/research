# How rules get born without a human, and validated without self-deception

**Researched 2026-08-26 by a `sonnet` subagent (Rule 1). The Deflated Sharpe and PBO
formulas were extracted from the primary PDFs directly, not from a summarizer.**

## The verdict that matters most, up front

This project's failure — 479,323 hypotheses refused, 0 written — is **not** a symbolic
regression problem or a statistics problem. **It is a schema problem upstream of
validation.**

> Every credible system solves "how does rule #1 get born" by *constraining what a
> hypothesis is allowed to look like at creation time*, not by relaxing the gate that
> checks it afterward.

An LLM, template, or miner that can emit a hypothesis **missing** its falsification
criterion and regime tag will do so, forever, and a downstream gate that requires those
fields but does not require them *of the generator* just accumulates an unusable queue.
**The instruction writer refusing is the Rule 0 / Rule 8 apparatus working correctly and
telling the truth about an upstream defect.**

## Deflated Sharpe Ratio — exact formula, from the primary PDF (Bailey & Lopez de Prado)

Eq. 2, p.8:

    DSR = PSR(SR_0) = Z[ (SR - SR_0) * sqrt(T - 1)
                         / sqrt(1 - g3*SR + ((g4 - 1)/4)*SR^2) ]

Deflation threshold, same page:

    SR_0 = sqrt(V[{SR_n}]) * ( (1 - gamma) * Z^-1[1 - 1/N]
                              + gamma * Z^-1[1 - (1/N)*e^-1] )

- `Z` = standard Normal CDF; `g3`, `g4` = skewness and kurtosis of the **selected**
  strategy's returns; `T` = sample length; `V[{SR_n}]` = variance across the `N` trials'
  Sharpe estimates; `N` = number of independent trials attempted;
  `gamma` = 0.5772 (Euler-Mascheroni).
- Eq. 1, p.7 is the mechanism: the expected max Sharpe under **pure noise** grows with N.
  More trials mechanically inflate the best-looking result with zero true skill.
- **Worked example, p.9-10:** N=100, V=0.5, T=1250, skew=-3, kurtosis=10, annualised
  SR=2.5 -> SR_0 ~ 0.1132 -> **DSR ~ 0.9004 < 0.95 -> REJECTED.** An apparently excellent
  Sharpe, rejected once the trial count and non-Normality are accounted for. This is the
  canonical illustration that "it backtested well" is not evidence.
- No canonical maintained permissive package implements DSR alone; the formula is ~15
  lines. `pypbo` has it but is **AGPL-3.0** (copyleft — check before vendoring).

## Probability of Backtest Overfitting (PBO) via CSCV

Primary: Bailey, Borwein, Lopez de Prado, Zhu.

- **Def 2.1 (exact):** overfitting occurs if
  `sum_n E[r_bar_n | r in Omega*_n] * Prob[r in Omega*_n] <= N/2`
  — the in-sample-best strategy has an *expected* out-of-sample rank at or below median.
- **CSCV procedure:** split the performance matrix into `S` contiguous blocks; for every
  partition of those blocks into IS/OOS halves (all `C(S, S/2)` combinations), take the
  IS-best strategy, find its OOS rank, convert to a logit, average. The fraction of
  combinations where the logit is negative **is** the PBO estimate.
- **Input is only a matrix of trial performance series** — model-agnostic, a real advantage
  over DSR, which needs N, V, skew and kurtosis estimated separately.
- Implementation `esvhd/pypbo` is **AGPL-3.0** (LICENSE fetched directly). No permissive
  maintained alternative found; CSCV is small enough to implement from the definition.

## White's Reality Check / Hansen SPA

- Tests whether the *best* of many candidates truly beats a benchmark vs data-snooping —
  the frequentist sibling of DSR/PBO, bootstrap rather than parametric.
- **`arch.bootstrap.SPA`** implements both (stationary / circular-block / moving-block
  bootstrap, default 1000 replications). **License permissive (NCSA/BSD-style).**
  **cp314 wheel CONFIRMED**: `arch-8.0.0-cp314-cp314-manylinux2014_x86_64.whl`.

## Benjamini-Hochberg FDR

`statsmodels.stats.multitest.multipletests(pvals, method='fdr_bh')`. BSD-3.
**cp314 wheel CONFIRMED**: `statsmodels-0.14.6-cp314-cp314-manylinux2014_x86_64.whl`.

## Purged K-fold with embargo, and CPCV

- **`mlfinlab` (Hudson & Thames) is CLOSED SOURCE / all-rights-reserved** — commercial
  licence required. Do not build on it.
- **`eslazarev/purged-cross-validation`** — **MIT** (LICENSE fetched directly),
  sklearn-compatible splitter protocol, implements purging + embargo, expanding/rolling
  walk-forward, purged and group-purged K-fold, **Combinatorial Purged CV with path
  reconstruction**, plus PSR/DSR/Minimum Track Record Length. Best licence-and-coverage
  match found. UNVERIFIED: copyright year 2026, very new, source not read — read the
  splitter before trusting it against real capital.
- **`mlfinpy`** (PyPI, MIT) — licence and summary checked only; version 0.1.2. UNVERIFIED.
- Procedure (secondary-source knowledge; *Advances in Financial Machine Learning* was not
  fetched): labels span intervals, so naive K-fold leaks. **Purge** training observations
  whose label interval overlaps the test set's. **Embargo** an additional window
  immediately after the test set, against serial-correlation leakage. **CPCV** uses every
  `C(N, k)` choice of test groups, producing many backtest paths instead of one — feeding
  a distribution to PBO/DSR rather than a point estimate. No universal numeric defaults.

## Where rule #1 comes from, in practice — three patterns and what breaks

1. **Seeded from a hand-written prior library** (WorldQuant's Alpha101 is the cited real
   example): declare a fixed vocabulary, then mutate/recombine over it. **Breaks:** the
   seed encodes the author's blind spots as the entire reachable universe — no
   mean-reversion primitive means mean reversion is undiscoverable. Lowest risk of
   nonsense output, and closest in spirit to this project's blueprint-first architecture.
2. **Coarse grid search over a small hand-bounded operator space** — a few hundred to a
   few thousand candidates, each carrying a real falsification criterion from birth.
   **Breaks:** combinatorial explosion, and **if every candidate is not counted as a trial
   for DSR/PBO, you have silently done the exact overfitting the apparatus exists to
   prevent — the trial count is the grid size, not the number promoted.**
3. **LLM proposing candidates** (AlphaGen's `alphagen_llm`, RD-Agent). **Breaks: exactly
   this project's bug** — a generator with no schema constraint will not spontaneously
   attach a criterion and a regime tag.

## AlphaGen / RD-Agent

- `github.com/RL-MLDM/alphagen` (KDD 2023, 10.1145/3580305.3599831). Modules: `/alphagen`,
  `/alphagen_qlib`, `/alphagen_generic`, `/alphagen_llm`, bundled `/gplearn`, `/dso`.
  Mechanism: **maskable PPO** builds an expression token-by-token in RPN; reward is the
  **IC of the alpha *combination pool***, not one alpha's fitness — optimising a set
  jointly to reduce redundancy among survivors.
- Claimed IC 0.0515 / "75% improvement" — **UNVERIFIED**, from a WebSearch summary of a
  follow-on paper, tables not read.
- `microsoft/RD-Agent` — multi-agent LLM loop generating arbitrary code. **A poor fit here
  on principle**: this project has already excluded "learned output becomes executable
  code" by design. The "2x returns, 70% fewer factors" figure is blog-sourced; do not use it.

## Libraries and cp314 status

| Library | cp314 | Note |
|---|---|---|
| **gplearn** 0.4.3 | **CONFIRMED** — `py3-none-any`, pure Python | Fitness = user metric minus `parsimony_coefficient` on tree size. Overfitting controls it ships: `p_hoist_mutation`, `max_samples` (OOB-like). **No purged CV, no multiple-testing correction — must be wrapped** |
| **DEAP** 1.4.4 | **CONFIRMED** — pure Python wheel | Right tool if the primitive set must be hand-constrained to a bounded grammar |
| **PySR** 2.0.0 | Wrapper installs, but first import downloads **Julia** via juliaup (~1-2 GB, minutes, UNVERIFIED). User-space capable, no C compiler needed | Real cost — flag before installing |
| `arch` 8.0.0 | **CONFIRMED** | SPA / Reality Check |
| `statsmodels` 0.14.6 | **CONFIRMED** | BH FDR |
| `river` 0.26.1 | **CONFIRMED** (a cp315 wheel exists too) | `ADWIN`, `PageHinkley` online drift detectors |

## Online retirement — CUSUM / SPRT

CUSUM accumulates deviations from a reference mean and fires at threshold `h`; it is
mathematically a sequence of restarted SPRTs with absorbing barriers at 0 and `h` — not a
separate technique. SPRT stops as soon as evidence suffices rather than at a fixed sample
size, which matters when a degraded rule loses money every day it stays live.
`river` ships ADWIN and PageHinkley.

**The retire-on-noise risk:** a rule with a true edge still has losing streaks. A naive
"N losses in a row -> kill it" is itself an ungated, unfalsifiable rule. Thresholds need
the same trial-counted distributional reasoning as DSR/PBO. UNVERIFIED — no finance-specific
primary paper proposing SPRT for rule retirement with published parameters was found.

## What not to bother with — stated plainly

**Pure genetic-programming alpha discovery with a naive single-metric fitness and no
purge/embargo/multiple-testing layer is a well-documented way to overfit, not a
well-documented way to find alpha.** The DSR paper's own worked example — a 2.5 Sharpe
rejected once 100 trials are counted — is exactly the shape a naive GP run produces at
scale. It will always find *something*; the something is a memorised noise pattern with
"null power over the future" (their words, p.5).

**A small hand-bounded operator space searched by grid or coarse random sampling, gated by
DSR/PBO/CPCV with an honestly counted trial number, gets further with far less machinery.**
GP only earns its keep once a validation harness exists that can withstand thousands of
auto-generated candidates without fooling itself. **This project has 0 proven instructions
— building the GP generator now is building the more dangerous half first.**

## UNVERIFIED / low-confidence

1. AlphaGen IC 0.0515 / "75% improvement" — WebSearch summary, tables not read.
2. RD-Agent "2x returns, 70% fewer factors" — blog (saulius.io), marketing-adjacent.
3. `mlfinpy` — licence and summary only; correctness, cp314 wheel, coverage unverified.
4. `eslazarev/purged-cross-validation` — MIT confirmed by direct fetch; source not read,
   maintenance and adoption unknown.
5. CPCV numeric defaults (N, k, embargo length) — book not fetched; procedure directionally
   correct, not word-for-word verified.
6. CUSUM/SPRT as a practitioner standard for retiring trading rules — general
   sequential-analysis theory, no finance-specific citation with parameters.
7. gplearn/PySR candidates-generated-vs-survived ratios — no citable figure found; none invented.
8. PySR's Julia install footprint — qualitative, not measured on this box.
9. `arch` on cp314 — wheel availability confirmed, not run.

Sources: davidhbailey.com/dhbpapers/deflated-sharpe.pdf; .../backtest-prob.pdf;
gplearn.readthedocs.io/en/stable/intro.html; github.com/{trevorstephens/gplearn,
MilesCranmer/PySR, RL-MLDM/alphagen, microsoft/RD-Agent, esvhd/pypbo,
hudson-and-thames/mlfinlab, eslazarev/purged-cross-validation}; ACM DL 10.1145/3580305.3599831;
arch.readthedocs.io .../SPA.html; riverml.xyz/dev/api/drift/PageHinkley/;
pypi.org/pypi/{statsmodels,arch,river,gplearn,pysr,deap,mlfinpy}/json.
