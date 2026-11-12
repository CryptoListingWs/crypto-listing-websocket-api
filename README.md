# CLW (cryptolisting.ws) — crypto listing announcement WebSocket API

CLW streams exchange announcements — new listings, delistings, Binance HODLer airdrops, Binance Monitoring Tag changes, Korean caution releases and Robinhood asset additions — from **Binance, Upbit, Bithumb and Robinhood** as structured JSON events over a persistent WebSocket connection. Each event arrives already classified (exchange, event type, ticker), so a bot reads fields instead of parsing notice titles.

- Website: https://cryptolisting.ws/?utm_source=github&utm_medium=referral
- Documentation: https://cryptolisting.ws/docs/book/
- Pricing, including the two free tiers: https://cryptolisting.ws/pricing/?utm_source=github&utm_medium=referral
- Recent announcements, public page: https://cryptolisting.ws/latest-listings/

Operating since April 2026. CLW is a technical data feed, not financial advice, and is not affiliated with Binance, Upbit, Bithumb or Robinhood.

## Coverage

| Exchange | Events (`listingType`) |
| --- | --- |
| Binance | `spot_listing`, `futures_listing`, `spot_delisting`, `futures_delisting`, `hodler_airdrop`, `monitoring_tag_extend`, `monitoring_tag_remove`, `not_listing` |
| Upbit | `spot_listing` (with a `markets` field: KRW, BTC, USDT), `spot_delisting`, `caution_released` |
| Bithumb | `spot_listing`, `spot_delisting`, `caution_released` |
| Robinhood | `wallet_listing` (added to Robinhood assets, before trading opens), `cex_listing` (tradable on Robinhood) |

Full list and edge cases: [Message Reference](https://cryptolisting.ws/docs/book/message-reference.html).

## Endpoints

| URL | Region | Exchanges |
| --- | --- | --- |
| `wss://jp.cryptolisting.ws` | AWS Tokyo | Binance, Upbit, Bithumb |
| `wss://kr.cryptolisting.ws` | AWS Seoul | Upbit, Bithumb |
| `wss://us.cryptolisting.ws` | AWS N. Virginia | Robinhood |

- One API key works on all three endpoints. Pick the endpoint closest to where your bot runs.
- Filter any endpoint by exchange with `?cex=`, for example `wss://jp.cryptolisting.ws?cex=binance,upbit`.
- Authenticate with the `X-API-Key` header. A key passed as a query parameter is rejected (HTTP 401).
- Frames are binary; each payload is UTF-8 JSON.

## Quick start

### 1. Get a key

A free `SpeedTrial` key (15 days, renewable) or a `FreeDelayed` key (first month free) is issued on request: message [@CLWebsocket](https://telegram.me/CLWebsocket) on Telegram or write to contact@cryptolisting.ws. Keys look like `dsk_` followed by 64 hexadecimal characters.

### 2. Connect

Python:

```bash
pip install websocket-client
```

```python
import json, websocket

URL    = "wss://jp.cryptolisting.ws"
HEADER = ["X-API-Key: dsk_your_key_here"]

def on_message(ws, message):
    data = json.loads(message)
    if data["type"] == "announcement":
        tickers = data["ticker"].split(",")   # one alert can carry several tokens
        print(f"{data['listingType']} | {','.join(tickers)} | {data['publisher']}")
        print(f"  {data['title']}")

websocket.WebSocketApp(URL, header=HEADER, on_message=on_message).run_forever()
```

Node.js:

```bash
npm install ws
```

```javascript
const WebSocket = require("ws");

const ws = new WebSocket("wss://jp.cryptolisting.ws", {
  headers: { "X-API-Key": "dsk_your_key_here" },
});

ws.on("message", (raw) => {
  const data = JSON.parse(raw);
  if (data.type === "announcement") {
    const tickers = data.ticker.split(",");   // one alert can carry several tokens
    console.log(`${data.listingType} | ${tickers.join(",")} | ${data.publisher}`);
    console.log(`  ${data.title}`);
  }
});
```

Production-ready clients with reconnection, in Python, Node.js, Go and Rust: [Code Examples](https://cryptolisting.ws/docs/book/code-examples.html).

### 3. Handle the stream

- On connect the server sends a `welcome` message (your tier and limits), then `announcement` messages when events are detected and a `heartbeat` every 30 seconds. `changelog` and `renewal_notice` messages can also arrive.
- Always switch on `type` and ignore values you do not know: new message types can ship without notice.
- Always split `ticker` on `,`: one notice that lists several tokens arrives as one message.
- To test your parser without waiting for a real event, send `{"type":"test"}`; the server answers with a `test_announcement` (at most once per minute per key).
- Treat 60 seconds without any frame (no PING, no `heartbeat`) as a dead connection and reconnect. See [Error Handling & Reconnection](https://cryptolisting.ws/docs/book/error-handling.html).

## Message format

An `announcement` looks like this:

```jsonc
{
  "type": "announcement",
  "title": "Binance Will List TOKEN (TOKEN)",
  "ticker": "TOKEN",
  "publisher": "binance",
  "listingType": "spot_listing"
  // plus detectedTimestampUs, dispatchTimestampUs and abnormalDetectionLatency (see below)
}
```

| Field | Type | Meaning |
| --- | --- | --- |
| `type` | string | `announcement` for a detected event |
| `title` | string | The exchange's original title |
| `ticker` | string | Token symbol(s), comma-separated when a notice lists several |
| `publisher` | string | `binance`, `upbit`, `bithumb` or `robinhood` |
| `listingType` | string | Event type (see Coverage) |
| `detectedTimestampUs` | integer | Detection time, microseconds since the Unix epoch |
| `dispatchTimestampUs` | integer | Time the message was sent, microseconds since the Unix epoch |
| `abnormalDetectionLatency` | boolean | `true` = detected under unusual conditions: handle with care |
| `markets` | string | Optional, Upbit spot listings only: the quote markets opened, e.g. `KRW,BTC` |

Machine-readable definition of every message: [AsyncAPI schema](https://cryptolisting.ws/docs/asyncapi.yaml).

## Tiers

| Tier | Content | Delivery | Price |
| --- | --- | --- | --- |
| SpeedTrial | `ticker` and `title` redacted (for benchmarking) | +1 ms courtesy delay | Free (15-day key, renewable) |
| FreeDelayed | Full feed | +240 ms delay | Free for the first month, then 200 USDT per month, all exchanges included |
| Basic | Full feed | +20 ms delay | From 200 USDT per exchange per month |
| Premium | Full feed | No delay | From 350 USDT per exchange per month |

Basic and Premium are billed per exchange in USDT, with bundle and duration discounts; they require a short identity verification (KYC) and are not available in every country. Current prices and the calculator: https://cryptolisting.ws/pricing/?utm_source=github&utm_medium=referral.

## Documentation

- [Quick Start](https://cryptolisting.ws/docs/book/quick-start.html)
- [WebSocket API](https://cryptolisting.ws/docs/book/websocket-api.html)
- [Authentication](https://cryptolisting.ws/docs/book/authentication.html)
- [Message Reference](https://cryptolisting.ws/docs/book/message-reference.html)
- [Exchange Filtering](https://cryptolisting.ws/docs/book/exchange-filtering.html)
- [Rate Limits & Security](https://cryptolisting.ws/docs/book/rate-limits.html)
- [Error Handling & Reconnection](https://cryptolisting.ws/docs/book/error-handling.html)
- [Changelog](https://cryptolisting.ws/docs/book/changelog.html)

## Guides

- [Upbit WebSocket: market data vs listing announcements](https://cryptolisting.ws/upbit-websocket/) — [한국어](https://cryptolisting.ws/ko/upbit-websocket/)
- [Bithumb WebSocket](https://cryptolisting.ws/bithumb-websocket/) — [한국어](https://cryptolisting.ws/ko/bithumb-websocket/)
- [Binance WebSocket](https://cryptolisting.ws/binance-websocket/) — [한국어](https://cryptolisting.ws/ko/binance-websocket/)
- The site is available in 14 languages, including [한국어](https://cryptolisting.ws/ko/) and [日本語](https://cryptolisting.ws/ja/).

## Contact

- Telegram, free announcement channel: [@CLWfeed](https://telegram.me/CLWfeed)
- Telegram, API keys and support: [@CLWebsocket](https://telegram.me/CLWebsocket)
- E-mail: contact@cryptolisting.ws
- X: [@cryptolistingws](https://x.com/cryptolistingws)
- Legal notice and privacy: https://cryptolisting.ws/legal/

Our only domain is `cryptolisting.ws`; look-alike domains on other extensions are not operated by us.
