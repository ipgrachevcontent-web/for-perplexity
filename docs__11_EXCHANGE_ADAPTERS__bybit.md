# 11.2 · Bybit Connector

> USDT Perpetual (Linear), **native interval** primary source.
> Base: `_BASE_CONNECTOR.md`.

---

## 1. Source kind

- **Primary:** `native_interval` — Bybit предоставляет ready-made 5m/15m/30m бары через `/v5/market/open-interest`.
- **Fallback:** `degraded_fallback` к snapshot polling (`/v5/market/tickers` имеет `openInterest` field) при деградации native API.

---

## 2. URLs

- **Base:** `https://api.bybit.com`
- **Доки:** https://bybit-exchange.github.io/docs/v5/intro

---

## 3. Endpoints

### 3.1 Список инструментов

`GET /v5/market/instruments-info?category=linear`

- Фильтр: `category=linear` для USDT-M.
- Cadence: каждые 5 минут.

### 3.2 Open Interest history (primary)

`GET /v5/market/open-interest?category=linear&symbol=BTCUSDT&intervalTime=5min&limit=200`

- intervalTime: `5min`, `15min`, `30min`, `1h`, `4h`, `1d`.
- Limit: max 200.
- Поддерживает `cursor` для пагинации.
- Response:
```json
{
  "retCode": 0,
  "result": {
    "category": "linear",
    "symbol": "BTCUSDT",
    "list": [
      {
        "openInterest": "12345.678",
        "timestamp": "1714478700000"
      },
      ...
    ],
    "nextPageCursor": "..."
  }
}
```

### 3.3 Tickers (snapshot + price)

`GET /v5/market/tickers?category=linear`

- Возвращает все linear contracts: `lastPrice`, `markPrice`, `indexPrice`, `openInterest`, `openInterestValue`.
- Используется для fallback OI и для получения цен.

---

## 4. Rate limits

- 600 req/5sec для public market endpoints (per IP).
- HTTP 403 + `retCode = 10006` при превышении.

**Конфиг:**
```python
RATE_LIMIT_CAPACITY = 600
RATE_LIMIT_REFILL_PER_SEC = 120
MAX_CONCURRENT_REQUESTS = 20
```

---

## 5. Symbol naming

- Pattern: `BASEUSDT`.
- Examples: `BTCUSDT`, `ETHUSDT`, `1000SHIBUSDT`, `1000PEPEUSDT`.
- Аналогично Binance, prefix `1000` для мелких токенов → `multiplier = 1000`.

---

## 6. Filter (USDT-M Perpetual / Linear)

```python
def is_usdt_m_perpetual(item: dict) -> bool:
    return (
        item["category"] == "linear"           # вычисляется на уровне запроса
        and item["settleCoin"] == "USDT"
        and item["contractType"] == "LinearPerpetual"
        and item["status"] == "Trading"
    )
```

Поля:
- `contractType`: `"LinearPerpetual"` | `"InversePerpetual"` | `"LinearFutures"` | `"InverseFutures"`.
- `settleCoin`: `"USDT"` для USDT-M.
- `quoteCoin`: `"USDT"`.
- `status`: `"Trading"` | `"PreLaunch"` | `"Closed"`.

---

## 7. Parser

### 7.1 От `/v5/market/open-interest` (native_interval, primary)

```python
def parse_open_interest(item: dict, symbol: str) -> ParsedOI:
    # ts_exchange = bucket_end (Bybit returns end-of-interval timestamp)
    return ParsedOI(
        native_symbol=symbol,
        ts_exchange=datetime.fromtimestamp(int(item["timestamp"]) / 1000, tz=UTC),
        oi_raw=Decimal(item["openInterest"]),
        oi_unit_hint="base_asset",
        oi_notional_provided=None,
        price_provided=None,
        raw_extras={},
    )
```

### 7.2 От `/v5/market/tickers` (snapshot fallback)

```python
def parse_ticker_oi(item: dict) -> ParsedOI:
    return ParsedOI(
        native_symbol=item["symbol"],
        ts_exchange=utcnow(),     # tickers не содержат ts по OI; используем серверное
        oi_raw=Decimal(item["openInterest"]),
        oi_unit_hint="base_asset",
        oi_notional_provided=Decimal(item["openInterestValue"]),  # уже в USDT
        price_provided=Decimal(item["markPrice"]),
        raw_extras={"last_price": item["lastPrice"], "index_price": item["indexPrice"]},
    )
```

**ВАЖНО:** Bybit прямо в документации указывает: для `linear` (BTCUSDT) поле `openInterest` — в **base asset (BTC)**, а не в USDT. Поле `openInterestValue` в tickers — уже в USDT (authoritative notional).

---

## 8. Unit resolution

- `oi_unit_hint = "base_asset"` (BTC для BTCUSDT).
- `oi_coins = oi_raw` напрямую.
- `oi_notional_usdt`:
  - Если `oi_notional_provided` (`openInterestValue`) is set (tickers) → `valuation_status = "authoritative"`.
  - Иначе (open-interest endpoint) → `oi_coins × mark_price`, `valuation_status = "good_estimate"`.

---

## 9. Price resolution

- Primary: `markPrice` из `/v5/market/tickers`.
- Fallback: `indexPrice` оттуда же.
- Last fallback: `lastPrice`.

Bulk: `/v5/market/tickers?category=linear` возвращает все символы за раз.

---

## 10. Watermark logic

Native_interval требует watermark для инкрементального fetch:

```python
async def fetch_range(self, instrument, interval, start, end):
    last_ts = await self.watermark_repo.get_watermark(self.exchange, instrument.native_symbol, interval)
    actual_start = last_ts or start

    cursor = None
    all_items = []
    while True:
        params = {
            "category": "linear",
            "symbol": instrument.native_symbol,
            "intervalTime": _interval_to_param(interval),
            "startTime": int(actual_start.timestamp() * 1000),
            "endTime": int(end.timestamp() * 1000),
            "limit": 200,
        }
        if cursor:
            params["cursor"] = cursor

        response = await self._http.get("/v5/market/open-interest", params=params)
        data = response.json()
        all_items.extend(data["result"]["list"])
        cursor = data["result"].get("nextPageCursor")
        if not cursor:
            break

    if all_items:
        max_ts = max(int(i["timestamp"]) for i in all_items)
        await self.watermark_repo.set_watermark(
            self.exchange, instrument.native_symbol, interval,
            datetime.fromtimestamp(max_ts / 1000, tz=UTC),
        )

    return [self._make_raw_event(item, ...) for item in all_items]
```

---

## 11. Edge cases

### 11.1 Volatility delays

Bybit прямо предупреждает в docs: при экстремальной волатильности возможны задержки доставки данных в open-interest endpoint. Mitigation:
- При обнаружении `freshness_age > freshness_budget_native_interval` (default `5min + 60s = 360s`) — алерт engine помечает точку как stale.
- При повторных деградациях (3 cycle подряд stale) → `degraded_fallback` mode.

### 11.2 Retention

Bybit не документирует явный retention для open-interest endpoint, но эмпирически 7+ дней доступны. Это покрывает наши потребности.

### 11.3 Pagination edge

`nextPageCursor` может быть пустой строкой `""` или null — оба означают «больше нет страниц».

### 11.4 PreLaunch status

При `status == "PreLaunch"` контракт ещё не торгуется. Не добавляем в активные.

---

## 12. Test fixtures

```
tests/contract/bybit/fixtures/
├── instruments_info_linear.json
├── open_interest_5min_btcusdt.json
├── open_interest_15min_btcusdt.json
├── open_interest_with_pagination.json
└── tickers_linear.json
```

---

## 13. References

- Bybit V5 API: https://bybit-exchange.github.io/docs/v5/market/open-interest
- Tickers: https://bybit-exchange.github.io/docs/v5/market/tickers
- Instruments info: https://bybit-exchange.github.io/docs/v5/market/instrument
