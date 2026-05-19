# 11.1 · Binance Connector

> USDT-M Perpetual (USDⓈ-M Futures), snapshot per-symbol polling.
> Base: `_BASE_CONNECTOR.md`.

---

## 1. Source kind

- **Primary:** `snapshot` — `/fapi/v1/openInterest` отдаёт текущее OI.
- **Опционально:** `/futures/data/openInterestHist` — historical статистика 5m/15m/30m с `sumOpenInterestValue` (authoritative notional). Используется как **дополнительный** источник для сверки/backfill, не primary.

---

## 2. URLs

- **Base:** `https://fapi.binance.com`
- **Доки:** https://binance-docs.github.io/apidocs/futures/en/

---

## 3. Endpoints

### 3.1 Список инструментов

`GET /fapi/v1/exchangeInfo`

- Weight: 1.
- Cadence: каждые 5 минут (instrument sync).

### 3.2 Open Interest (snapshot)

`GET /fapi/v1/openInterest?symbol=BTCUSDT`

- Weight: 1.
- Возвращает текущее OI для одного символа.
- Response:
```json
{
  "openInterest": "10659.509",
  "symbol": "BTCUSDT",
  "time": 1589437530011
}
```

### 3.3 Open Interest history (опциональный, для backfill)

`GET /futures/data/openInterestHist?symbol=BTCUSDT&period=5m&limit=30`

- Weight: 1 per request.
- Periods: `5m`, `15m`, `30m`, `1h`, `2h`, `4h`, `6h`, `12h`, `1d`.
- Limit: max 500.
- Доступная история: ~30 дней.
- Response:
```json
[
  {
    "symbol": "BTCUSDT",
    "sumOpenInterest": "20403.63700000",
    "sumOpenInterestValue": "150570784.07809979",
    "timestamp": 1583128500000
  },
  ...
]
```

`sumOpenInterestValue` — authoritative notional USDT, используется как высококачественный источник для replay/backfill.

### 3.4 Mark price

`GET /fapi/v1/premiumIndex?symbol=BTCUSDT`

- Weight: 1 (или 10 если без symbol).
- Bulk запрос без symbol даёт все символы за один вызов.
- Response:
```json
{
  "symbol": "BTCUSDT",
  "markPrice": "67420.50",
  "indexPrice": "67400.10",
  "lastFundingRate": "0.0001",
  "time": 1589437530011
}
```

---

## 4. Rate limits

- 2400 weight/min для futures public endpoints (на IP).
- При нарушении — HTTP 429 + `Retry-After` header.
- При повторных нарушениях — temporary IP ban (2 мин).

**Конфиг:**
```python
RATE_LIMIT_CAPACITY = 2400
RATE_LIMIT_REFILL_PER_SEC = 40   # 2400 / 60
MAX_CONCURRENT_REQUESTS = 20
```

При 300 символов × weight=1 = 300 weight за цикл — укладываемся легко.

---

## 5. Symbol naming

- Pattern: `BASEUSDT` (без разделителей).
- Examples: `BTCUSDT`, `ETHUSDT`, `1000PEPEUSDT`, `XRPUSDT`.
- Special prefix `1000`: для токенов с очень малой ценой (`SHIB1000USDT`, `1000PEPEUSDT`). На canonical mapping не влияет (`SHIB`, `PEPE`).

---

## 6. Filter (USDT-M Perpetual)

```python
def is_usdt_m_perpetual(symbol_info: dict) -> bool:
    return (
        symbol_info["contractType"] == "PERPETUAL"
        and symbol_info["quoteAsset"] == "USDT"
        and symbol_info["status"] == "TRADING"
    )
```

Поля:
- `contractType`: `"PERPETUAL"` | `"CURRENT_QUARTER"` | `"NEXT_QUARTER"` | `"CURRENT_MONTH"`.
- `quoteAsset`: для нашего scope `"USDT"`.
- `status`: `"TRADING"` | `"PENDING_TRADING"` | `"PAUSED"` | `"BREAK"` | `"SETTLING"` | `"CLOSE"`.
- `marginAsset`: `"USDT"` для USDT-M (доп. подтверждение).

---

## 7. Parser

### 7.1 От `/fapi/v1/openInterest` (snapshot)

```python
def parse_open_interest(payload: dict) -> ParsedOI:
    return ParsedOI(
        native_symbol=payload["symbol"],
        ts_exchange=datetime.fromtimestamp(payload["time"] / 1000, tz=UTC),
        oi_raw=Decimal(payload["openInterest"]),
        oi_unit_hint="base_asset",
        oi_notional_provided=None,
        price_provided=None,
        raw_extras={},
    )
```

### 7.2 От `/futures/data/openInterestHist` (history)

```python
def parse_open_interest_hist(item: dict) -> ParsedOI:
    return ParsedOI(
        native_symbol=item["symbol"],
        ts_exchange=datetime.fromtimestamp(item["timestamp"] / 1000, tz=UTC),
        oi_raw=Decimal(item["sumOpenInterest"]),
        oi_unit_hint="base_asset",
        oi_notional_provided=Decimal(item["sumOpenInterestValue"]),
        price_provided=None,
        raw_extras={},
    )
```

`sumOpenInterestValue` → `valuation_status = "authoritative"`.

---

## 8. Unit resolution

- `oi_unit_hint = "base_asset"` (Binance возвращает OI в монетах).
- `oi_coins = oi_raw` напрямую.
- `oi_notional_usdt`:
  - Если `oi_notional_provided` is set (history endpoint) → используем как authoritative.
  - Иначе → `oi_coins × mark_price`, `valuation_status = "good_estimate"`.

---

## 9. Price resolution

- Primary: `markPrice` из `/fapi/v1/premiumIndex`.
- Fallback: `indexPrice` из того же endpoint'а.
- Last fallback: ticker last (`/fapi/v1/ticker/price`) — but rarely needed.

Bulk-запрос `/fapi/v1/premiumIndex` (без symbol) возвращает все символы за один вызов — рекомендуется использовать его раз за цикл.

---

## 10. Edge cases

### 10.1 Listing in `PENDING_TRADING`

Символ есть в `exchangeInfo`, но `status != "TRADING"` → не добавляем в активные инструменты.

### 10.2 Settlement / Close

Контракт закрывается с делистингом (например, мейнет-сворачивание). При `status == "CLOSE"` или отсутствии в `exchangeInfo` → `delisted_at = now()`.

### 10.3 1000-prefix tokens

`1000SHIBUSDT`, `1000PEPEUSDT`. Canonical:
- `1000SHIB` → `SHIB` (по соглашению, делим на 1000 при отображении OI).
- В `Instrument` храним `multiplier = 1000`, чтобы получить «настоящие монеты».

NB: текущая логика — `oi_coins = oi_raw × multiplier`, чтобы видеть OI в реальных SHIB, не в "1000-units".

### 10.4 BUSD / FDUSD / USDC pairs

Раньше Binance имел `BTCBUSD` контракты. Они **не USDT-M** (quoteAsset = "BUSD"). Должны фильтроваться `quoteAsset == "USDT"`.

---

## 11. Test fixtures

```
tests/contract/binance/fixtures/
├── exchange_info.json         # /fapi/v1/exchangeInfo
├── open_interest_btcusdt.json # /fapi/v1/openInterest
├── open_interest_hist_btcusdt.json
├── premium_index_all.json     # /fapi/v1/premiumIndex без symbol
└── premium_index_btcusdt.json # /fapi/v1/premiumIndex с symbol
```

---

## 12. References

- Binance Futures docs: https://binance-docs.github.io/apidocs/futures/en/
- Open Interest: https://binance-docs.github.io/apidocs/futures/en/#open-interest
- Open Interest Statistics: https://binance-docs.github.io/apidocs/futures/en/#open-interest-statistics
- Mark Price: https://binance-docs.github.io/apidocs/futures/en/#mark-price
