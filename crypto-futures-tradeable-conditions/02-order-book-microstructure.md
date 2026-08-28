# Order-book and order-flow microstructure conditions

**Researched by:** a `sonnet` research-web subagent, **2026-08-28**.
**Constraints given:** Binance USDM (fresh top-5/10/20 ladder every 100ms) + Bybit linear
(snapshot-then-delta); no L3, no queue position, no colocation — ordinary internet latency
(tens of ms). Taker fees ~5-5.5 bps/side, so ~10-11 bps round trip.

**Agent's own method note:** the OFI, VPIN and Cont-de-Larrard formulas below were taken
from the **primary PDFs fetched raw** (`curl` + read), not from a summarizer. Everything
else came through WebSearch, which returns synthesized summaries — those are flagged
*search-summarized* and are lower confidence.

---

## The headline for this project

**None of these is a standalone taker-order trigger at our latency and fee floor.** Every
cited result operates on aggregation windows (10 seconds to minutes, or a volume clock)
that assume either maker-side execution or a slower conditioning role. Their honest use
here is as **state variables feeding a slower model**.

---

## 1. Order book imbalance (OBI)

    OBI_N(t) = (V_bid,N(t) − V_ask,N(t)) / (V_bid,N(t) + V_ask,N(t))        ∈ [−1, 1]

summed resting size over the top N levels. Recompute on every book update, not on a clock —
it is a level, not an event.

**N and any threshold on OBI are arbitrary.** Nothing in the literature justifies N=5 over
N=20. This generic ratio is **not** in Cont-Kukanov-Stoikov or Cont-de-Larrard; it is the
practitioner form.

The theoretically grounded version is **Cont & de Larrard (2011)**, *Price dynamics in a
Markovian limit order market*, arXiv:1104.4596 — under Poisson limit-order (λ), market-order
(μ) and cancellation (θ) arrivals with a one-tick spread, they derive P(next price move is up)
in closed form from the best-queue sizes (q^b, q^a). Verified from the raw PDF: built from
**Level-1 data alone**, which is exactly our situation.

**Crypto evidence:** *Explainable Patterns in Cryptocurrency Microstructure*
(arXiv:2602.00776), Binance Futures perps, 1-second, Jan 2022–Oct 2025, claims OBI is "a
potent predictor of near-term price movements, especially during periods of market
instability." **Search-summarized — no quantitative numbers extracted.**
*When Does Order Flow Matter? State-Dependent L2 Liquidity-State Transitions in Crypto
Futures* (arXiv:2607.09230), BTCUSDT/ETHUSDT Binance 2023–mid-2026 — the title's finding is
that it is **state-dependent**: imbalance matters in some regimes and not others.

**What invalidates it:** icebergs distort visible imbalance; a partial ladder systematically
understates true depth imbalance in fast markets; and it breaks down exactly when it would be
most valuable (one-sided flushes) because the book empties faster than it can be acted on.

**Decay:** the most heavily fitted, most published signal in market microstructure. On liquid
perps this is HFT/market-maker territory. **A non-colocated participant should not expect
top-of-book OBI to be tradeable as a taker signal.** Remaining use: conditioning input to a
seconds-to-minutes model, or maker quote-skew.

---

## 2. Depth and spread, jointly

    S(t)  = P_ask − P_bid           (normalise as S/mid in bps)
    AD_i  = 1/(2(N(T_i)−N(T_{i−1})−1)) · Σ (q^B_n + q^A_n)     (CKS average depth)

**The relationship worth knowing, confirmed from the raw CKS PDF:** the price-impact
coefficient β in `ΔP_k = β·OFI_k + ε_k` scales as

    β_i = c / AD_i^λ + ν_i          (λ ≈ 1 in the stylized model)

i.e. **price impact per unit of order flow is inversely proportional to depth.** A wide
spread with thin depth means a small OFI moves price a lot; a wide spread with deep two-sided
size is a different regime entirely — **the spread alone does not tell you which one you are
in.** Peer-reviewed (Journal of Financial Econometrics 2014), NYSE TAQ, equities, R² ≈ 65%
for the linear OFI→ΔP relation over 10-second windows — with CKS's own caveat that the R² is
partly tautological because OFI is constructed from price-changing events (excluding those
gives 35-60%).

**Wide spread as opportunity vs warning is UNVERIFIED domain reasoning**, not sourced: stable
two-sided depth + low realised vol = passive-provision opportunity; a spread that *widened
suddenly* while one side's depth collapsed = market makers pulling quotes ahead of expected
volatility, and taking the other side means picking up the risk they just refused.

---

## 3. Absorption / iceberg detection — the weakest-evidenced

No standard formula. The measurable proxy: track resting size `q(t)` at a level alongside
cumulative signed trade volume executed there; "absorption" is `ΣV_traded` growing while
`q(t)` stays flat or refills faster than consumed.

- Zotikov (2019), *CME Iceberg Order Detection and Prediction*, arXiv:1909.09495,
  *Quantitative Finance* — detects icebergs via displayed-vs-executed discrepancy plus
  post-trade modification patterns. **CME futures, and it targets exchange-native iceberg
  order types.** Large crypto flow more often hides via off-book slicing algos that leave no
  iceberg flag, so **this method may not transfer at all.**
- Frey & Sandås (2009), *The Impact of Iceberg Orders in Limit Order Books*, QJF — equities
  (Xetra), peer-reviewed. Icebergs are real and reduce the submitter's impact.
- Everything else (Bookmap tooling, tape-reading blogs) is trading education, not research.

**The killer:** without L3 we cannot distinguish one large genuine passive order from an algo
re-quoting after each partial fill — both look identical as `q(t)` flat while trades
accumulate. **Any absorption detector we build is a heuristic with an unmeasured false-positive
rate, and the code should say so.**

---

## 4. Trade-flow: OFI, aggressor volume, VPIN

**OFI — exact, from the raw PDF (Cont, Kukanov, Stoikov, arXiv:1011.6402, §2.1):**

    e_n = 1{P^B_n ≥ P^B_{n−1}}·q^B_n − 1{P^B_n ≤ P^B_{n−1}}·q^B_{n−1}
        − 1{P^A_n ≤ P^A_{n−1}}·q^A_n + 1{P^A_n ≥ P^A_{n−1}}·q^A_{n−1}

    OFI_k = Σ_{n=N(t_{k−1})+1}^{N(t_k)} e_n

indexed by successive book-update events — **includes limit orders, market orders AND
cancellations, not just trades.** Stylized closed form `ΔP = OFI/(2D) + ε`; empirically
`ΔP_k = β·OFI_k + ε_k`, R² ≈ 65% at 10s aggregation on NYSE TAQ.

**This is the formula to implement.** It is computable entirely from our top-N depth feed and
needs no trade prints, which makes it **orthogonal information** to trade-based measures.

**Aggressor-side / trade-flow imbalance:** `TFI(t) = Σ(signed trade volume)`, sign taken from
the taker side — **both venues label trade side, so no Lee-Ready tick-rule inference is
needed.** This is a genuine data advantage over the equities literature. CKS §4.1 compare the
two directly and find **OFI explains price moves better than trade-based measures.**

**VPIN — exact, from the raw PDF:** with volume buckets of size V and n buckets,

    V^B_τ = Σ_i V_i · Z((S_i − S_{i−1}) / σ_ΔS)          (bulk volume classification)
    V^S_τ = V − V^B_τ
    VPIN  = Σ_τ |V^S_τ − V^B_τ| / (n·V)

Peer-reviewed lineage (J. Financial Econometrics 2008 → J. Portfolio Management 2011 → RFS).
LBNL reviewed it at SEC request and found it predictive of the May 2010 Flash Crash ~1 hour
ahead — **a single, famous, retrospectively identified event.** Subsequent literature is
mixed: Abad & Yagüe (2012), *Spanish Review of Financial Economics*, find "the key variable is
the number of buckets used" — i.e. **VPIN's usefulness is sensitive to a tuning parameter with
no principled selection rule.** Menkveld & Yueshen (2013) find the flash-crash behaviour is
consistent with one large seller's strategic pattern rather than universal toxicity.

**Agent's own inference (reasoning, not a cited transfer):** VPIN's bulk volume classification
exists to work around *not having trade-side data*. We *do* have taker-side labels, so a
simpler `Σ|signed taker volume|/(n·V)` computed directly from labelled prints is arguably
better grounded for us than reconstructing BVC.

**What invalidates OFI:** it is symmetric to cancellations and limit orders, so a book being
**wall-spoofed** (large orders posted and pulled, never filled) generates OFI signal with zero
informational content. This is a known real pattern on crypto perps and neither CKS nor VPIN
distinguishes it from genuine supply/demand change.

---

## 5. What survives without L3 — the good news

- **Cont-de-Larrard is Level-1 by design** (confirmed from the raw PDF: "queue sizes at the
  best bid and ask … are more easily obtainable … than Level II data"). **Caveat from the same
  PDF: the entire tractability rests on the spread being 1 tick >98% of the time.** Crypto
  perps routinely trade multiple ticks wide during volatility. **This assumption must be
  checked against our own tape before the closed-form results are trusted — UNVERIFIED for
  crypto.**
- **Microprice** (Stoikov 2018, SSRN 2970694) — mid adjusted by queue imbalance and spread
  jointly, via a fitted Markov transition matrix rather than a linear weight; a martingale by
  construction and a better short-horizon fair-price estimate than mid. **Search-summarized —
  fetch the raw SSRN PDF before coding it.**
- **Survives without L3:** queue-imbalance direction prediction, OFI, microprice, VPIN/TFI.
- **Does NOT survive:** true iceberg/hidden-order detection, and anything needing queue
  *position* within a level (so priority-based fill-probability models are out of reach).

---

## Bottom line

**If forced to keep one: OFI, as a state variable feeding a slower model.** Best-evidenced,
formula-level, computable from the quote/depth feed alone, and CKS's own result — **fit
improves at longer aggregation, not shorter** — works in favour of a retail-latency
participant, which is unusual for this literature.

**Skip entirely:** treating any of these as a standalone taker trigger against the ~10-11 bps
round-trip floor. None of the cited evidence supports that use.

---

## UNVERIFIED

1. Quantitative OBI performance on crypto perps from arXiv:2602.00776 — only abstract-level
   claim retrieved, no numbers.
2. "Predictive effects decay rapidly out-of-sample beyond 1-day horizon" for OFI-type signals
   — from a search summary citing a blog post; underlying study unidentified.
3. The wide-spread opportunity-vs-warning framing (§2) — the agent's own domain reasoning, not
   sourced to any paper.
4. Stoikov's microprice transition-matrix formula — from search summaries. **Do not code the
   microprice from this report.**
5. Whether Cont-de-Larrard's ">98% one-tick spread" assumption holds for BTC/ETH perps on our
   venues — not checked by the agent or by any paper found.
6. VPIN's applicability to crypto perps — **no crypto replication found.** All VPIN evidence
   here is equities.
7. Exact current Binance/Bybit taker fee bps — from third-party comparison sites, not the
   venues' own fee schedules. Confirm against the actual account tier before hardcoding a
   break-even.
8. Whether CME iceberg detection transfers to crypto at all.

## Sources
- Cont, Kukanov, Stoikov (2011/2014), *The Price Impact of Order Book Events*, arXiv:1011.6402 — primary PDF read
- Cont & de Larrard (2011), *Price Dynamics in a Markovian Limit Order Market*, arXiv:1104.4596 — primary PDF read
- VPIN derivation, sourcing Easley/Engle/O'Hara/Wu (2008), Easley/López de Prado/O'Hara (2011), RFS — primary PDF read
- Abad & Yagüe (2012), *From PIN to VPIN*, Spanish Review of Financial Economics, doi:10.1016/j.srfe.2012.10.002
- Menkveld & Yueshen (2013), *Anatomy of the Flash Crash*, SSRN 2243520
- Zotikov (2019), *CME Iceberg Order Detection and Prediction*, arXiv:1909.09495
- Frey & Sandås, *The Impact of Iceberg Orders in Limit Order Books*, SSRN 1108485
- Stoikov (2018), *The Micro-Price*, SSRN 2970694
- *Explainable Patterns in Cryptocurrency Microstructure*, arXiv:2602.00776
- *When Does Order Flow Matter?*, arXiv:2607.09230
