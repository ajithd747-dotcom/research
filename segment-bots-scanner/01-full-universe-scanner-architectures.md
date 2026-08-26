# How real systems scan a full crypto-perpetual universe, and what breaks

**Researched 2026-08-26 by a `sonnet` subagent (Rule 1), for the ajit-segment-bots
universal opportunity scanner. Primary sources raw-fetched where the page was static;
Binance's developer portal is a client-rendered SPA that returns 0 bytes to `curl`, so
those figures came from WebFetch cross-checked against two independent WebSearch sets.**

## The headline finding, which contradicts the premise of the question

**Exchange WebSocket limits are not the bottleneck at 1,586 symbols.** Both venues need
a handful of connections, not hundreds.

### Binance USDM (`fstream.binance.com`)
- **1024 streams per connection**, raised from 200 on **2025-07-02**. Change log:
  *"A single connection of maximum streams change from 200 to 1024."*
- Connection lifetime **24 hours hard cap** — server disconnects unconditionally.
- **All-market aggregate streams exist**: `!ticker@arr`, `!miniTicker@arr`,
  `!markPrice@arr` push every symbol in one stream. This is the first-party primitive
  for universe-wide scanning without spending the per-connection stream budget.
- 746 perpetuals x 2 streams = ~1,492 -> **2 connections**.

### Bybit v5 linear (raw-grepped from the static docs page, verbatim)
- **"No args limit for Futures and Spread for now"** — no per-connection topic cap
  for linear (Spot is 10, Options 2000).
- **"you cannot have length of 'args' array over 21,000 characters"** per public connection.
- **"Do not build over 500 connections in 5 minutes."** Counted per WebSocket domain.
- **Ping every 20 seconds**; 10 minutes of inactivity cuts the connection.
- **No all-symbols aggregate stream for linear.** Searched specifically, none found.
  This is a real asymmetry against Binance — on Bybit the cheap first tier must be built
  from periodic REST `/v5/market/tickers` (bulk, all symbols in one call).
- 840 symbols at ~15-20 chars each into 21,000 -> ~1,000-1,300 topics/connection.

## What actually breaks first, ranked for this box

`ulimit -n` on this machine is **524,288** — file descriptors are not the ceiling.
(Check `systemctl --user show ajit-spine | grep LimitNOFILE` separately; a unit does
not inherit an interactive shell's ulimit.)

1. **CPU from per-message Python work in an event-driven per-symbol-callback design.**
   Corroborated by this project's own measured history: 89,747 msg/s at load 28.7
   dropped to 36,974 msg/s at load 12.5 purely by changing *when* parts republish.
2. **The client library, not the exchange.** CCXT issue #16620: `watchTradesForSymbols`
   hit Binance's real 429 (*"Too much request weight used; current limit is 1200 request
   weight per 1 MINUTE"*) at **~200 symbols**, while native Binance WS handled ~1,000.
   The wrapper's batching imposed a ceiling 5x below the exchange's own.
3. **Message-bus fan-out** — already this project's documented failure mode.
4. **Memory per symbol for order books** — no source published a measured figure.
   UNVERIFIED; no number given rather than an invented one.
5. **File descriptors** — not the bottleneck here.

## What the six open-source systems actually do

| System | Pattern | Documented ceiling |
|---|---|---|
| **Freqtrade** | Single process, serial per-pair loop; universe cut by a `VolumePairList` chain sorted on `quoteVolume` | **Its own docs tell users to keep pair count low**: *"usually by running a low number of pairs and having a CPU with a good clock speed."* `VolumePairList` example config uses `"number_assets": 20`. Issue #1046: at 100 pairs "a pair may get processed once every 2-5 minutes" |
| **Hummingbot** | Per-pair `OrderBook` in a dict, one `OrderBookTracker` per connector; "dynamic order book initialization" added to defer loading | No published symbol ceiling. UNVERIFIED |
| **Nautilus Trader** | Rust core, Python control plane, message bus, nanosecond stamps. Architecturally the most serious | No measured live ceiling found. One discussion mentions CPU concern at 50 instruments in a *backtest* — different workload. UNVERIFIED |
| **Jesse** | "Universe selection" documented as a strategy concept, not infrastructure | None found. UNVERIFIED |
| **OctoBot** | Clearest first-party statement of the real lever: *"REST interfaces can handle a limited amount of requests per second... only a limited amount of trading pairs can be handled simultaneously when using a REST interface"* vs WS | "Limitless" is vendor-FAQ language, not a measurement |
| **Barter-rs** | Multithreaded Rust, Tokio, "cache-friendly state management... O(1) constant lookups" | No benchmark or operator report found. UNVERIFIED |

**Reading across all six: none publish "we tested N=2000 and it worked" or "it broke at
N=X." The only concrete first-party numeric ceiling in the entire set is Freqtrade telling
users to keep pair counts low. That absence is itself the finding — these frameworks were
not built or tested for 500-3000-symbol live scanning.**

## How quiet/illiquid symbols are handled

Every system that addresses it at all **tiers by volume**. Nobody treats all symbols
equally at these scales.

- Freqtrade's `VolumePairList`: sort by `quoteVolume`, take top `number_assets`,
  `refresh_period` default 1800s. This is the default mechanism, not an edge case.
- The standard two-tier design: **scan cheap, watch expensive** — one aggregate stream
  as a liquidity/volatility signal, expensive per-symbol depth/trade streams only for
  the subset that clears a bar. On Binance the cheap tier is a first-party primitive;
  on Bybit it has to be built from bulk REST.

## What not to bother with

- **Do not adopt any of the six wholesale.** None was demonstrated at this scale by its
  own authors. Take exactly two ideas: Binance's all-market aggregate streams as the
  cheap first tier, and volume-based tiering for who earns an expensive subscription.
- **Do not worry about exchange stream caps at 1,500 symbols.** The arithmetic says a
  handful of connections. This is the opposite of the usual first assumption.
- **Do not trust CCXT's `watch*ForSymbols` batching** without testing at target N.
- **Most people cap at 20-100 symbols, and the reason is CPU per cycle in a
  single-process design, not exchange limits.** This project's 327-process architecture
  is structurally different from all six — which also means none of their scaling
  behaviour transfers, in either direction.
- **Do not treat more symbols as free because the fd limit is 524,288.** CPU and message
  volume are what contend on this box.

## UNVERIFIED / low-confidence

1. Binance incoming message rate (5 vs 10 msg/sec) — conflicting figures across Binance's
   own SPA pages, neither raw-grepped.
2. Binance "300 connections per 5 min per IP" — confirmed only on the Spot docs page, not
   found stated for `fstream.binance.com`.
3. Binance ping/pong intervals (3 min vs 20s) — two different figures surfaced, could not
   disambiguate for USDM. Bybit's 20s **was** raw-grep confirmed.
4. "~10K concurrent connections per asyncio process" — blog-level (websocket.org), no
   primary benchmark. Directionally consistent with this project's own bus evidence.
5. Per-symbol order-book memory at scale — no source gave a measured number; none invented.
6. Nautilus / Jesse / Barter-rs / Hummingbot real ceilings — absence of a documented number
   is reported as absence, not as "they scale fine."
7. Bybit having zero all-symbols aggregate ticker stream for linear — the `/connect` page
   was raw-grepped, the ticker page was not. Moderate, not high, confidence on this negative.
8. `ajit-spine`'s actual `LimitNOFILE` — the shell's ulimit was checked, not the unit's.

Sources: Binance USDM WS market streams / change log / all-market mini tickers / mark price
for all market; Bybit v5 `ws/connect`; Freqtrade `strategy-customization.md` (raw, develop),
pairlists docs, issue #1046; CCXT issue #16620; NautilusTrader repo + discussion #3736;
Hummingbot connector architecture; OctoBot FAQ; Barter-rs repo; websocket.org connection-limits.
