# Which perp opportunity types are real, and what free data each needs

**Researched 2026-08-26 by a `sonnet` subagent (Rule 1). Bybit docs are static GitHub
Pages and rendered cleanly. Binance's new docs site is JS-rendered (Zudoku/React) and
neither `curl` nor `WebFetch` got raw markdown for two pages — leverage brackets and the
liquidation stream — so those fell back to WebSearch summaries plus a GitHub connector
source. Flagged inline.**

## The ranking — build two, maybe three, drop five

| Opportunity | Free data | Evidence | Verdict |
|---|---|---|---|
| **Funding rate (as CARRY, not direction)** | Excellent — universe-wide, 1 call per venue | Published, quantified (Christin) | **BUILD** — as a carry/basis signal, not a directional predictor |
| **Liquidation map (Bybit)** | Good — `allLiquidation` complete, `risk-limit` PUBLIC | Mechanistic; arithmetic on public margin tiers, not folklore | **BUILD NOW** — this is exactly the stated blocker |
| **Liquidation map (Binance)** | Weak — throttled stream, **signed** bracket endpoint | Same mechanism | **BUILD SECOND**, after wiring a signed client |
| Open interest (Bybit only) | Free in the same `tickers` call | **Blog-tier only**, no quantified effect found | Cautious secondary feature on Bybit only |
| Basis (perp vs index) | Trivial, same call as funding | Not distinguishable from funding in evidence found | **Fold into funding — do not build separately.** Very likely the same information twice |
| Long/short ratio, taker buy/sell | Per-symbol, rate-limited, 30-day window, Binance only | **Folklore, no quantified evidence** | **DROP** — rate-limit-disqualified at 1500 symbols |
| Cross-venue divergence | Free, trivial to compute | Blog-tier, but mechanism unambiguous | **DROP** — arbitraged below retail latency |
| On-chain whale flows | Free tier unusable | N/A | **DROP** |
| Social sentiment | Unverified free-tier limits | Unverified | **DROP for now** — deserves a 5-minute doc check, not a build |
| Options IV (Deribit) | Free but **BTC/ETH only** | Real index, ~2 of 1500 symbols | **DROP as a per-symbol input**; keep as a macro gate at most |

## Funding rate — exact endpoints

**Binance USD-M** (`https://fapi.binance.com`):
- `GET /fapi/v1/premiumIndex` — symbol optional. **Weight 1 with symbol, 10 without.**
  Returns `markPrice`, `indexPrice`, `lastFundingRate`, `nextFundingTime`.
  *Caveat:* the field is named `lastFundingRate`, not a forward predicted rate. Verify
  against accumulation logic before treating it as a forecast.
- `GET /fapi/v1/fundingRate` — `symbol` opt, `startTime`, `endTime`, `limit` (default 100,
  max 1000). Shares a **500 req / 5 min / IP** budget with `fundingInfo`.
- `GET /fapi/v1/fundingInfo` — no params, weight 0. `adjustedFundingRateCap`,
  `adjustedFundingRateFloor`, `fundingIntervalHours`.

**Bybit v5** (`https://api.bybit.com`):
- `GET /v5/market/tickers?category=linear` — **one call, whole universe**: `fundingRate`,
  `indexPrice`, `markPrice`, `nextFundingTime`, **and `openInterest`/`openInterestValue`**.
- `GET /v5/market/funding/history?category=linear&symbol=...` — `symbol` and `category`
  **required**, `limit` 1-200 default 200. Rate limit "not specified" in docs (UNVERIFIED).

**Both venues give the whole universe's current funding in a single call.** No rate-limit
disqualification. History is per-symbol on both.

**Evidence:** Christin, *The Crypto Carry Trade* — carry (long spot / short perp, collect
funding) mean return ~8%/period, vol ~0.8% (via abstract, **full text UNVERIFIED**).
Separately: *"a 10% increase in standardized carry predicts a 22% increase in
sell-liquidations relative to open interest over the following month."* That is a
liquidation-risk prediction, not a clean directional edge. **This is a market-neutral carry
strategy needing an offsetting position — not something a directional scanner captures.**

## Open interest — the first real disqualifier

- Binance: `GET /fapi/v1/openInterest?symbol=` (weight 1, **single symbol only, no universe
  endpoint**); `GET /futures/data/openInterestHist?symbol=&period=5m..1d&limit=` (max 500,
  **latest 30 days only**, weight 0 but the `/futures/data/*` family shares **1000 req /
  5 min / IP**).
- Bybit: `GET /v5/market/open-interest?category=linear&symbol=&intervalTime=5min..1d` —
  also per-symbol.

**Neither venue has a universe-wide OI snapshot endpoint. At 1,500 symbols that is 1,500
REST calls per refresh against a 1000-req/5min budget — a hard wall on Binance.** But
Bybit's `tickers` call already carries `openInterest` per symbol, so **on Bybit OI is free
as a byproduct of the funding call; on Binance it is not.** Design around the asymmetry.

**Evidence for OI+price divergence: none found.** Every hit was retail-education content.
**UNVERIFIED / blog-tier.** The mechanism is intuitive and widely repeated with no
quantified effect size, backtest, or study. Folklore until backtested on our own tape.

## Liquidation data — and the exact fix for the empty maps

**Binance WS** (fallback-sourced, flagged): per-symbol `<symbol>@forceOrder`, all-market
`!forceOrder@arr`. **Documented throttle, confirmed by two independent hits:** *"for each
symbol, only the latest one liquidation order within 1000ms will be pushed as the
snapshot."* So `!forceOrder@arr` is **a 1-per-second-per-symbol sample, not a full
liquidation tape.**

**Bybit WS:** the deprecated `Liquidation` topic has the same throttle;
**`allLiquidation.{symbol}` (current) pushes every liquidation, batched at 500ms** —
fields `T`, `s`, `S`, `v`, `p`. **Binance has no equivalent, so Binance is structurally
worse for this opportunity type**, not merely differently documented.

**Maintenance margin / leverage brackets — the piece the mapper cannot load:**
- **Binance `GET /fapi/v1/leverageBracket` is a SIGNED endpoint.** Free account, but not an
  anonymous public GET like the rest of the market data. Returns `symbol`, `notionalCoef`,
  `brackets[]` with `bracket`, `initialLeverage`, `notionalCap`, `notionalFloor`,
  `maintMarginRatio`, `cum`. **If the mapper is hitting this unauthenticated, that is
  exactly why it returns nothing — it needs API-key wiring, not a different endpoint.**
- **Bybit `GET /v5/market/risk-limit?category=linear&symbol=` is PUBLIC, unauthenticated** —
  `id` (riskId), `riskLimitValue`, `maintenanceMargin`, `initialMargin`, `maxLeverage`,
  `isLowestRisk`, `mmDeduction`, paginated 15 symbols/page via `cursor`.

**This is the immediately actionable finding: the liquidation-map blocker is
Binance-specific and Bybit needs no auth at all.**

## Cross-venue divergence — drop it

Breakeven at 0.1% taker each side is ~0.2% round trip. One worked example: a 0.4% gross
spread on $10,000 nets ~$6 (0.06%) after fees — needing ~167 trades/day or ~$167k/trade to
clear $1,000/day. Arbitraged away by co-located HFT below what a REST-polling scanner on
this box can reach. **All sources blog-tier, but multiple independent low-quality sources
agree and none contradicts, which raises confidence in the conclusion though not in any
single source.**

## On-chain and social — drop

- **Whale Alert free tier: 10 API calls/minute, and only transactions >$500k USD.**
  Unusable for a 1,500-symbol universe — cannot poll once per symbol per hour, and the
  long tail will never produce a whale event at that floor.
- **Glassnode / CryptoQuant free tiers: daily resolution**, restricted metric subset.
  Dashboard-browsing data, not feed data. (UNVERIFIED exact limits — no primary pricing
  page fetch succeeded.)
- **LunarCrush** claims real-time across 4,000+ coins, which would be the best coverage
  here if true — but free-tier rate limits and whether "real-time" holds on it are
  **entirely unverified, from marketing pages**. Not a confirmed limitation, which is why
  it is "drop for now" rather than "disqualified."

## Options IV — drop as a scanner input

Deribit's public API is free; **DVOL covers BTC and ETH only**, confirmed by multiple
independent sources. No altcoin IV surface exists on any free source found. At 1,500
symbols that is ~2 of them.

## UNVERIFIED / low-confidence

1. **Binance `!forceOrder@arr` USDM payload schema** — the field list quoted appears to
   blend COIN-M fields (`ps`, `st`) into a USDM example. The docs page is JS-rendered and
   returned 0 bytes to curl. **Confirm against a live subscription before coding a parser.**
2. Bybit rate limits for `funding/history` and `open-interest` — docs said "not specified";
   very likely wrong, check Bybit's rate-limit page.
3. `/futures/data/takerlongshortRatio` response schema — WebSearch summary, not raw doc.
4. Christin's carry numbers (8%, 0.8% vol, 10%->22%) — abstract text, tables not verified.
5. OI-divergence and long/short-ratio predictive claims — retail/blog only, no academic or
   quant-backtest source found despite searching. **Folklore until backtested on our tape.**
6. Cross-venue fee-breakeven numbers — SEO/affiliate blogs, not primary fee-schedule math.
7. Glassnode/CryptoQuant free-tier limits and metric lists — third-party comparison sites.
8. LunarCrush free-tier real-time-ness and limits — marketing/review pages only.

Sources: developers.binance.com/docs/derivatives/usds-margined-futures/{market-data/rest-api/
Mark-Price, Get-Funding-Rate-History, Get-Funding-Rate-Info, Open-Interest,
Open-Interest-Statistics, Taker-BuySell-Volume, Long-Short-Ratio}, account/rest-api/
Notional-and-Leverage-Brackets, websocket-market-streams/Liquidation-Order-Streams;
bybit-exchange.github.io/docs/v5/market/{history-fund-rate, tickers, open-interest,
risk-limit, instrument}, /v5/websocket/public/{all-liquidation, liquidation};
binance/binance-futures-connector-python; gerbil.life/papers/CarryTrade.v1.2.pdf;
doi.org/10.1287/mnsc.2024.05069; insights.deribit.com DVOL; developer.whale-alert.io
{documentation, pricing}; lunarcrush.com/developers/api/overview.
