# ccxt vs native websocket clients for continuous futures capture (Binance USDⓈ-M, Bybit v5)

Scope: informs the 13 parts reading live futures trades, 1m candles, and shallow order
book snapshots from Binance USDⓈ-M and Bybit v5 as a permanent tape. This project
(`ajit-segment-bots`) only — no RL-0xx rulings or `~/trading-system` decisions apply.

Method: raw docs and source pulled with `curl`/`WebFetch` against
`raw.githubusercontent.com`, `docs.ccxt.com`, and PyPI's JSON API; issues pulled with
`gh issue view/list --repo ccxt/ccxt` (never web search for issue content, per project
rule). A real install was done with `uv` against this box's actual interpreter
(`/usr/bin/python3.14`, CPython 3.14.4, standard build — not free-threaded), not
inferred from metadata.

---

## 1. The ccxt / ccxt.pro split — current state

**Websocket streaming (`watch*`) is in the free, open-source `ccxt` package. It is not
a separate paid product and has not been since 2022.**

Primary source, `ccxt/ccxt` wiki manual, fetched raw
(`https://raw.githubusercontent.com/ccxt/ccxt/master/wiki/ccxt.pro.manual.md`):

> "CCXT Pro is a free part of CCXT that adds support for WebSocket streaming:
> https://github.com/ccxt/ccxt/issues/15171"

That line appears five times across the docs mirror I pulled — the same sentence is
repeated verbatim on `docs.ccxt.com/docs/pro-manual` — so this is not a stray or stale
line.

The linked issue is the primary record of *when* this changed. Pulled directly with
`gh issue view 15171 --repo ccxt/ccxt`:

> Title: "CCXT Pro Websockets merged with CCXT!" — closed, opened 2022-10-03.
>
> "We are happy to announce the merge of CCXT Pro into CCXT. CCXT Pro is now a part of
> the free CCXT package as of version 1.95+. All of the CCXT Pro Websocket functionality
> is retained in CCXT."

So: **merged at ccxt 1.95 (October 2022)**, current version is **4.5.75** (checked
2026-08-21 via `https://pypi.org/pypi/ccxt/json`) — nearly four years and thousands of
releases past the merge. There is no separate license, key, or subscription gate on
`watch*` methods today.

Confirmed by actually importing it, not just reading about it — real `uv pip install ccxt`
into a venv on `/usr/bin/python3.14` (no paid credentials, no license file):

```
>>> import ccxt.pro as ccxtpro
>>> ex = ccxtpro.binanceusdm()
>>> hasattr(ex, 'watchTrades'), hasattr(ex, 'watchOHLCV'), hasattr(ex, 'watchOrderBook')
(True, True, True)
>>> ex2 = ccxtpro.bybit()
```
Both venue classes instantiate and carry the `watch*` methods with a plain `pip install
ccxt` — no `ccxt.pro` PyPI package exists; `ccxt.pro` is a submodule of the same
distribution (`ccxt.pro.binanceusdm.binanceusdm` subclasses
`ccxt.async_support.binance.binance`, confirmed via `__mro__`).

**This is the load-bearing fact for the architecture question**: the blueprint's "one
unified interface" reading is not blocked by licensing. Whatever the recommendation ends
up being, it is not driven by cost.

---

## 2. Python packaging facts

Source: `https://pypi.org/pypi/ccxt/json`, and a real `uv venv --python /usr/bin/python3.14`
+ `uv pip install ccxt` (both dry-run and real) on this box.

- **Current version: 4.5.75** (as of 2026-08-21).
- **Wheel**: `ccxt-4.5.75-py3-none-any.whl` — pure Python, no compiled extension of its
  own, `requires_python >=3.10`. There is also an sdist, but it is not needed here since
  the wheel is universal.
- **Transitive dependencies actually resolved on this box** (`uv pip install ccxt`,
  real install, not inferred):
  `aiohappyeyeballs, aiohttp, aiohttp-fast-zlib, aiosignal, attrs, certifi, cffi,
  charset-normalizer, cryptography, frozenlist, idna, multidict, orjson, propcache,
  pycparser, requests, typing-extensions, urllib3, uvloop, yarl, zlib-ng` — 22 packages
  total including ccxt itself.
- **All of them installed from prebuilt wheels — zero compilation, verified by running
  the install for real**, not just checking metadata:
  ```
  uv venv /tmp/ccxt_test_venv2 --python /usr/bin/python3.14
  uv pip install ccxt --python /tmp/ccxt_test_venv2/bin/python
  # Installed 22 packages in 830ms — no build step, no compiler invoked
  ```
- One dependency needed a closer look because this box has **no C compiler and no
  sudo**: `coincurve==21.0.0` is pinned in ccxt's metadata but marked
  `python_version < "3.14"` — it is **not** pulled in on 3.14.4, and it has no cp314
  wheel on PyPI at all, so it would have been a real blocker if the marker didn't
  exclude it. Verify this doesn't silently regress on a ccxt version bump.
- `cryptography==50.0.0`'s only wheels tagged for the `cp314` ABI are `cp314t`
  (free-threaded) — but it also ships `cp39-abi3` wheels, which satisfy standard
  CPython 3.14 via the stable ABI, so the resolve is not actually blocked. This is the
  kind of packaging detail that looks like a blocker from metadata alone and isn't;
  I only trust it because the real install above succeeded.
- No `ta-lib`-style situation here: nothing in ccxt's own dependency tree lacks a
  `cp314` (or `cp39-abi3` / `py3-none-any`) wheel today.

---

## 3. Async model

Primary source: `wiki/ccxt.pro.manual.md`, "Instantiation" section (Python subsection):

> "The Python implementation of CCXT Pro relies on builtin
> [asyncio](https://docs.python.org/3/library/asyncio.html) and Event Loop in
> particular. In Python it is possible to supply an asyncio's event loop instance in
> the constructor arguments as shown below (identical to `ccxt.async_support`)."

- It is **stdlib `asyncio`**, not a custom loop or a bundled reactor. `uvloop` is listed
  as a dependency (platform-conditional) and ccxt will use it as an accelerator where
  available, but the programming model is plain `asyncio`.
- **Sync (`ccxt`) and async (`ccxt.pro` / `ccxt.async_support`) clients are separate
  classes with a separate import path** (`import ccxt` vs `import ccxt.pro`), and both
  can be imported and instantiated side by side in the same process — confirmed by
  import in the real venv above. They are not literally the same object switching mode.
- **`while True:` around a `watch*()` call is the documented pattern**, not a
  short-session convenience. Every worked example in the manual — the canonical one,
  quoted directly:
  ```python
  import ccxt.pro as ccxtpro
  from asyncio import run

  async def main():
      exchange = ccxtpro.kraken({'newUpdates': False})
      while True:
          orderbook = await exchange.watch_order_book('BTC/USD')
          print(orderbook['asks'][0], orderbook['bids'][0])
      await exchange.close()
  ```
  and 13 further `while True:` blocks counted in the manual (`grep -c` on the raw
  file). This is architecturally the same shape a permanent-tape capture loop would use
  — the library is designed around long-lived loops, not one-shot calls.
- Reconnection is handled inside the loop, not by the caller restarting it. From the
  manual's "Streaming Specifics" section:
  > "Upon a critical exception, a disconnect or a connection timeout/failure, the next
  > iteration of the tick function will call the `watch` method that will trigger a
  > reconnection... CCXT Pro applies the necessary rate-limiting and exponential
  > backoff reconnection delays. All of that functionality is enabled by default."

---

## 4. Known reliability issues for long-running capture — from ccxt's own issue tracker

Pulled with `gh issue view` / `gh issue list --repo ccxt/ccxt`, not web search.
All four below were **OPEN** at the time of checking (2026-08-21):

**a) Silent stalls after hours of uptime — [#23214](https://github.com/ccxt/ccxt/issues/23214),
"watchOrderBook methods stop returning up to date data after random delay"** (opened
2024-07-25, still open). Reporter, on ccxt 4.3.15, running ~20 concurrent
`watchOrderBook` coroutines:

> "After a few hours I always get the web sockets for one or more exchanges stopping
> returning timely data. Most of the time it just stops returning data but I've also
> seen it keep one returning data on each iteration but with an exchange timestamp that
> is stuck in the past... It's enormously intermittent."

Corroborated by five independent commenters in the thread (`ByTheSeaL`,
`adsonfilipe`, `stone-za`, `ivanmorozwl`, `bernigaud`), across Python and JS, across
Kraken, Coinex, Mexc. One commenter notes it started after an upgrade from 4.1.66 to
4.3.63 and asks whether periodic reconnection is the recommended workaround — no ccxt
maintainer response confirming a fix is in the thread as pulled. **No ccxt-side
watchdog/heartbeat-timeout exists that a caller can rely on** — a caller must build its
own staleness detector.

**b) Memory leak in `watchOrderBook()` — [#26753](https://github.com/ccxt/ccxt/issues/26753)**
(opened 2025-09-01, still open). Root cause as diagnosed by the reporter and partially
acknowledged by a maintainer (`carlosmiei` shipped a fix, reporter says the fix "should
not eliminate it completely" and worked around it by inflating `maxRetriesAttempt`):
a dropped/stale internal websocket client is not always released from memory
(`this.clients[url]` deletion race against `handleOrderBook()` still firing), so a
long-running process can accumulate orderbook cache entries. Directly relevant to
**multi-day** runs — this is not a short-session-only defect.

**c) Trade gaps after reconnect are not backfilled — [#26945](https://github.com/ccxt/ccxt/issues/26945),
"watchTrades does not backfill after WS reconnect (code 1006)"** — filed specifically
against **`binanceusdm`** (this project's exact venue), opened 2025-10-01, still open:

> "When the websocket connection closes with code 1006 on Binance USD-M futures
> (`binanceusdm`), `watchTrades` reconnects but misses all trades that happened during
> the downtime... there is no catch-up: minutes overlapping the disconnect window end
> up with zero volume (and sometimes wrong O/H/L/C if the user aggregates). This is
> expected at the exchange level (Binance WS trade streams don't replay)."

A commenter (`ArturStankevicz`) calls this "the biggest PITA with ccxt right now" and
describes routing around it entirely with a separate buffering layer. **This is not a
ccxt bug to be fixed** — it is a property of the exchange's WS trade stream (no replay)
that ccxt does not paper over. Any capture system, ccxt or native, needs its own REST
gap-fill on reconnect if trade completeness matters for the tape.

**d) `watchOrderBook` does maintain local book state and does self-heal on a detected
gap — but only if the gap is detected.** From the manual (`ccxt.pro.manual.md`,
"Streaming Specifics"):

> "the application listening on the client-side has to keep a local snapshot of the
> data in memory and merge the updates received from the exchange server into the
> local snapshot."

ccxt does this merging for the caller (checksum validation is on by default per the
maintainer's own comment in a live thread — see below), but issue
[#28466](https://github.com/ccxt/ccxt/issues/28466), "Binance Spot Order Book
Incremental Updates Cause Price Drift in watch_order_book" (opened 2026-04-27, open),
is worth reading for the mechanism even though it did not end up confirmed as a real
bug: a maintainer (`pcriadoperez`) explained the actual recovery path —

> "With the default `watchOrderBook.checksum=true`, any missing/out-of-sequence depth
> event raises `ChecksumError`, the local book is invalidated, and the next
> `watch_order_book` call re-fetches the snapshot, so drift from a dropped message
> shouldn't be possible."

— and the reporter's own follow-up data showed `nonce_diff` of 1-2 consistently, which
reads as a benign REST-vs-WS timing artifact rather than real drift; the thread does not
reach a confirmed-bug resolution. Included here because it is the clearest primary-source
statement of how gap detection actually works, not because it demonstrates a defect.

---

## 5. The realistic alternative: ccxt for REST, native websocket client for streams

**What ccxt does for you that a hand-rolled websocket client does not:**

- **Per-venue message normalisation** — Binance USDⓈ-M's futures depth-update JSON
  shape and Bybit v5's `orderbook.50.SYMBOL` topic shape are structurally different
  (different field names, different diff-vs-snapshot conventions, different sequence
  number semantics — `pu`/`u` on Binance vs `u`/`seq` on Bybit). ccxt's `watchOrderBook`
  normalises both into the same unified `{bids, asks, timestamp, nonce}` shape. Losing
  ccxt here means writing and maintaining two parsers, one per venue, including their
  respective gap-detection logic (Binance's `pu` continuity check; Bybit's `seq`/`u`
  check) by hand.
- **Symbol naming** — ccxt maps venue-native symbols (`BTCUSDT` on both, but Bybit v5
  distinguishes linear/inverse/spot categories in the same raw symbol string) to a
  single unified `BTC/USDT:USDT`-style symbol. Without ccxt, the 13 parts downstream of
  capture would need venue-aware symbol parsing wherever they currently assume a
  unified symbol.
- **Precision/limits metadata** — `loadMarkets()` gives tick size, lot size, and
  contract multiplier per symbol per venue from each exchange's own `exchangeInfo`
  endpoint, kept current by ccxt's own market-fetching. A native client would need to
  hit and parse `GET /fapi/v1/exchangeInfo` (Binance) and `GET /v5/market/instruments-info`
  (Bybit) directly and keep that logic in sync as the venues change their schemas.
- **Reconnection/backoff plumbing** — documented above (section 3): exponential backoff,
  ping/pong keepalive (`keepAlive`, `maxPingPongMisses` are configurable exchange
  properties per the manual), and channel re-subscription on reconnect are handled
  inside `watch*()`. A native client needs to reimplement this, though it is the
  least venue-specific of the four and the most commonly available in generic websocket
  libraries.

**What is NOT lost by the split** (REST via ccxt, streaming via a native client): the
REST half (symbol lists, historical klines, snapshots for cold-start/gap-fill) keeps
100% of ccxt's normalisation value with none of the streaming-specific risk in section 4,
since REST calls are short-lived request/response, not long-running connections that can
silently stall over multi-day runs.

**What is lost, concretely, if streaming moves to a native client**: the four items
above must be rewritten per venue — two order-book diff appliers with two different gap
semantics, two symbol/precision mappers (or reuse of ccxt's `loadMarkets()` purely for
metadata while writing your own stream parser, which is a real middle option), and a
reconnect/backoff/keepalive loop per venue. This is bounded work — two venues, three
data types (trades/candles/book) — not an open-ended rewrite, but it is real ongoing
maintenance surface that ccxt currently owns for free across 100+ exchanges.

---

## UNVERIFIED

1. **Multi-day memory growth, quantified.** I found a confirmed, still-open memory-leak
   issue (#26753) but no measurement of growth rate (MB/day) for the specific
   `binanceusdm` + `bybit` combination this project needs. UNVERIFIED — no data source
   for that number exists in the issue tracker as searched; would need to be measured
   directly against this project's own long-running process.
2. **Whether Bybit v5-specific `watchOrderBook`/`watchTrades` have their own open
   reliability issues beyond the general ones found.** `gh issue list --search "bybit
   watchOrderBook"` returned only one tangential hit (#23251, `watchOrderBookForSymbols`
   missing updates, exchange-agnostic). UNVERIFIED whether this means Bybit is
   genuinely more stable in ccxt than Binance, or whether it is simply less searched/
   less reported — issue volume is not a reliability measurement.
3. **Whether the checksum/gap-detection path in section 4(d) is actually exercised
   correctly for `binanceusdm` and `bybit` specifically** (as opposed to the general
   `binance` spot class discussed in #28466). The manual documents the mechanism
   project-wide; I did not find or run a test proving it fires correctly on the
   USDⓈ-M or Bybit v5 order-book diff format specifically. UNVERIFIED — would require a
   live capture run against real market data with an induced gap.
4. **`uvloop` interaction with this project's own runtime substrate** (already decided:
   standard CPython 3.14.4, file-backed `numpy.memmap`, SQLite/WAL). I confirmed
   `uvloop==0.22.1` resolves and installs with a `cp314` wheel, but did not test running
   it inside this project's actual governor/process-per-part model. UNVERIFIED whether
   there is any interaction between ccxt's asyncio+uvloop usage and the transistor-rule
   process model (T-3 "off means genuinely off") — that is a design question this
   research does not answer.
5. **Bybit v5's own rate limits and WS connection limits per IP** were not checked
   against ccxt's built-in rate limiter configuration for `bybit` specifically — only
   the general leaky-bucket/rolling-window mechanism described in the README was
   confirmed. UNVERIFIED whether ccxt's defaults for `bybit` are tuned correctly for
   this project's expected connection count (2 venues × 3 data types, times however
   many symbols the futures vertical needs).

---

## Recommendation (judgement, not sourced fact — separated deliberately)

The licensing objection is gone: `watch*` streaming has been in the free `ccxt` package
since 2022, four years and ~3,000 releases ago, so "one unified interface" via ccxt is
not blocked by cost or license, and the blueprint's phrasing is achievable literally.

That said, my read of the evidence in section 4 is that **ccxt's streaming half carries
real, currently-open reliability risk for a "permanent tape, continuous, multi-day"
use case specifically**: a documented silent-stall failure mode with no built-in
watchdog (#23214), a memory leak whose fix a reporter says is incomplete (#26753), and
a confirmed no-backfill gap on reconnect that is intrinsic to the exchange, not ccxt,
but which ccxt does not paper over (#26945, filed against this exact venue,
`binanceusdm`). None of these are exotic edge cases — they are the specific failure
modes a permanent tape is built to avoid.

My recommendation is the split described in section 5: **ccxt for the REST half**
(symbol/market metadata, historical klines for backfill, precision/limits), where its
value is high and its risk surface is the low-risk request/response path, and a **native
per-venue websocket client for the streaming half**, with your own staleness watchdog,
your own reconnect/backoff, and your own REST-backed gap-fill on reconnect — which you
need to write *regardless* of which library you stream with, since #26945 shows ccxt
does not solve it either.

**What would change this recommendation**: a demonstrated fix (not just a partial one)
for #23214 and #26753 in a recent ccxt release, run against this project's own
multi-day capture workload without a stall or unbounded memory growth. If that
verification is ever done and passes, ccxt end-to-end for both REST and streaming
becomes the simpler, lower-maintenance choice — the unified-interface value described
in section 5 is real and worth reclaiming once the reliability question is answered by
measurement rather than by reading other users' issue reports.
