# HT Data Engine | TSE & IFB | websocket
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
| `/live/data/websocket/symbol/isin` | Subscribe using **ISIN** codes |
| `/live/data/websocket/symbol/name` | Subscribe using **symbol names** |

**Note:** The output payload structure is identical for both endpoints, except the symbol identifier field:
- `symbolIsin` for `/symbol/isin`
- `symbolName` for `/symbol/name`

---

## 4. Connection Flow

1. Establish WebSocket connection with proper `Authorization` header.
2. Send query parameters for symbols/ISINs and channels:

```
?symbol_isins=<comma-separated-list>&channels=<comma-separated-list>
```
or
```
?symbol_names=<comma-separated-list>&channels=<comma-separated-list>
```

3. Server verifies permission via `SecuritiesExchangeLiveData_WS_Access_Permission`.
4. If verification passes, the WebSocket is accepted.
5. The server subscribes to requested Redis streams internally.
6. Real-time messages are sent continuously until the connection is closed.

**Unauthorized connections** are closed immediately with code `1008`.

---

## 5. Query Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `symbol_isins` | list of strings | List of ISIN codes to subscribe (for `/symbol/isin`) |
| `symbol_names` | list of strings | List of symbol names to subscribe (for `/symbol/name`) |
| `channels` | list of strings | List of channels to subscribe (see Section 6) |

**Example URL:**

```
wss://core.hedgetech.ir/live/data/websocket/symbol/isin?symbol_isins=IRO1XYZ1234,IRO2ABC567&channels=best-limit,order-book
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

### 6.2 Example: `ohlcv-last-1m`

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

### 6.3 Other Channels

Payload models follow the **Pydantic models** provided (`Aggregate`, `OrderBook`, `InstitutionalVsIndividual`, `ContractInfo`, `FundInfo`) and always adhere to the format:

```json
{
  "channel": "<channel-name>",
  "symbolIsin": "<symbolIsin>",
  "timestamp": "<ISO8601 timestamp>",
  "data": { ...channel-specific data... }
}
```

---

## 7. Error Handling

| Code | Description |
|------|-------------|
| `1008` | Policy violation (invalid JWT, invalid symbols, or invalid channels) |
| Connection closed | Occurs if Redis stream fails or server error |

---

## 8. Examples

### 8.1 Python (WebSocket Client)

```python
import asyncio
import websockets
import json

async def subscribe():
    url = "wss://core.hedgetech.ir/live/data/websocket/symbol/isin?symbol_isins=IRO1XYZ1234&channels=best-limit,order-book"
    headers = {
        "Authorization": "<your_token>"
    }
    async with websockets.connect(url, extra_headers=headers) as ws:
        async for message in ws:
            data = json.loads(message)
            print(data)

asyncio.run(subscribe())
```

### 8.2 Subscription Notes

- Multiple symbols and channels can be subscribed in a **single WebSocket connection**.
- The server streams messages continuously; handle them asynchronously.
- Always respect **Rate Limits** if applicable.

---

## 9. Best Practices

- Reconnect with **exponential backoff** in case of disconnects.
- Validate your JWT **before subscribing**.
- Subscribe only to the channels you need to reduce bandwidth.
- Handle `symbolIsin` extraction consistently in your client code.

---
