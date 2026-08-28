# Cross-venue, cointegration, event-driven, and liquidity screening conditions

**Researched by:** a `sonnet` research-web subagent, **2026-08-28**.
**Constraints given:** two venues only (Binance USDM, Bybit linear), both large and centralised;
~30-50 symbols each; ordinary internet latency, no colocation; ~6 days of tape; no on-chain data,
no paid news feed.

**Agent's method note:** all findings via native WebSearch plus one WebFetch of an arXiv abstract.
No quota limit hit. Full-text PDFs of Makarov & Schoar, Krauss 2017, and Alexander et al. 2024
could **not** be pulled — only abstracts/snippets — so numeric specifics from those are flagged.

---

## 1. Cross-venue dislocation — skip it

    dislocation_bps = (mid_A − mid_B) / ((mid_A + mid_B)/2) × 10_000

**Blunt answer: not a standalone strategy at our latency.**

- **Makarov & Schoar, *Trading and Arbitrage in Cryptocurrency Markets*, J. Financial Economics
  135(2), 2020** — the canonical study. Found deviations that "often persist for weeks", **but its
  setting is Bitcoin spot across geographically and capital-control-segmented venues (Korea, Japan
  vs US)** — not two large USD-denominated CEX perp books. **That result does not transfer to
  Binance-vs-Bybit USDT perps**, which are far more integrated. UNVERIFIED as applied to this pair.
- Practitioner consensus (all anecdotal tier, no methodology disclosed): typical Binance-vs-Bybit
  BTC perp spreads quoted around 0.3-0.5%, but break-even after two taker legs (~0.09-0.11%
  combined even with discounts) plus ~0.05% slippage each side needs roughly **0.20-0.25% gross
  spread**. On BTC/ETH — deepest on both venues — spreads that wide are rare and close in seconds
  to low minutes. Explicitly described as **not humanly actionable**.
- A figure worth naming only to distrust: "17% of observations exhibit spreads ≥20bps; only 40% of
  the top opportunities are net-profitable after costs and reversal risk" — **from a search-engine
  synthesis, no paper/author/year. UNVERIFIED, do not code against it.**

**What invalidates it:** our own stated constraint. Tens-of-ms internet latency, two-leg taker
execution across two connections. Colocated participants close these before we can observe them.

**On the long tail:** wider, slower-closing dislocations may exist on illiquid alt perps because
fewer bots watch them — but **that is a liquidity-screening question, not a cross-venue edge, and
the wider spread there is compensation for one-sided risk (possibly not being able to close both
legs), not free money.**

**Verdict: do not build this as a primary edge.** At most a *feature* — persistent one-directional
lag between venues — feeding a different signal.

---

## 2. Cointegration / pairs trading — build the pipeline, distrust the pairs

Standard stack: Engle-Granger two-step (regress A on B, ADF-test the residual) or Johansen for >2;
entry/exit on the spread's z-score; size by the OU half-life `t_half = ln(2)/θ`.

**Evidence:** *Evaluation of Dynamic Cointegration-Based Pairs Trading Strategy in the
Cryptocurrency Market*, arXiv:2109.10662 (2021) — uses exactly this stack (Engle-Granger + KSS +
Johansen, OU half-life setting the lookback, minute-binned BitMEX data, realistic bid/ask
execution). Reported to exceed buy-and-hold with low drawdown. **Exact half-life, Sharpe, and
out-of-sample degradation could not be confirmed from the abstract — UNVERIFIED.**

**The paper's own caveat, echoed everywhere:** *"it is not difficult to find cointegrated pairs
during some chosen historical period, but they can easily lose cointegration in the subsequent
out-of-sample period."* **Treat pair stability as weeks, not months.**

### The multiple-testing problem — the main thing to get right

With ~100 candidate series across both venues, `C(100,2) ≈ 4,950` pairwise tests (fewer,
~1,225-2,000, restricted to same-venue). **At a naive 5% threshold with no correction, expect
~60-250 pairs to pass purely by chance even if none are truly cointegrated.** (This is the agent's
own arithmetic from our symbol count, not a citation.)

**Correct handling:**
- **Benjamini-Hochberg FDR control** across all pairs tested per scan, not a flat p < 0.05.
- **A held-out period:** fit on window N, require it to still pass on window N+1 before trading.
  A pair discovered and traded on the same data is not evidence, it is overfitting.
- **Re-test every rolling window and drop failures.** Never grandfather a pair that once cointegrated.
- **With ~6 days of tape we do not have enough history to run this responsibly.** Cointegration and
  OU half-life estimation wants weeks-to-months. Six days is enough to prototype code, not to trust
  a signal.

**Horizon:** hours to a few days, set by the fitted half-life. **A half-life coming back in minutes
is probably a window artefact; weeks+ is probably not exploitable net of carry.** Both are red flags.

**Verdict: worth building — but the deliverable is the FDR-controlled, walk-forward-validated
scanning pipeline, not any specific pair.**

*(Note: this project's `cointegration-pair-finder` currently reports 263 cointegrated pairs from
2,211 correlated pairs. Against the arithmetic above, that count is exactly the shape a
multiple-testing artefact would produce, and the part does not appear to apply an FDR correction.
Worth checking before any of those verdicts is acted on.)*

---

## 3. Perp-vs-spot and calendar basis

**Calendar basis is not available to us at all** — neither venue's linear/USDM perp has a paired
dated-futures contract in our data. Not a research gap, a data gap. Skip.

**Perp-vs-spot** is computable from mark/index: `basis_bps = (mark − index)/index × 10_000`.
This is the classic cash-and-carry trade, and it is **documented as decayed, not open:**

- **BitMEX, *State of Crypto Perpetual Swaps 2025*** (exchange research, practitioner tier):
  funding-rate arbitrage was productised by Ethena in 2024 and became overcrowded through 2025,
  with funding compressing toward a **~4% annualised baseline** — described explicitly as
  "killing the funding rate trade."
- The mechanism of decay: institutional capital now shorts perps at scale the instant funding
  spikes, compressing the premium **within the funding interval itself** — the opposite of a slow,
  retail-capturable signal.
- Corroborating figure (also in file 01): carry Sharpe 6.45 full-sample → 4.06 from 2024 →
  **negative in 2025**. UNVERIFIED provenance, but consistent with the BitMEX narrative.

**If built anyway:** trade only when annualised funding exceeds round-trip cost by a real margin —
practically the tail of altcoin perps during acute one-sided demand (a listing pump, a squeeze),
**not** the steady-state trade on majors. And on illiquid alts it stops being a clean arb and
becomes an inventory-risk trade where you can get stuck unable to close without moving the market.

---

## 4. Event-driven — skip, and for a specific reason

- The evidence is about **spot exchange listings**, not perp listings. One working paper reports
  **5.7% abnormal return on listing day, 9.2% over ±3 days** — and notably **front-loaded *before*
  the event, turning negative after.** So the tradeable part, if any, is the **pre-announcement
  drift, not a post-listing pop.** UNVERIFIED whether this transfers to perp listings at all, since
  a perp listing usually follows an already-priced spot listing.
- **We have no announcement or news feed.** We can only observe a symbol appearing in the listing
  feed — by which point, per the same studies, the abnormal return has already happened.
  **Building a detector on "new symbol appears" is building a detector for the tail end of an
  already-priced move.**
- **Delisting:** same problem, same absence of a perp-specific study.
- **Funding-interval changes** (8h→4h→1h under stress): documented mechanically by Binance itself,
  but **no event study quantifying a price effect from the interval change**. The change is a
  *symptom* of divergence, not a shown cause of a tradeable move. Treat "trade the interval change"
  as **unsupported, not merely uncited.**
- **Index inclusion:** not applicable to us.

---

## 5. Liquidity and tradeability screening — the most useful section

**No peer-reviewed paper deriving these thresholds for crypto perps was found.** Everything below
is practitioner tier plus one peer-reviewed MDPI paper (*Order Book Liquidity on Crypto Exchanges*,
IJFS 18(3):124, 2025) whose own recommended cutoffs the agent could not extract. **These are
reasonable cost-accounting starting heuristics, not externally validated numbers.**

| Metric | Formula | Threshold (practitioner consensus, unverified) | Reasoning |
|---|---|---|---|
| 24h quote volume | Σ trade notional, 24h | ≥ **$1M/day** floor cited repeatedly; "safe" nearer several $M/day; hard floor for tiny size ~$100K/day | Below this your own order is a meaningful fraction of daily flow |
| Bid-ask spread | (ask−bid)/mid | **< 0.10-0.15%** beyond BTC/ETH tier; majors run ~1bp | A direct unavoidable cost on both legs |
| Depth within X bps | Σ resting notional within ±X bps of mid | Depth within ±50-100bps should comfortably exceed intended size; **use no more than 1-5% of visible depth** | Stops your own order walking the book |
| Slippage on a test clip | simulated fill vs pre-trade mid | **< 0.3-0.5%** on a representative clip (e.g. $10K) | Measures directly what the depth check proxies |
| Tick-size-to-price | tick_size / price | flag/exclude when **> ~1-5 bps** | On low-price alts the spread is **structurally floored by the tick** — no amount of liquidity fixes it |
| Trade count | count over rolling window | no well-evidenced number; require tens per hour, not single digits | **A volume floor can be met by one huge print; trade count catches that** |

**The mechanic that makes this the gate for everything else:** edge per round trip must exceed
`2 × half-spread + expected slippage + 2 × taker fee`. On majors that floor is ~3-15 bps; **on our
weaker 20-30 symbols it is easily 30-100 bps+, which eats most of the statistical edges in files
01-04 before the strategy's own gross edge is even counted.** A cointegrated pair or a basis signal
on a symbol that fails this screen is not tradeable regardless of how real the relationship is.

**What invalidates the screen:** thresholds computed from a 6-day tape are themselves noisy —
volume and spread on the smaller symbols swing several-fold week to week. **Re-screen on a rolling
basis; "passed once" is not durable.** This does not decay; it is arithmetic that must be redone.

---

## Skip outright

- **Cross-venue latency arb on BTC/ETH between our two venues** — dead for anyone non-colocated.
- **Steady-state funding/cash-and-carry on majors** — crowded, Sharpe decayed to negative through 2025.
- **Calendar/dated-futures basis** — not buildable with our data at all.
- **Event-driven listing/delisting/announcement trading** — no news feed, and the return happens
  before we could observe it.
- **Trusting any cointegration output on 6 days of tape.** Build the pipeline; do not trade it.

---

## UNVERIFIED

1. "17% of observations ≥20bps; 40% net-profitable" — unattributed search synthesis. Do not code against.
2. Exact half-life/Sharpe/OOS degradation from arXiv:2109.10662 — abstract-level methodology only.
3. Krauss (2017) multiple-testing/FDR specifics — could not retrieve. **The ~60-250 spurious-pairs
   estimate is the agent's own arithmetic from our symbol count, not a citation.**
4. Carry Sharpe 6.45 → 4.06 → negative — provenance unconfirmed; consistent with BitMEX narrative.
5. 5.7% listing-day / 9.2% ±3-day abnormal returns — working-paper tier, and **spot** listings;
   transfer to perp listings unconfirmed.
6. **All §5 numeric thresholds** — converging practitioner sources, not a peer-reviewed derivation.
   A starting screen to backtest and tighten against our own fill data, not settled numbers.
7. Whether funding-interval changes cause an independent price effect — only descriptive sourcing found.

## Sources
- Makarov & Schoar (2020), *Trading and Arbitrage in Cryptocurrency Markets*, JFE 135(2) — <https://personal.lse.ac.uk/makarov1/index_files/CryptocurrencyMarkets.pdf>
- *Evaluation of Dynamic Cointegration-Based Pairs Trading Strategy in the Cryptocurrency Market*, arXiv:2109.10662
- Krauss (2017), *Statistical Arbitrage Pairs Trading Strategies: Review and Outlook*, J. Economic Surveys 31(2)
- Alexander, Chen, Deng, Wang (2024), *Arbitrage opportunities and efficiency tests in crypto derivatives*, J. Financial Markets
- *Order Book Liquidity on Crypto Exchanges*, MDPI IJFS 18(3):124 (2025)
- *Market Reaction to Exchange Listings of Cryptocurrencies*, Blockchain Research Lab working paper
- BitMEX, *State of Crypto Perpetual Swaps 2025* — <https://www.bitmex.com/blog/state-of-crypto-perps-2025>
- Binance, *Price Convergence and Funding Fees in Perpetual Futures Markets*
