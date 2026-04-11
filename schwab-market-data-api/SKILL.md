---
name: schwab-market-data-api
description: "Use this skill when building anything that interacts with the Charles Schwab Market Data API (part of Trader API - Individual). Covers REST endpoints for quotes, option chains, expiration chains, price history, movers, market hours, and instruments, plus the WebSocket Streamer API for real-time Level 1/2 data, charts, screeners, and account activity. Trigger on: Schwab quotes, Schwab option chains, Schwab price history, Schwab movers, Schwab market hours, Schwab instruments, Schwab streaming, Schwab WebSocket streamer, schwabapi.com/marketdata, LEVELONE_EQUITIES/OPTIONS/FUTURES/FOREX, CHART_EQUITY, SCREENER_EQUITY, NYSE_BOOK/NASDAQ_BOOK, ACCT_ACTIVITY streamer service. Do NOT use for: order placement, account balances, positions, transaction history, OAuth2 token refresh, or the finance PWA sync engine — use schwab-trading-api for those."
---

# Schwab Market Data API

Base URL: `https://api.schwabapi.com/marketdata/v1`

All endpoints require an OAuth 2.0 Bearer token in the `Authorization` header. Token management (access token ~30min, refresh token ~7 days) is handled in the `schwab-trading-api` skill.

---

## REST Endpoints

### 1. Quotes

**GET /quotes** — Batch quotes by symbol list

| Param | Type | Description |
|-------|------|-------------|
| `symbols` | string (query) | Comma-separated symbols. Supports equities, ETFs, mutual funds, indices (`$SPX`, `$DJI`), options, futures (`/ESH23`), futures options (`./ADUF23C0.55`), forex (`AUD/CAD`) |
| `fields` | string (query) | Subset: `quote`, `fundamental`, `extended`, `reference`, `regular`. Omit for all. |
| `indicative` | boolean (query) | Include indicative quotes for ETFs (e.g. `$ABC.IV`) |

**GET /{symbol_id}/quotes** — Single symbol quote

| Param | Type | Description |
|-------|------|-------------|
| `symbol_id` | string (path) | Single symbol (e.g. `TSLA`) |
| `fields` | string (query) | Same as batch |

**Response structure** (keyed by symbol):
```json
{
  "AAPL": {
    "assetMainType": "EQUITY",
    "symbol": "AAPL",
    "quoteType": "NBBO",
    "realtime": true,
    "ssid": 1973757747,
    "reference": { "cusip", "description", "exchange", "exchangeName" },
    "quote": { "lastPrice", "bidPrice", "askPrice", "openPrice", "highPrice", "lowPrice", "closePrice", "totalVolume", "netChange", "netPercentChange", "mark", "52WeekHigh", "52WeekLow", "volatility" },
    "regular": { "regularMarketLastPrice", "regularMarketNetChange", "regularMarketPercentChange" },
    "fundamental": { "peRatio", "eps", "divYield", "divAmount", "avg10DaysVolume" }
  }
}
```

- `assetMainType`: EQUITY, OPTION, INDEX, MUTUAL_FUND, FUTURE, FOREX, FUTURE_OPTION
- `quoteType`: NBBO = realtime, NFL = delayed

**Asset types:** EQUITY, ETF (subtype), MUTUAL_FUND, INDEX, OPTION, FUTURE, FUTURE_OPTION, FOREX, BOND

### 2. Option Chains

**GET /chains** — Full option chain for an optionable symbol

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `symbol` | string | Yes | Underlying symbol (e.g. `AAPL`) |
| `contractType` | string | No | `CALL`, `PUT`, `ALL` |
| `strikeCount` | int | No | Strikes above/below ATM |
| `strategy` | string | No | `SINGLE` (default), `ANALYTICAL`, `COVERED`, `VERTICAL`, `CALENDAR`, `STRANGLE`, `STRADDLE`, `BUTTERFLY`, `CONDOR`, `DIAGONAL`, `COLLAR`, `ROLL` |
| `strike` | double | No | Specific strike price |
| `range` | string | No | ITM/NTM/OTM filter |
| `fromDate` / `toDate` | date | No | `yyyy-MM-dd` format |
| `volatility` | double | No | For ANALYTICAL strategy only |
| `underlyingPrice` | double | No | For ANALYTICAL strategy only |
| `interestRate` | double | No | For ANALYTICAL strategy only |
| `daysToExpiration` | int | No | For ANALYTICAL strategy only |
| `expMonth` | string | No | `JAN`–`DEC` or `ALL` |
| `includeUnderlyingQuote` | boolean | No | Include underlying quote |

**Response:** Contains `callExpDateMap` and `putExpDateMap`, each keyed by expiration date, then by strike price. Each option contract includes: `putCall`, `symbol`, `bidPrice`, `askPrice`, `lastPrice`, `markPrice`, `totalVolume`, `openInterest`, `volatility`, `delta`, `gamma`, `theta`, `vega`, `rho`, `strikePrice`, `expirationDate`, `daysToExpiration`, `isInTheMoney`, `theoreticalOptionValue`, etc.

### 3. Option Expiration Chain

**GET /expirationchain** — Expiration dates only (no contracts)

| Param | Type | Required |
|-------|------|----------|
| `symbol` | string | Yes |

Returns: `{ "expirationList": [{ "expirationDate": "2022-01-07", "daysToExpiration": 2, "expirationType": "W", "standard": true }, ...] }`

### 4. Price History

**GET /pricehistory** — OHLCV candles

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `symbol` | string | Yes | Equity symbol |
| `periodType` | string | No | `day`, `month`, `year`, `ytd` |
| `period` | int | No | Number of periods (varies by type) |
| `frequencyType` | string | No | `minute`, `daily`, `weekly`, `monthly` |
| `frequency` | int | No | Aggregation interval (1, 5, 10, 15, 30 for minute) |
| `startDate` / `endDate` | long | No | Epoch milliseconds |
| `needExtendedHoursData` | boolean | No | Include extended hours |
| `needPreviousClose` | boolean | No | Include prev close |

**Period/Frequency rules:**
- `day` → `minute` (1,5,10,15,30 min), default period=10
- `month` → `daily` or `weekly`, default period=1
- `year` → `daily`, `weekly`, or `monthly`, default period=1
- `ytd` → `daily` or `weekly`, default period=1

**Response:**
```json
{
  "symbol": "AAPL",
  "candles": [{ "open", "high", "low", "close", "volume", "datetime" }],
  "previousClose": 174.56,
  "previousCloseDate": 1639029600000
}
```

### 5. Movers

**GET /movers/{symbol_id}** — Top 10 movers for an index

| Param | Type | Description |
|-------|------|-------------|
| `symbol_id` | string (path) | `$DJI`, `$COMPX`, `$SPX`, `NYSE`, `NASDAQ`, `OTCBB`, `INDEX_ALL`, `EQUITY_ALL`, `OPTION_ALL`, `OPTION_PUT`, `OPTION_CALL` |
| `sort` | string | `VOLUME`, `TRADES`, `PERCENT_CHANGE_UP`, `PERCENT_CHANGE_DOWN` |
| `frequency` | int | `0` (all day), `1`, `5`, `10`, `30`, `60` minutes |

### 6. Market Hours

**GET /markets** — Multiple markets: `equity`, `option`, `bond`, `future`, `forex`

**GET /markets/{market_id}** — Single market

| Param | Type |
|-------|------|
| `date` | date (query, `YYYY-MM-DD`, optional, defaults to today) |

Returns `isOpen`, `sessionHours` with `preMarket`, `regularMarket`, `postMarket` start/end times.

### 7. Instruments

**GET /instruments** — Search by symbol or description

| Param | Type | Description |
|-------|------|-------------|
| `symbol` | string | Symbol(s) to search |
| `projection` | string | `symbol-search`, `symbol-regex`, `desc-search`, `desc-regex`, `search`, `fundamental` |

**GET /instruments/{cusip_id}** — Lookup by CUSIP

---

## WebSocket Streamer API

Real-time streaming via WebSocket with JSON messages. Connection URL and credentials come from the **GET User Preference** endpoint (in the trading API).

### Connection Flow

1. Connect to WebSocket URL from User Preference
2. Send LOGIN command with access token
3. Wait for successful response (code 0)
4. Subscribe to services
5. Receive data streams
6. LOGOUT when done

### Request Format

```json
{
  "requests": [
    {
      "requestid": "1",
      "service": "ADMIN",
      "command": "LOGIN",
      "SchwabClientCustomerId": "<from User Preference>",
      "SchwabClientCorrelId": "<from User Preference>",
      "parameters": {
        "Authorization": "<access_token>",
        "SchwabClientChannel": "<from User Preference>",
        "SchwabClientFunctionId": "<from User Preference>"
      }
    }
  ]
}
```

### Commands

| Command | Description |
|---------|-------------|
| `LOGIN` | Must succeed before other commands |
| `SUBS` | Subscribe (replaces previous subs for that service) |
| `ADD` | Add symbols without clearing existing subs |
| `UNSUBS` | Unsubscribe symbols |
| `VIEW` | Change field subscription for a service |
| `LOGOUT` | Close connection |

### Response Types

- **response** — Replies to commands (code 0 = success)
- **notify** — Heartbeats: `{"notify":[{"heartbeat":"1668715930582"}]}`
- **data** — Market data updates with numeric field keys

### Available Services

| Service | Description | Delivery |
|---------|-------------|----------|
| `LEVELONE_EQUITIES` | L1 equity quotes | Change-based |
| `LEVELONE_OPTIONS` | L1 option quotes | Change-based |
| `LEVELONE_FUTURES` | L1 futures quotes | Change-based |
| `LEVELONE_FUTURES_OPTIONS` | L1 futures options | Change-based |
| `LEVELONE_FOREX` | L1 forex quotes | Change-based |
| `NYSE_BOOK` | L2 NYSE book | Whole |
| `NASDAQ_BOOK` | L2 NASDAQ book | Whole |
| `OPTIONS_BOOK` | L2 options book | Whole |
| `CHART_EQUITY` | 1-min candles (equity) | All Sequence |
| `CHART_FUTURES` | 1-min candles (futures) | All Sequence |
| `SCREENER_EQUITY` | Equity advances/decliners | Whole |
| `SCREENER_OPTION` | Option advances/decliners | Whole |
| `ACCT_ACTIVITY` | Order fills, account events | All Sequence |

### Key Field Definitions

#### LEVELONE_EQUITIES (subscribe with numeric field IDs)

| Field | Name | Type |
|-------|------|------|
| 0 | Symbol | String |
| 1 | Bid Price | double |
| 2 | Ask Price | double |
| 3 | Last Price | double |
| 4 | Bid Size | int (lots) |
| 5 | Ask Size | int (lots) |
| 8 | Total Volume | long |
| 9 | Last Size | long (shares) |
| 10 | High Price | double |
| 11 | Low Price | double |
| 12 | Close Price | double |
| 15 | Description | String |
| 17 | Open Price | double |
| 18 | Net Change | double |
| 19 | 52 Week High | double |
| 20 | 52 Week Low | double |
| 21 | PE Ratio | double |
| 28 | Delta/Gamma/etc | (options only) |
| 29 | Regular Market Last Price | double |
| 33 | Mark Price | double |
| 34 | Quote Time (ms epoch) | long |
| 35 | Trade Time (ms epoch) | long |
| 42 | Net Percent Change | double |
| 50 | Post-Market Net Change | double |
| 51 | Post-Market Percent Change | double |

Also returned per symbol: `key` (symbol), `delayed` (boolean), `assetMainType`, `assetSubType`, `cusip`.

#### LEVELONE_OPTIONS (key format: `AAPL  251219C00200000`)

Key fields: 2=Bid, 3=Ask, 4=Last, 7=Close, 8=Volume, 9=Open Interest, 10=Volatility, 11=Intrinsic Value, 19=Net Change, 20=Strike, 21=Contract Type, 22=Underlying, 27=Days to Expiration, 28=Delta, 29=Gamma, 30=Theta, 31=Vega, 35=Underlying Price, 37=Mark

#### LEVELONE_FUTURES (key format: `/ESZ24`)

Key fields: 1=Bid, 2=Ask, 3=Last, 8=Volume, 10=Quote Time, 11=Trade Time, 12=High, 13=Low, 14=Close, 18=Open, 19=Net Change, 20=Percent Change, 23=Open Interest, 24=Mark

Month codes: F=Jan, G=Feb, H=Mar, J=Apr, K=May, M=Jun, N=Jul, Q=Aug, U=Sep, V=Oct, X=Nov, Z=Dec

#### CHART_EQUITY / CHART_FUTURES

Equity fields: 1=Open, 2=High, 3=Low, 4=Close, 5=Volume, 6=Sequence, 7=Chart Time
Futures fields: 1=Chart Time, 2=Open, 3=High, 4=Low, 5=Close, 6=Volume

#### SCREENER_EQUITY / SCREENER_OPTION

Key format: `{PREFIX}_{SORTFIELD}_{FREQUENCY}`
- Prefix: `$COMPX`, `$DJI`, `$SPX`, `INDEX_ALL`, `NYSE`, `NASDAQ`, `OTCBB`, `EQUITY_ALL`, `OPTION_PUT`, `OPTION_CALL`, `OPTION_ALL`
- Sort: `VOLUME`, `TRADES`, `PERCENT_CHANGE_UP`, `PERCENT_CHANGE_DOWN`, `AVERAGE_PERCENT_VOLUME`
- Frequency: `0` (all day), `1`, `5`, `10`, `30`, `60`

#### ACCT_ACTIVITY

Fields: 1=Account Number, 2=Message Type, 3=Message Data (JSON)

### Response Codes

| Code | Meaning | Connection Severed? |
|------|---------|-------------------|
| 0 | Success | No |
| 3 | Login Denied | Yes — reconnect with new token |
| 12 | Max connections (limit: 1) | Yes |
| 19 | Symbol limit reached | No |
| 21 | Bad command format | No |
| 30 | Stop streaming (inactivity) | Yes |

### Delivery Types

- **Change** — Only changed fields sent (L1 services). Conflated by streamer.
- **Whole** — Full data unit, throttled (Book, Screener).
- **All Sequence** — Every update with sequence number, no conflation (Chart, Account).

---

## Common Error Responses

All REST endpoints return standard errors:
- **400** — Bad request (missing header, invalid params)
- **401** — Unauthorized (token expired/invalid)
- **404** — Not found
- **500** — Internal server error

All include `Schwab-Client-CorrelId` header for support.

---

## Quick Reference: Symbol Formats

| Type | Format | Example |
|------|--------|---------|
| Equity | `SYMBOL` | `AAPL` |
| Index | `$SYMBOL` | `$SPX`, `$DJI` |
| Option | `RRRRRR YYMMDDsWWWWWddd` | `AAPL  251219C00200000` |
| Future | `/ROOT_MONTH_YEAR` | `/ESZ24` |
| Future Option | `./ROOT_MONTH_YEAR_C/P_STRIKE` | `./OZCZ23C565` |
| Forex | `CUR1/CUR2` | `EUR/USD` |
