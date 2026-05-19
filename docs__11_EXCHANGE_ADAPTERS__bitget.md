# 11.4 · Bitget Connector

> USDT-M Perpetual (umcbl), snapshot polling.
> Base: `_BASE_CONNECTOR.md`.

---

## 1. Source kind

- **Primary:** `snapshot` через `/api/v2/mix/market/open-interest`.

---

## 2. URLs

- **Base:** `https://api.bitget.com`
- **Доки:** https://www.bitget.com/api-doc/contract/market/Get-Open-Interest

---

## 3. Endpoints

### 3.1 Список инструментов

`GET /api/v2/mix/market/contracts?productType=USDT-FUTURES`

- `productType`: `"USDT-FUTURES"` для USDT-M perp.
- Cadence: каждые 5 минут.

### 3.2 Open Interest (per symbol or bulk)

`GET /api/v2/mix/market/open-interest?productType=USDT-FUTURES&symbol=BTCUSDT`

- Без `symbol` возвращает все.
- Response:
```json
{
  "code": "00000",
  "data": {
    "symbol": "BTCUSDT",
    "amount": "12345.678",
    "timestamp": "1714478700000"
  }
}
```

`amount` — в базовом активе (для USDT-M linear).

### 3.3 Tickers (bulk OI + price)

`GET /api/v2/mix/market/tickers?productType=USDT-FUTURES`

Возвращает все символы с полями: `lastPr`, `markPrice`, `holdingAmount` (= OI в base), `quoteVolume`, `baseVolume`. Удобно для bulk fetch.

### 3.4 Mark price

`GET /api/v2/mix/market/symbol-price?productType=USDT-FUTURES&symbol=BTCUSDT`

Или из tickers (см. выше).

---

## 4. Rate limits

- 20 req / sec для public market endpoints.
- HTTP 429 при превышении.

**Конфиг:**
```python
RATE_LIMIT_CAPACITY = 20
RATE_LIMIT_REFILL_PER_SEC = 20
MAX_CONCURRENT_REQUESTS = 10
```

**Стратегия:** использовать bulk `/tickers` endpoint для большинства данных за 1 запрос/цикл.

---

## 5. Symbol naming

- Pattern: `BASEUSDT` (без разделителей).
- Examples: `BTCUSDT`, `ETHUSDT`.
- В V2 API убран суффикс `_UMCBL` (был в V1).

---

## 6. Filter (USDT-M Perpetual)

```python
def is_usdt_m_perpetual(item: dict) -> bool:
    return (
        item["productType"] == "USDT-FUTURES"  # вычисляется на уровне query
        and item["symbolType"] == "perpetual"
        and item["symbolStatus"] == "normal"
        and item["quoteCoin"] == "USDT"
    )
```

Поля:
- `productType`: `"USDT-FUTURES"` | `"COIN-FUTURES"` | `"USDC-FUTURES"`.
- `symbolType`: `"perpetual"` | `"delivery"`.
- `symbolStatus`: `"normal"` | `"limit_open"` | `"restricted"` | `"off"`.
- `quoteCoin`: `"USDT"`.
- `baseCoin`, `quoteCoin`, `supportMarginCoins`.

---

## 7. Parser

### 7.1 От `/api/v2/mix/market/open-interest`

```python
def parse_open_interest(payload: dict) -> ParsedOI:
    data = payload["data"]
    return ParsedOI(
        native_symbol=data["symbol"],
        ts_exchange=datetime.fromtimestamp(int(data["timestamp"]) / 1000, tz=UTC),
        oi_raw=Decimal(data["amount"]),
        oi_unit_hint="base_asset",
        oi_notional_provided=None,
        price_provided=None,
        raw_extras={},
    )
```

### 7.2 От `/api/v2/mix/market/tickers`

```python
def parse_ticker_oi(item: dict) -> ParsedOI:
    return ParsedOI(
        native_symbol=item["symbol"],
        ts_exchange=datetime.fromtimestamp(int(item["ts"]) / 1000, tz=UTC),
        oi_raw=Decimal(item["holdingAmount"]),
        oi_unit_hint="base_asset",
        oi_notional_provided=None,
        price_provided=Decimal(item["markPrice"]),
        raw_extras={"last_price": item["lastPr"]},
    )
```

---

## 8. Unit resolution

- `oi_unit_hint = "base_asset"`.
- `oi_coins = oi_raw`.
- `oi_notional_usdt = oi_coins × mark_price`, `valuation_status = "good_estimate"`.

---

## 9. Price resolution

- Primary: `markPrice` из tickers или mark-price endpoint.
- Fallback: `lastPr` (last price).

---

## 10. Edge cases

### 10.1 V1 vs V2 API

Bitget V1 имела `BTCUSDT_UMCBL` naming. В V2 убран суффикс. Используем V2.

### 10.2 Coin-margined contracts

`COIN-FUTURES` (например `BTCUSD_DMCBL` в V1) — фильтруются `productType != "USDT-FUTURES"`.

### 10.3 USDC-futures

С 2023 Bitget добавил USDC-M (`productType == "USDC-FUTURES"`). Не относится к нашему scope.

### 10.4 Restricted symbols

`symbolStatus == "restricted"` — торговля частично ограничена. Опрашиваем (если торгуется), но в `Instrument.warnings` добавляем `restricted_status`.

---

## 11. Test fixtures

```
tests/contract/bitget/fixtures/
├── contracts_usdt_futures.json
├── open_interest_btcusdt.json
├── tickers_usdt_futures.json
└── mark_price_btcusdt.json
```

---

## 12. References

- Bitget Mix API: https://www.bitget.com/api-doc/contract/intro
- Open Interest: https://www.bitget.com/api-doc/contract/market/Get-Open-Interest
- Tickers: https://www.bitget.com/api-doc/contract/market/Get-All-Symbol-Ticker
