# Bybit v5 public market-data limits — linear (USDT) perpetual futures

Compiled 2026-08-21 by direct raw fetch of Bybit's live documentation site
(`bybit-exchange.github.io/docs/v5/...`, Docusaurus-rendered static HTML —
the pre-rendered `<article>` content was extracted and stripped of markup,
no summarizing tool was used) plus one live call against the production REST
API for the symbol count. Every figure below is followed by its exact quoted
source sentence and the URL it came from. The docs carry no version/date
stamp of their own; the pages were fetched at 2026-08-21.

---

## 1. WebSocket connection limits

**Public stream endpoint (linear):**
`wss://stream.bybit.com/v5/public/linear` (testnet:
`wss://stream-testnet.bybit.com/v5/public/linear`)
— [Connect](https://bybit-exchange.github.io/docs/v5/ws/connect)

### Max topic subscriptions / args per connection
Quote, [Connect](https://bybit-exchange.github.io/docs/v5/ws/connect), "Public
channel - Args limits" section:

> "Regardless of Perpetual, Futures, Options or Spot, for one public
> connection, you cannot have length of "args" array over 21,000 characters.
> Spot can input up to 10 args for each subscription request sent to one
> connection. Options can input up to 2000 args for a single connection.
> No args limit for Futures and Spread for now"

So for **linear futures specifically there is no stated numeric cap on the
number of topics**, only the blanket **21,000-character** limit on the
serialized `args` array in a single subscribe message. That 21,000-char
limit governs how many topic strings fit in *one subscribe frame*, not how
many subscriptions a connection can hold in total across multiple subscribe
calls — the docs do not state a total-subscriptions-per-connection cap for
futures. (Flagged in UNVERIFIED below.)

### Max connections per IP / connection-rate limit
Quote, [Connect](https://bybit-exchange.github.io/docs/v5/ws/connect), "IP
Limits" section:

> "Do not frequently connect and disconnect the connection. Do not build
> over 500 connections in 5 minutes. This is counted per WebSocket domain."

Quote, [Rate Limit Rules](https://bybit-exchange.github.io/docs/v5/rate-limit),
"Websocket IP limit" section:

> "Do not establish more than 500 connections within a 5-minute window. This
> limit applies to all connections directed to stream.bybit.com as well as
> local site hostnames such as stream.bybit.kz. Do not frequently connect
> and disconnect the connection. Do not establish more than 1,000
> connections per IP for market data. The connection limits are counted
> separately for Spot, Linear, Inverse, and Options markets"

So: **≤500 new connections per IP per rolling 5-minute window** (per
WebSocket domain), and separately **≤1,000 total concurrent connections per
IP for market data**, counted per product line (Linear counted separately
from Spot/Inverse/Options).

### Heartbeat / ping requirement and idle disconnect
Quote, [Connect](https://bybit-exchange.github.io/docs/v5/ws/connect), "How
to Send the Heartbeat Packet":

> "To avoid network or program issues, we recommend that you send the ping
> heartbeat packet every 20 seconds to maintain the WebSocket connection."

Idle-disconnect behaviour, quote from the same page, "Customise Private
Connection Alive Time" section (worded generally, but stated only in the
context of the private/order-entry connections — see caveat below):

> "In general, if there is no "ping-pong" and no stream data sent from
> server end, the connection will be cut off after 10 minutes."

**Caveat:** this 10-minute idle-cutoff sentence sits under the section
documenting `max_active_time` for **private** and order-entry connections,
not under the public-stream section. Whether the same 10-minute idle timeout
applies unmodified to the **public** linear stream is not explicitly
re-stated on the public-topic pages. Treated as UNVERIFIED for public
streams specifically (see UNVERIFIED section).

Ping send format, quote from the same page:

```
ws.send(JSON.stringify({"req_id": "100001", "op": "ping"}));
```

Pong response for linear (contract) public channels, quoted example from the
same page:

```json
{"success": true, "ret_msg": "pong", "conn_id": "465772b1-7630-4fdc-a492-e003e6f0f260", "req_id": "", "op": "ping"}
```

### What happens on exceeding each limit
- HTTP/REST IP limit exceeded → **error text "403, access too frequent"**,
  and the documented remedy: quote,
  [Rate Limit Rules](https://bybit-exchange.github.io/docs/v5/rate-limit),
  "HTTP IP limit":

  > "If you encounter the error "403, access too frequent", it indicates
  > that your IP has exceeded the allowed request frequency. In this case,
  > you should terminate all HTTP sessions and wait for at least 10 minutes.
  > The ban will be lifted automatically."

  So HTTP-side: **≥10 minutes**, auto-lifted, no fixed upper bound stated.
- REST-endpoint (per-UID) rate-limit hit → **error code `10006`**, quote,
  [Error codes](https://bybit-exchange.github.io/docs/v5/error): "10006 Too
  many visits. Exceeded the API Rate Limit." — this is a per-UID limit (see
  §3), not directly applicable to unauthenticated public endpoints, which are
  IP-limited instead (code `10018`, see below).
- IP-level rate limit (distinct code) → **error code `10018`**, quote from
  the same error-code page: "10018 Exceeded the IP Rate Limit."
- WebSocket session sending requests too fast → **error code `20003`**,
  quote from the same page: "20003 Too frequent requests under the same
  session."
- **No documented duration for a WebSocket-specific IP ban** was found (the
  only stated ban duration, "at least 10 minutes," is written under the
  HTTP/REST section). This is UNVERIFIED for WS — see below.

---

## 2. Public topics needed, exact strings and cadence

### Public trade — `publicTrade.{symbol}`
Quote, [Public Trade](https://bybit-exchange.github.io/docs/v5/websocket/public/trade):

> "Subscribe to the recent trades stream. After subscription, you will be
> pushed trade messages in real-time. Push frequency: real-time. Topic:
> publicTrade.{symbol}"

It is **every print, not aggregated** — but a single WS message can batch
multiple trades. Quote from the same page:

> "For Futures and Spot, a single message may have up to 1024 trades. As
> such, multiple messages may be sent for the same seq."

Each trade record carries a `seq` (cross sequence) field for ordering, `i`
(trade ID), `T` (fill timestamp, ms), price `p`, size `v`, taker side `S`.

### 1-minute kline — `kline.1.{symbol}`
Quote, [Public Kline](https://bybit-exchange.github.io/docs/v5/websocket/public/kline):

> "Available intervals: 1 3 5 15 30 (min) 60 120 240 360 720 (min) D (day) W
> (week) M (month). Push frequency: 1-60s. Topic: kline.{interval}.{symbol}"

A closed candle is signalled by the boolean `confirm` field on the kline
data object:

> "tip: If confirm=true, this means that the candle has closed. Otherwise,
> the candle is still open and updating."

So for the 1-minute topic the exact string is `kline.1.{symbol}` (e.g.
`kline.1.BTCUSDT`), pushed at 1–60s cadence while open, and the consumer
must wait for `"confirm": true` on the message to treat that minute as
closed rather than trusting elapsed wall-clock time.

### Orderbook — `orderbook.{depth}.{symbol}`
Quote, [Public Orderbook](https://bybit-exchange.github.io/docs/v5/websocket/public/orderbook),
"Depths" section:

> "Linear & inverse: Level 1 data, push frequency: 10ms. Level 50 data, push
> frequency: 20ms. Level 200 data, push frequency: 100ms. Level 1000 data,
> push frequency: 200ms."

So for linear the available depth levels are **1, 50, 200, 1000**, at push
intervals **10ms / 20ms / 100ms / 200ms** respectively. Topic string, e.g.
`orderbook.50.BTCUSDT`.

**Snapshot vs delta**, quote, same page, "Process snapshot/delta" section:

> "Once you have subscribed successfully, you will receive a snapshot. The
> WebSocket will keep pushing delta messages every time the orderbook
> changes. If you receive a new snapshot message, you will have to reset
> your local orderbook. If there is a problem on Bybit's end, a snapshot
> will be re-sent, which is guaranteed to contain the latest data. To apply
> delta updates: If you receive an amount that is 0, delete the entry. If
> you receive an amount that does not exist, insert it. If the entry exists,
> you simply update the value."

Level-1-specific behaviour, same page:

> "Linear, inverse, spot level 1 data: if 3 seconds have elapsed without a
> change in the orderbook, a snapshot message will be pushed again, and the
> field u will be the same as that in the previous message. Linear, inverse,
> spot level 1 data has snapshot message only"

The `u` (Update ID) field, same page:

> "u integer Update ID. Occasionally, you'll receive "u"=1, which is a
> snapshot data due to the restart of the service. So please overwrite your
> local orderbook. For level 1 of linear, inverse Perps and Futures, the
> snapshot data will be pushed again when there is no change in 3 seconds,
> and the "u" will be the same as that in the previous message."

There is also a `seq` (cross sequence) field, documented as: "You can use
this field to compare different levels orderbook data, and for the smaller
seq, then it means the data is generated earlier."

### Gap detection and resync — the precise answer requested

**The documentation states the reset triggers (a fresh `type: "snapshot"`
message, or `u == 1`) but does not document a client-side procedure for
*detecting* a skipped delta from the `u` field** — e.g. it never states
"delta `u` must equal previous `u` + 1; if not, resubscribe." It only says
the server will re-push a full snapshot "if there is a problem on Bybit's
end" — i.e. gap recovery is asserted to be server-driven, not something the
client is told to verify.

To check whether the reference client implements gap-detection anyway, I
pulled Bybit's own Python SDK (`pybit`) source directly, since the FAQ names
it as the canonical example:
- Quote, [FAQ](https://bybit-exchange.github.io/docs/faq), "How can I
  process WebSocket snapshot and delta messages?": "Please refer to the
  orderbook topic's documentation. Note that if you're using pybit, it
  handles these messages for you and always delivers a complete orderbook.
  Working code examples: Python: `_process_delta_orderbook()` method within
  pybit"

Fetched `https://raw.githubusercontent.com/bybit-exchange/pybit/master/pybit/_websocket_stream.py`,
`_process_delta_orderbook()` (lines 446-489 at fetch time). **It stores
`u` and `seq` from each delta message but never compares the new `u` to the
previous one, and never checks for a gap.** It blindly applies every delta
it receives in arrival order. There is no local integrity check in the
reference SDK.

**Conclusion for the record: a client that only subscribes to a shallow
depth level (e.g. `orderbook.1` or `orderbook.50`) and applies deltas as
they arrive, the way the official SDK does, has no documented and no
reference-implemented way to detect that it silently missed a delta**
(e.g. from a brief network stall) short of Bybit proactively re-pushing a
snapshot. The `u`/`seq` fields exist and are monotonic by construction, but
verifying continuity is left to the integrator; Bybit's own SDK does not do
it. This matters directly for the "can a shallow book capture be trusted as
a record" question in the task: **not without the capturing client
independently checking `u` sequentially itself** (Bybit does not do this for
you, contrary to what the FAQ's phrasing — "pybit... always delivers a
complete orderbook" — might imply; "complete" there refers to point-in-time
completeness after each applied delta, not to gap-free history).

---

## 3. REST limits for the public endpoints needed

### General IP limit (applies to all HTTP traffic, including unauthenticated public endpoints)
Quote, [Rate Limit Rules](https://bybit-exchange.github.io/docs/v5/rate-limit),
"HTTP IP limit":

> "You are allowed to send 600 requests within a 5-second window per IP by
> default. This limit applies to all traffic directed to api.bybit.com,
> api.bybick.com, and local site hostnames such as api.bybit.kz."

### Per-endpoint UID-based rate-limit table
The published [Rate Limit Rules](https://bybit-exchange.github.io/docs/v5/rate-limit)
page's "API Rate Limit Table" only lists **Trade / Position / Account /
Asset / User / Spot Margin Trade / Spread Trading / RFQ / Institutional
Loan** endpoint groups. **`GET /v5/market/kline`,
`GET /v5/market/orderbook`, and `GET /v5/market/instruments-info` — the
three public market-data endpoints this system needs — do not appear
anywhere in that per-endpoint table.** Quote, same page, framing the whole
table: "The API Rate Limit is based on the rolling time window per second
and UID. In other words, it is per second per UID." Since public market-data
calls are unauthenticated, this per-UID framing does not apply to them; the
only documented control over these three endpoints is the blanket **600
req / 5s per IP** figure above. This is UNVERIFIED as a per-endpoint number
— see below.

### Rate-limit response headers
Quote, [Rate Limit Rules](https://bybit-exchange.github.io/docs/v5/rate-limit),
"API Rate Limit" section:

> "Every request to the API returns response header shown in the code
> panel: X-Bapi-Limit-Status - your remaining requests for current endpoint.
> X-Bapi-Limit - your current limit for current endpoint.
> X-Bapi-Limit-Reset-Timestamp - the timestamp indicating when your request
> limit resets if you have exceeded your rate_limit."

Example given on the same page:

```
X-Bapi-Limit: 10
X-Bapi-Limit-Status: 9
X-Bapi-Limit-Reset-Timestamp: 1672738134824
```

The window these headers describe is **1 second**, per the "rolling time
window per second and UID" statement above — but again that statement is in
the authenticated-endpoint context; whether these headers are populated
(and with what window) on the three public market-data calls was not
confirmed by documentation text — see UNVERIFIED.

---

## 4. Ban behaviour — rejection vs IP ban, codes, duration

| Trigger | Error signal | Documented duration | Source |
|---|---|---|---|
| Per-UID API rate limit exceeded (authenticated endpoints) | `retCode: 10006`, `"Too many visits!"` / doc text "Too many visits. Exceeded the API Rate Limit." | Not stated as a ban; resets on the rolling per-second window (`X-Bapi-Limit-Reset-Timestamp`) | [Rate Limit Rules](https://bybit-exchange.github.io/docs/v5/rate-limit), [Error codes](https://bybit-exchange.github.io/docs/v5/error) |
| Per-IP HTTP rate limit exceeded | `"403, access too frequent"` (also generically code `10018` "Exceeded the IP Rate Limit" per the error-code table) | **"at least 10 minutes"**, auto-lifted | [Rate Limit Rules](https://bybit-exchange.github.io/docs/v5/rate-limit) |
| Too many WS connections built too fast (>500/5min or >1000 concurrent/IP) | Not stated as a distinct error code; docs only phrase it as a "do not" rule | **Not stated** | [Connect](https://bybit-exchange.github.io/docs/v5/ws/connect), [Rate Limit Rules](https://bybit-exchange.github.io/docs/v5/rate-limit) |
| Too-frequent requests inside one WS session | `op: "ping"`/subscribe flood → code `20003` "Too frequent requests under the same session" | Not stated | [Error codes](https://bybit-exchange.github.io/docs/v5/error) |

There is **no single documented "IP ban" mechanism distinct from the
403/10-minute HTTP throttle** — the docs describe the HTTP case precisely
(10 minutes, auto-lift) but never describe a WS-specific ban duration or
whether repeated violations escalate the ban length. Treat any WS-side ban
duration as UNVERIFIED.

---

## 5. Symbol universe — linear USDT perpetual futures

Endpoint: `GET /v5/market/instruments-info?category=linear`
— [Get Instruments Info](https://bybit-exchange.github.io/docs/v5/market/instrument)

Documentation quote (a stated lower bound, not an exact count):

> "This endpoint returns 500 entries by default. There are now more than
> 500 linear symbols on the platform. As a result, you will need to use
> cursor for pagination or limit to get all entries."

**Live measurement** (this is a real-time API call, not a documentation
fetch — timestamped and reproducible, not a documented guarantee): called
`GET https://api.bybit.com/v5/market/instruments-info?category=linear&limit=1000`
on 2026-08-21 at server-reported `time: 1787312967451` ms
(2026-08-21T11:49:27.451Z), single page, no `nextPageCursor` returned. Of
833 total `linear`-category, `status: "Trading"` entries returned:

| quoteCoin | contractType | count |
|---|---|---|
| USDT | LinearPerpetual | **725** |
| USDT | LinearFutures (dated/quarterly, not perpetual) | 40 |
| USDC | LinearPerpetual | 68 |

So **725 is the live count of USDT-quoted linear perpetual symbols
currently trading**, as of the query above. This is a point-in-time count,
not a documented constant — Bybit lists and delists symbols continuously,
so this number should be re-measured at deploy time and periodically
thereafter, not hardcoded.

---

## UNVERIFIED

1. **Total subscribed-topics cap per connection for linear/futures.** The
   docs state "No args limit for Futures and Spread for now" and a 21,000
   character cap on a single subscribe message's `args` array, but do not
   state whether repeatedly sending subscribe messages to accumulate, say,
   10,000 total live subscriptions on one connection is permitted or has an
   undocumented ceiling. Could not confirm either way from the docs.
2. **Whether the 10-minute idle-disconnect rule (no ping-pong, no data) that
   is documented under the private/`max_active_time` section also governs
   the public linear stream unmodified.** The sentence is worded generally
   ("In general, if there is no ping-pong...") but sits physically inside
   the private-connection-customization section of the page, not the public
   stream section. Not re-stated on the public trade/kline/orderbook pages.
3. **Duration of any IP ban specifically for violating the WebSocket
   connection limits** (>500 connections/5min, or >1,000 concurrent
   connections/IP for market data). Only the HTTP 403 case has a stated
   duration ("at least 10 minutes"); no WS-specific figure was found.
4. **Whether escalating/repeated violations lengthen the ban** (HTTP or WS).
   Not addressed anywhere in the fetched pages.
5. **Rate limit (and headers) actually applying to the three public
   market-data REST endpoints** (`/v5/market/kline`,
   `/v5/market/orderbook`, `/v5/market/instruments-info`). They are absent
   from the published per-endpoint rate-limit table entirely; the only
   figure confirmed to apply is the blanket 600-req/5s-per-IP HTTP limit.
   Whether `X-Bapi-Limit`/`X-Bapi-Limit-Status` headers are even populated
   on these public, unauthenticated calls was not confirmed by doc text (no
   response header example is shown for a public-endpoint call).
6. **A documented, client-actionable gap-detection procedure for orderbook
   deltas** (e.g. "u must equal previous u + 1"). Confirmed absent from both
   the docs and Bybit's own reference Python SDK (`pybit`) — see §2. Marked
   UNVERIFIED as "does this exist at all," not merely "could not find it" —
   the SDK source check makes it likely gap detection genuinely is not
   provided and must be built by the consumer.

---

## What this means for connection budgeting

Capacity planning against these numbers should assume the **conservative,
stated figures only**, and build in slack against everything marked
UNVERIFIED rather than assuming the friendlier interpretation. Concretely:
the hard, confirmed ceiling to plan against is **500 new WS connections per
IP per rolling 5 minutes** and **1,000 concurrent per IP for market data**,
with **no stated total-subscription cap for futures topics** but a
**21,000-character ceiling per individual subscribe frame** (meaning many
symbols must be split across multiple subscribe calls on the same
connection, not necessarily across multiple connections). Ping every 20
seconds per the documented recommendation, and do not rely on any
documented idle-timeout number for the public stream specifically since
that figure was only confirmed for private/order-entry connections
(item 2 above) — treat idle disconnection as something to detect
empirically rather than schedule against a trusted number. On REST, plan
against the blanket 600-req/5s-per-IP ceiling since no narrower documented
number exists for kline/orderbook/instruments-info specifically, and budget
for a minimum 10-minute lockout if that ceiling is crossed. Most
importantly: **do not treat a captured shallow orderbook as gap-free** —
neither Bybit's docs nor its own reference SDK implement or describe a
delta-continuity check, so if this system needs a provably continuous book
record, gap detection (comparing successive `u` values, or minimally the
non-decreasing `seq` field, and re-snapshotting on any discontinuity) has to
be built by this project, not assumed from the feed. The 725-symbol USDT
linear-perpetual universe is a live count that should be re-measured at
deploy time and periodically after, not hardcoded, since Bybit lists and
delists symbols continuously.
