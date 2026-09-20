# Binance USDⓈ-M Futures — public market-data limits (WebSocket + REST)

Researched 2026-08-21. All quotes below are verbatim from raw fetches of
`developers.binance.com` (via `r.jina.ai` text-proxy, which returns the
rendered page text unsummarized — not an LLM) and one live REST call.
Doc pages themselves are dated by Binance as "Last modified on August 21, 2026"
(today), so this is the current live doc, not a stale mirror. Sources are
listed under each fact.

**A live disclosure up front, because it changes the answer to "how many
connections do I need":** the futures WebSocket base URL has just been split
into three routed sub-paths — `/public`, `/market`, `/private` — replacing the
old flat `wss://fstream.binance.com/ws` and `/stream`. The migration notice
states legacy (unrouted) URLs "will remain available until 2026-04-23" — a
date that is **already in the past relative to today (2026-08-21)**. This is
either a documentation date bug on Binance's side or the cutover already
happened. Either way: **do not build against the legacy flat URL scheme.**
Use the routed `/public` and `/market` paths from day one. See the UNVERIFIED
section.

---

## 1. WebSocket limits (`fstream.binance.com`)

Source (primary, "Connect" page):
https://developers.binance.com/docs/derivatives/usds-margined-futures/websocket-market-streams/Connect

> "A single connection can listen to a maximum of **1024** streams."

> "A single connection is only valid for 24 hours; expect to be disconnected
> at the 24 hour mark"

> "The websocket server will send a `ping frame` every 3 minutes. If the
> websocket server does not receive a `pong frame` back from the connection
> within a 10 minute period, the connection will be disconnected. Unsolicited
> `pong frames` are allowed (the client can send pong frames at a frequency
> higher than every 15 minutes to maintain the connection)."

> "WebSocket connections have a limit of **10 incoming messages per second**."

> "A connection that goes beyond the limit will be disconnected; IPs that are
> repeatedly disconnected may be banned."

**Routing (new, live as of today):**

> "Three routed endpoints are available based on data type:
> - **Public** (high-frequency public market data): `wss://fstream.binance.com/public`
> - **Market** (regular market data): `wss://fstream.binance.com/market`
> - **Private** (user data): `wss://fstream.binance.com/private`"

> "Connections that do not include a routed path (`/public`, `/market`, or
> `/private`) will only receive data from the **Public** endpoint. Streams
> belonging to `/market` or `/private` will not push data on unrouted
> connections."

`@depth`, `@depth<levels>`, `@bookTicker`, `!bookTicker` live under `/public`.
`@aggTrade`, `@kline_*`, `@markPrice`, `@ticker`, `@forceOrder`,
`@continuousKline_*` live under `/market`. Mixing them on one unrouted
connection silently drops the `/market` half — the connection will look
healthy while half the tape goes dark.
(Source: https://developers.binance.com/docs/derivatives/usds-margined-futures/websocket-market-streams/Important-WebSocket-Change-Notice)

**Connections per IP / connection-rate limit:** not stated anywhere in the
futures docs I could reach (Connect page, General Info, WebSocket API General
Info, Important-WebSocket-Change-Notice). The commonly cited "300 connection
attempts per 5 minutes per IP" figure is a **spot**-API number, not futures —
see UNVERIFIED.

**Consequence of exceeding limits, as stated:**
- Over 10 incoming messages/sec on one connection → that connection is
  disconnected; repeated offenses → **IP may be banned** (no duration given
  for this specific case in the WS docs).
- Over 1024 streams on one connection → not explicitly stated what happens
  (presumably the subscribe request is rejected or the connection dropped);
  not directly quoted anywhere I found — see UNVERIFIED.

---

## 2. Specific public streams

Source (aggTrade, kline — "Market" category API reference):
https://developers.binance.com/en/docs/catalog/core-trading-derivatives-trading-usd-s-m-futures/api/ws-streams/market

Source (depth streams — "Public" category, full page):
https://developers.binance.com/docs/derivatives/usds-margined-futures/websocket-market-streams

### Trades: only `@aggTrade` exists for USDⓈ-M futures

There is **no `@trade` (raw individual-trade) stream** in the current futures
API reference — I searched the full "Market" stream catalog page (all `##`
headings) and the only trade-related entry is "Aggregate Trade Streams". Spot
has both `@trade` and `@aggTrade`; futures documents only the aggregate one.

> "The Aggregate Trade Streams push market trade information that is
> **aggregated for fills with same price and taking side every 100
> milliseconds**. Only market trades will be aggregated, which means the
> insurance fund trades and ADL trades won't be aggregated."

Stream name: `{symbol}@aggTrade`, update speed **100ms**, subscribed under
`/market`: `wss://fstream.binance.com/market/ws/{symbol}@aggTrade`.

Because it is aggregated (same price + same taking side merged into one
event, with `f`/`l` giving the first/last underlying trade ID range), it is
**not** a tick-by-tick print-by-print record — trades against different
counterparties at the identical price and side within the 100ms window
collapse into one aggTrade event. It is the best available public print feed
on this venue, but it is not "the complete record of prints" in the sense of
one event per fill. Treat `f`..`l` as the count of underlying fills folded in.

### Kline/candlestick stream

> "The Kline/Candlestick Stream push updates to the current klines/candlestick
> every **250 milliseconds** (if existing)."

Stream: `{symbol}@kline_{interval}`, subscribed under `/market`. `interval`
enum includes `1m` (plus `3m,5m,15m,30m,1h,2h,4h,...` up to `1M`).

**Closed-candle flag (`x`):** the payload documents field `x` as `boolean`,
"Is this kline closed?" — but the docs give **no reliability statement** about
`x` (no guarantee it fires exactly once, no note about duplicate/late
"closed" events). I could not find any Binance-authored caveat either way in
the pages I fetched. Given the stream pushes updates every 250ms rather than
only on close, a consumer must not assume one `x:true` event per interval —
it should instead treat `x:true` as "this is the terminal update for this
`t`" and dedupe on `(t, x)`. This reliability question is UNVERIFIED against
official docs; see below.

### Partial book depth streams

Source: full "Public" streams page (as above).

> "Top bids and asks" — `{symbol}@depth{levels}@{updateSpeed}`
> Levels enum: **5, 10, 20**
> Update speed enum: **100ms, 500ms** (the visible UI list also showed
> "250ms or 500ms or 100ms" as the stated update-speed line on the same
> stream card — the enum block itself only lists `100ms`/`500ms`; treat 250ms
> partial-depth as unconfirmed, see UNVERIFIED)

### Diff. (differential) book depth streams

> "Bids and asks, pushed every **250 milliseconds, 500 milliseconds, 100
> milliseconds** (if existing)."
> Stream: `{symbol}@depth@{updateSpeed}`

**Partial vs diff depth, the actual difference:**
- **Partial** (`@depth{levels}@speed`) sends a fresh top-N snapshot each tick
  — self-contained, no local book maintenance needed, but you only ever see N
  levels (5/10/20).
- **Diff** (`@depth@speed`) sends incremental updates (`U`, `u`, `pu`,
  bid/ask deltas with 0-quantity meaning "remove level") against a REST
  snapshot (`GET /fapi/v1/depth`) that the client must fetch once and then
  apply the stream on top of, per Binance's own documented procedure:

  > "1. Open a stream to `wss://fstream.binance.com/public/stream?streams=btcusdt@depth`.
  > 2. Buffer the events... 3. Get a depth snapshot from
  > `https://fapi.binance.com/fapi/v1/depth?symbol=BTCUSDT&limit=1000`.
  > 4. Drop any event where `u` is < `lastUpdateId` in the snapshot.
  > 5. The first processed event should have `U <= lastUpdateId AND u >= lastUpdateId`
  > ... each new event's `pu` should equal the previous event's `u`, otherwise
  > re-initialize from step 3."
  (Source: https://developers.binance.com/docs/derivatives/usds-margined-futures/websocket-market-streams/How-to-manage-a-local-order-book-correctly)

  Diff depth gives full-depth reconstruction capability but requires this
  snapshot+apply bootstrap and re-sync logic; partial depth is simpler but
  capped at 20 levels.

---

## 3. REST limits (`fapi.binance.com`)

Source (general): https://developers.binance.com/docs/derivatives/usds-margined-futures/general-info
Source (rate limiter enum): https://developers.binance.com/docs/derivatives/usds-margined-futures/common-definition
Source (per-endpoint weight): https://developers.binance.com/en/docs/catalog/core-trading-derivatives-trading-usd-s-m-futures/api/rest-api/market-data

> "The `/fapi/v1/exchangeInfo` `rateLimits` array contains objects related to
> the exchange's `RAW_REQUEST`, `REQUEST_WEIGHT`, and `ORDER` rate limits."

> "Every request will contain `X-MBX-USED-WEIGHT-(intervalNum)(intervalLetter)`
> in the response headers which has the current used weight for the IP for
> all request rate limiters defined."

**Confirmed live** — I called `GET https://fapi.binance.com/fapi/v1/exchangeInfo`
directly (2026-08-21) and the response headers included:

```
x-mbx-used-weight-1m: 2
```

matching the documented header name pattern.

**IP weight budget, from the live `exchangeInfo.rateLimits` array (fetched
2026-08-21):**

```json
[
  {"rateLimitType": "REQUEST_WEIGHT", "interval": "MINUTE", "intervalNum": 1, "limit": 2400},
  {"rateLimitType": "ORDERS", "interval": "MINUTE", "intervalNum": 1, "limit": 1200},
  {"rateLimitType": "ORDERS", "interval": "SECOND", "intervalNum": 10, "limit": 300}
]
```

So: **2400 request-weight per IP per 1-minute window.** (Order-count limits
are per-account, not relevant to public market data.) The docs' own example
payload for the ENUM definition of `REQUEST_WEIGHT` independently shows the
identical number:

> ```json
> {"rateLimitType": "REQUEST_WEIGHT", "interval": "MINUTE", "intervalNum": 1, "limit": 2400}
> ```

**Weight per relevant endpoint** (all quoted verbatim from the REST API
market-data catalog page):

- `GET /fapi/v1/exchangeInfo` — **IP Weight 1**
  > "IP Weight — 1"

- `GET /fapi/v1/klines` — weight depends on the `limit` query parameter:
  > "LIMIT weight
  > [1,100) 1
  > [100, 500) 2
  > [500, 1000] 5
  > > 1000 10"

- `GET /fapi/v1/depth` (order-book snapshot) — weight depends on `limit`:
  > "Limit Weight
  > 5, 10, 20, 50 2
  > 100 5
  > 500 10
  > 1000 20"

**Headers, exact fields as documented:**

> "Every request will contain `X-MBX-USED-WEIGHT-(intervalNum)(intervalLetter)`
> in the response headers."

I found **no documented `Retry-After` header** for futures 429/418 responses
anywhere in the general-info or error-code pages I fetched. The ban duration
is instead embedded in the error-message text itself:

> "-1003 TOO_MANY_REQUESTS
> Too many requests; current limit is %s requests per minute. Please use the
> websocket for live updates to avoid polling the API.
> Way too many requests; IP banned until %s. Please use the websocket for
> live updates to avoid bans."

(the `%s` is Binance's own template placeholder for the actual number/epoch
— not filled in by the docs). Whether a `Retry-After` HTTP header is also
sent is UNVERIFIED — I did not want to deliberately trigger a 429/418 to check.

---

## 4. Ban behaviour: 418 vs 429

Source: https://developers.binance.com/docs/derivatives/usds-margined-futures/general-info

> "HTTP `429` return code is used when breaking a request rate limit."

> "HTTP `418` return code is used when an IP has been auto-banned for
> continuing to send requests after receiving `429` codes."

> "When a 429 is received, it's your obligation as an API to back off and not
> spam the API."

> "**Repeatedly violating rate limits and/or failing to back off after
> receiving 429s will result in an automated IP ban (HTTP status 418).**"

> "**IP bans are tracked and scale in duration for repeat offenders, from 2
> minutes to 3 days.**"

> "**The limits on the API are based on the IPs, not the API keys.**"

So: 429 = "you crossed the weight limit right now, back off"; 418 = "you kept
going after 429s, you are now banned." Escalating repeat-offender duration:
**2 minutes minimum, up to 3 days maximum**, per IP, not per key. There is no
API key involved in public market data at all, so this is purely an IP-level
concern for a data-capture process — running multiple capture processes
behind the same NAT/IP shares this budget and this ban.

---

## 5. Symbol universe

Endpoint: `GET https://fapi.binance.com/fapi/v1/exchangeInfo` (weight 1, no
API key required — confirmed above).

**Live counts, fetched 2026-08-21 from the real endpoint (not a doc example):**

- Total symbols returned in `exchangeInfo.symbols`: **872**
- Symbols with `status: "TRADING"`: **744**
- Symbols with `status: "SETTLING"`: 127; `"PENDING_TRADING"`: 1
- Symbols with `contractType: "PERPETUAL"` (any status): **698**
- Symbols with `contractType: "PERPETUAL"` **and** `status: "TRADING"`: **570**

**570 is the number that matters for "how many perpetual symbols am I
actually capturing right now."** 872 includes dated/quarterly contracts
(`CURRENT_QUARTER`, `NEXT_QUARTER`, `CURRENT_MONTH`, `NEXT_MONTH`,
`PERPETUAL_DELIVERING`) and non-trading statuses. These figures are a live
measurement at fetch time, not a stated document constant — Binance lists new
perpetuals frequently, so this number drifts and should be treated as "as of
2026-08-21," re-measured at deploy time, not hardcoded.

---

## UNVERIFIED

1. **Connections-per-IP and connection-attempt-rate limit for futures
   specifically.** I could not find this stated anywhere in the futures
   `Connect`, `General Info`, `WebSocket API General Info`, or
   `Important-WebSocket-Change-Notice` pages. A commonly repeated figure
   ("300 connection attempts per 5 minutes per IP") exists in Binance's
   **spot** market documentation, but I have no confirmed futures-specific
   number and will not carry the spot figure across products as a guess.

2. **What exactly happens when a connection tries to exceed the 1024-stream
   cap** — rejected subscribe request vs. hard disconnect. Not stated in the
   quoted text I found.

3. **Whether the legacy (unrouted) `wss://fstream.binance.com/ws` /
   `/stream` URLs still work today.** The migration notice's own stated
   decommission date, "2026-04-23," is before today's date (2026-08-21),
   which is internally inconsistent with the page also being stamped "Last
   modified on August 21, 2026." I did not test the legacy URL live to avoid
   depending on behavior Binance may be actively killing. Build against the
   routed `/public` and `/market` paths regardless — that path is documented
   as correct either way.

4. **Whether `Retry-After` is actually sent on 429/418 responses.** Not
   documented in the pages fetched; not tested live (would require
   deliberately tripping a rate limit / ban).

5. **Reliability of the kline stream's `x` (closed) flag** — no Binance
   statement found, positive or negative, about whether exactly one
   `x:true` event fires per interval, whether it can be followed by a
   correction, or whether it can be missed on a dropped/resubscribed
   connection. The field is documented to exist; its behavioral guarantee is
   not.

6. **Whether partial-book-depth truly supports 250ms** as an update speed.
   The stream's summary line said "Update Speed 250ms or 500ms or 100ms" but
   the enum block for the `updateSpeed` parameter on the same page only
   listed `100ms` and `500ms`. This is a direct contradiction inside
   Binance's own page and I am flagging it rather than picking one.

7. **Whether a `wss://stream.binancefuture.com` alternate WS host** (seen
   referenced by one AI-summarized web search result, not by a raw fetch) is
   current or legacy/testnet-adjacent — not corroborated in any raw doc page
   I fetched. Treat as unconfirmed; go by `fstream.binance.com` only.

---

## What this means for connection budgeting

Plan around **hard, quoted numbers**: 1024 streams/connection, 10
incoming-messages/sec/connection, 24h forced reconnect, 3-min ping/10-min
pong timeout, 2400 request-weight/minute/IP on REST, and 570 live TRADING
perpetuals today (re-measure at deploy — this count moves). The two biggest
open risks are the missing connections-per-IP limit (item 1) and the
apparently-already-past legacy-URL decommission date (item 3) — do not size a
connection pool or choose a WS URL scheme without re-checking both
immediately before build, since the second one may change the required
migration date on no notice. Route every subscription explicitly through
`/public` or `/market` from the start; do not rely on the unrouted default.
