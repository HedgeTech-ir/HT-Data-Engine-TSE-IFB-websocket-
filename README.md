# HT Data Engine | TSE & IFB | WebSocket
## Real-Time Market Streams (TSE & IFB)  
### WebSocket • High-Frequency • Low-Latency

---

## 1. Overview

This documentation describes the **HT Data Engine WebSocket API** for subscribing to real-time market data for **TSE & IFB** symbols.  
The API provides multiple channels delivering structured financial data such as order books, aggregate trades, OHLCV, and fund/contract information.  

All WebSocket endpoints require **JWT token-based authentication** and support multiple symbols or ISINs in a single connection.

---

## 2. Authentication

To access protected WebSocket endpoints, a valid **JWT token** is required.

### How to get the token

Send a `POST` request to:

```
https://core.hedgetech.ir/auth/user/token/issue
```

Headers:

```
Content-Type: application/x-www-form-urlencoded
```

Body:

```
username=your_username&password=your_password
```

### Response Example

```json
{
    "Authorization": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

### Usage

Include the token in the WebSocket connection headers:

```
Authorization: <your_token>
```

**Important Notes:**

- Tokens are bound to your IP and browser fingerprint. A change invalidates the token.
- Ensure your account is registered and approved by an admin.
- Unauthorized connections are closed with **WS code 1008**.

---

## 3. WebSocket Endpoints

| Endpoint | Description |
|----------|-------------|
| `wss://core.hedgetech.ir/data-engine/tse-ifb/live/data/websocket/symbol/isin` | Subscribe using **ISIN** codes |
| `wss://core.hedgetech.ir/data-engine/tse-ifb/live/data/websocket/symbol/name` | Subscribe using **symbol names** |

**Note:** The output payload structure is identical for both endpoints, except the symbol identifier field:
- `symbolIsin` for `/symbol/isin`
- `symbolName` for `/symbol/name`

---

### 🔶 Important Clarification: ISIN vs Symbol-Name WebSocket Endpoints

The data engine provides **two separate WebSocket endpoints**:

1. **Subscribe using ISIN codes**
   ```
   wss://core.hedgetech.ir/data-engine/tse-ifb/live/data/websocket/symbol/isin
   ```

2. **Subscribe using Symbol Names**
   ```
   wss://core.hedgetech.ir/data-engine/tse-ifb/live/data/websocket/symbol/name
   ```

Both endpoints deliver **identical payload structures**, including the same channels and the same `data` schema.  
The **only difference** is the identifier field inside each message:

| Endpoint | Identifier Field in Payload |
|----------|------------------------------|
| `/symbol/isin` | `"symbolIsin": "<ISIN>"` |
| `/symbol/name` | `"symbolName": "<Symbol>"` |

Example for ISIN endpoint:

```json
{
  "channel": "best-limit",
  "symbolIsin": "IRO1XYZ1234",
  "timestamp": "2025-11-14T12:00:00.000000",
  "data": { ... }
}
```

Example for Symbol-Name endpoint:

```json
{
  "channel": "best-limit",
  "symbolName": "XYZ",
  "timestamp": "2025-11-14T12:00:00.000000",
  "data": { ... }
}
```

No other structural difference exists between these two WebSocket services.

**Why this clarification matters**

- Consumers might assume that subscribing to the symbol-name endpoint returns a different schema — it does not.
- Client implementations should be prepared to handle either identifier field (`symbolIsin` or `symbolName`) depending on which endpoint they connect to.
- This avoids confusing bugs (for example: looking for `symbolIsin` in messages coming from the `/symbol/name` endpoint).

---

## 4. Connection Flow (Updated)

1. Establish WebSocket connection with the proper `Authorization` header.

2. Include query parameters in the URL. **Each symbol/ISIN and channel is repeated as a separate query parameter**:

```
wss://core.hedgetech.ir/data-engine/tse-ifb/live/data/websocket/symbol/isin?
channels=<channel1>&channels=<channel2>&
symbol_isins=<ISIN1>&symbol_isins=<ISIN2>
```

or for symbol names:

```
wss://core.hedgetech.ir/data-engine/tse-ifb/live/data/websocket/symbol/name?
channels=<channel1>&channels=<channel2>&
symbol_names=<symbol1>&symbol_names=<symbol2>
```

3. If verification passes, the WebSocket connection is accepted.

4. Real-time messages are streamed continuously until the connection is closed.

**Important:** Unauthorized connections are closed immediately with **code 1008**.

---

## 5. Query Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `symbol_isins` | list of strings | List of ISIN codes to subscribe (for `/symbol/isin`) |
| `symbol_names` | list of strings | List of symbol names to subscribe (for `/symbol/name`) |
| `channels` | list of strings | List of channels to subscribe (see Section 6) |

**Example URL:**

```
wss://core.hedgetech.ir/data-engine/tse-ifb/live/data/websocket/symbol/isin?channels=order-book&channels=best-limit&symbol_isins=IRT3SATF0001&symbol_isins=IRTKMOFD0001
```

---

## 6. Channel List & Payload Schemas

All messages are delivered in the following **JSON structure**:

```json
{
    "channel": "best-limit",
    "symbolIsin": "IRO1XYZ1234",
    "timestamp": "2025-11-14T12:00:00.000000",
    "data": { ... channel-specific payload ... }
}
```

> NOTE: For the `/symbol/name` endpoint the `symbolIsin` field above is replaced by `symbolName`. Everything else remains the same.

| Channel | Payload Model | Description |
|---------|---------------|-------------|
| `best-limit` | `BestLimit` | Top buy/sell orders at each level |
| `order-book` | `OrderBook` | Full order book with aggregated price levels |
| `ohlcv-last-1m` | `OHLCV` | 1-minute candlestick data |
| `aggregate` | `Aggregate` | Aggregate trade data including volume, count, prices |
| `institutional-vs-individual` | `InstitutionalVsIndividual` | Buy/sell breakdown between individual and institutional investors |
| `contract-info` | `ContractInfo` | Contract details such as open interest and margin |
| `fund-info` | `FundInfo` | Fund NAV, units, market capitalization, timestamp |

---

### 6.1 Example: `best-limit`

```json
{
  "channel": "best-limit",
  "symbolIsin": "IRO1XYZ1234",
  "timestamp": "2025-11-14T12:00:00.000000",
  "data": {
    "1": {
      "buy_order_count": 5,
      "buy_quantity": 1000,
      "buy_price": 10000,
      "sell_order_count": 3,
      "sell_quantity": 800,
      "sell_price": 10050
    },
    "2": {
      "buy_order_count": 4,
      "buy_quantity": 500,
      "buy_price": 9950,
      "sell_order_count": 2,
      "sell_quantity": 400,
      "sell_price": 10100
    }
  }
}
```

---

### 6.2 Example: `order-book`

```json
{
  "channel": "order-book",
  "symbolIsin": "IRO1XYZ1234",
  "timestamp": "2025-11-14T12:00:00.000000",
  "data": {
      "Buy": [
        {
          "price": 0,
          "quantity": 0,
          "count": 0
        }
      ],
      "Sell": [
        {
          "price": 0,
          "quantity": 0,
          "count": 0
        }
      ]
  }
}
```

---

### 6.3 Example: `ohlcv-last-1m`

```json
{
  "channel": "ohlcv-last-1m",
  "symbolIsin": "IRO1XYZ1234",
  "timestamp": "2025-11-14T12:00:00.000000",
  "data": {
    "open": 10000.0,
    "high": 10100.0,
    "low": 9950.0,
    "close": 10050.0,
    "volume": 3500
  }
}
```

---

### 6.4 Example: `aggregate`

```json
{
  "channel": "aggregate",
  "symbolIsin": "IRO1XYZ1234",
  "timestamp": "2025-11-14T12:00:00.000000",
  "data": {
      "date": "string",
      "time": "string",
      "trade_count": 0,
      "total_volume": 0,
      "total_value": 0,
      "closing_price": 0,
      "last_price": 0,
      "low_price": 0,
      "high_price": 0,
      "open_price": 0,
      "previous_close": 0
  }
}
```

---

### 6.5 Example: `institutional-vs-individual`

```json
{
  "channel": "institutional-vs-individual",
  "symbolIsin": "IRO1XYZ1234",
  "timestamp": "2025-11-14T12:00:00.000000",
  "data": {
      "buy_count_individual": 0,
      "buy_volume_individual": 0,
      "buy_count_institution": 0,
      "buy_volume_institution": 0,
      "sell_count_individual": 0,
      "sell_volume_individual": 0,
      "sell_count_institution": 0,
      "sell_volume_institution": 0
  }
}
```

---

### 6.6 Example: `contract-info`

```json
{
  "channel": "contract-info",
  "symbolIsin": "IRO1XYZ1234",
  "timestamp": "2025-11-14T12:00:00.000000",
  "data": {
      "open_interest": 0,
      "initial_margin": 0,
      "required_margin": 0
  }
}
```

---

### 6.7 Example: `fund-info`

```json
{
  "channel": "fund-info",
  "symbolIsin": "IRO1XYZ1234",
  "timestamp": "2025-11-14T12:00:00.000000",
  "data": {
      "nav": 0,
      "units": 0,
      "marketCap": 0,
      "as_of": "2025-11-14T22:10:42.802Z"
  }
}
```

---

### 7. Other Channels

Payload models follow the **Pydantic models** provided (`Aggregate`, `OrderBook`, `InstitutionalVsIndividual`, `ContractInfo`, `FundInfo`) and always adhere to the format:

```json
{
  "channel": "<channel-name>",
  "symbolIsin": "<symbolIsin>",
  "timestamp": "<ISO8601 timestamp>",
  "data": { ...channel-specific data... }
}
```

> NOTE: Replace `symbolIsin` with `symbolName` when using the `/symbol/name` endpoint.

---

## 8. Error Handling

| Code | Description |
|------|-------------|
| `1008` | Policy violation (invalid JWT, invalid symbols, or invalid channels) |
| Connection closed | Occurs if Redis stream fails or server error |

---

## 9. Examples

### 9.1 Python (WebSocket Client)

This example adds a small helper to **support both endpoints** by checking which identifier field exists in incoming messages.

```python
import asyncio
import websockets
import json

async def subscribe(url: str, token: str):
    headers = {"Authorization": token}
    async with websockets.connect(url, extra_headers=headers) as ws:
        async for message in ws:
            data = json.loads(message)
            # safely extract identifier regardless of endpoint
            symbol = data.get("symbolIsin") or data.get("symbolName")
            channel = data.get("channel")
            ts = data.get("timestamp")
            # process payload...
            print(f"{ts} | {symbol} | {channel} -> {data['data']}")

# example use (choose the endpoint you need)
url = "wss://core.hedgetech.ir/data-engine/tse-ifb/live/data/websocket/symbol/isin?channels=order-book&channels=best-limit&symbol_isins=IRT3SATF0001"
token = "<your_token>"
asyncio.run(subscribe(url, token))
```

### 9.2 JavaScript (WebSocket Client)

```javascript
const WebSocket = require('ws');

function normalizeMessage(message) {
  const msg = JSON.parse(message);
  const symbol = msg.symbolIsin || msg.symbolName;
  return { ...msg, symbol };
}

const url = 'wss://core.hedgetech.ir/data-engine/tse-ifb/live/data/websocket/symbol/isin?channels=order-book&channels=best-limit&symbol_isins=IRT3SATF0001';
const token = '<your_token>';

const ws = new WebSocket(url, { headers: { Authorization: token } });

ws.on('open', () => console.log('Connected'));
ws.on('message', (data) => {
  const message = normalizeMessage(data);
  console.log(message.timestamp, message.symbol, message.channel, message.data);
});
ws.on('close', () => console.log('Disconnected'));
```

### 9.3 Go (WebSocket Client)

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
    "github.com/gorilla/websocket"
)

func main() {
    url := "wss://core.hedgetech.ir/data-engine/tse-ifb/live/data/websocket/symbol/isin?channels=order-book&channels=best-limit&symbol_isins=IRT3SATF0001"
    header := map[string][]string{
        "Authorization": {"<your_token>"},
    }

    c, _, err := websocket.DefaultDialer.Dial(url, header)
    if err != nil {
        log.Fatal("dial:", err)
    }
    defer c.Close()

    for {
        _, message, err := c.ReadMessage()
        if err != nil {
            log.Println("read:", err)
            break
        }
        var m map[string]interface{}
        json.Unmarshal(message, &m)
        symbol := ""
        if v, ok := m["symbolIsin"]; ok {
            symbol = v.(string)
        } else if v, ok := m["symbolName"]; ok {
            symbol = v.(string)
        }
        fmt.Println(m["timestamp"], symbol, m["channel"])
    }
}
```

### 9.4 Julia (WebSocket Client)

```julia
using WebSockets, JSON

url = "wss://core.hedgetech.ir/data-engine/tse-ifb/live/data/websocket/symbol/isin?channels=order-book&channels=best-limit&symbol_isins=IRT3SATF0001"
token = "<your_token>"

WebSockets.open(url, extra_headers=["Authorization" => token]) do ws
    while !eof(ws)
        msg = String(readavailable(ws))
        data = JSON.parse(msg)
        symbol = get(data, "symbolIsin", get(data, "symbolName", ""))
        println(data["timestamp"], " ", symbol, " ", data["channel"])
    end
end
```

### 9.5 Rust (WebSocket Client)

```rust
use tokio_tungstenite::connect_async;
use futures_util::{StreamExt};
use url::Url;
use serde_json::Value;

#[tokio::main]
async fn main() {
    let url = Url::parse("wss://core.hedgetech.ir/data-engine/tse-ifb/live/data/websocket/symbol/isin?channels=order-book&channels=best-limit&symbol_isins=IRT3SATF0001").unwrap();
    let req = tokio_tungstenite::tungstenite::client::IntoClientRequest::into_client_request(url).unwrap();
    let (ws_stream, _) = connect_async(req).await.expect("Failed to connect");

    let (_write, mut read) = ws_stream.split();

    while let Some(message) = read.next().await {
        let msg = message.unwrap();
        if msg.is_text() {
            let v: Value = serde_json::from_str(msg.to_text().unwrap()).unwrap();
            let symbol = v.get("symbolIsin").or_else(|| v.get("symbolName"));
            println!("{:?} {:?} {:?}", v["timestamp"], symbol, v["channel"]);
        }
    }
}
```

---

### 9.6 Subscription Notes

- Multiple symbols and channels can be subscribed in a **single WebSocket connection**.
- The server streams messages continuously; handle them asynchronously.
- Always respect **Rate Limits** if applicable.

---

## 10. Best Practices

- Reconnect with **exponential backoff** in case of disconnects.
- Validate your JWT **before subscribing**.
- Subscribe only to the channels you need to reduce bandwidth.
- Handle `symbolIsin` / `symbolName` extraction consistently in your client code:
  - Prefer a helper function that returns a single canonical `symbol` value.
  - Log incoming messages and detect which endpoint you are connected to (optional, for debugging).

---

## Appendix: Quick developer checklist (for avoiding common mistakes)

- ✅ Use correct endpoint for your identifier type (`symbol/isin` vs `symbol/name`).
- ✅ Provide `Authorization` header with a valid token on the WebSocket handshake.
- ✅ Include `channels` and `symbol_isins` / `symbol_names` as repeated query params.
- ✅ Handle both `symbolIsin` and `symbolName` in your message parsing to make the client resilient.
- ✅ Treat messages' `data` payload consistently across endpoints (same schema).
- ✅ Monitor for WS close code `1008` to detect authorization or policy errors.

---
