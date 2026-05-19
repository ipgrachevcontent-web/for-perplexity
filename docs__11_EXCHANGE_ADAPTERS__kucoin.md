# 11.7 · KuCoin Connector

> USDT-M Perpetual (FFWCSX), **native interval** primary source.
> Base: `_BASE_CONNECTOR.md`.

---

## 1. Source kind

- **Primary:** `native_interval` через `/api/ua/v1/market/open-interest`.
- **Fallback:** `degraded_fallback` через snapshot из `/api/v1/contracts/active` (поле `openInterest`).

---

## 2. URLs

- **Base:** `https://api-futures.kucoin.com`
- **Доки:** https://www.kucoin.com/docs/rest/futures-trading/market-data/

---

## 3. Endpoints

### 3.1 Список инструментов

`GET /api/v1/contracts/active`

- Возвращает все active futures contracts.
- Для USDT-M фильтр `quoteCurrency == "USDT"` AND `type == "FFWCSX"`.
- Cadence: каждые 5 минут.

### 3.2 Open Interest history (primary)

`GET /api/ua/v1/market/open-interest?symbol=BTCUSDTM&granularity=Min5&from=...&to=...`

- granularity: `Min5`, `Min15`, `Min30`, `Hour1`, `Hour4`, `Day1`.
- Retention: KuCoin документирует **7 дней** для `Min5`/`Min15`/`Min30`/`Hour1`/`Hour4`, **70 дней** для `Day1`.
- Response:
```json
{
  "code": "200000",
  "data": {
    "values": [
      [1714478700000, "12345.678"],
      [1714478400000, "12340.123"],
      ...
    ]
  }
}
```
Каждый элемент: `[timestamp, openInterest]`. `openInterest` в base asset.

### 3.3 Snapshot OI + price (fallback / supplementary)

`GET /api/v1/contracts/{symbol}` или `/api/v1/contracts/active` (bulk).

- Bulk `/active` возвращает все contracts с полями: `markPrice`, `indexPrice`, `lastTradePrice`, `openInterest`.
- Используется для fallback и для price source.

### 3.4 Mark price

`GET /api/v1/mark-price/{symbol}/current`

- Или из bulk `/contracts/active` (поле `markPrice`).

---

## 4. Rate limits

- Public market: 30 req / 3 sec.
- HTTP 429 при превышении.

**Конфиг:**
```python
RATE_LIMIT_CAPACITY = 30
RATE_LIMIT_REFILL_PER_SEC = 10
MAX_CONCURRENT_REQUESTS = 10
```

---

## 5. Symbol naming

- Pattern: `BASEUSDTM` (USDT + M суффикс для USDT-margined).
- Examples: `BTCUSDTM`, `ETHUSDTM`, `XBTUSDTM` (XBT — alias для BTC, legacy).

**Mapping в canonical:**
- `BTCUSDTM` → `BTC` (base = `BTC`, последняя `M` отбрасывается).
- `XBTUSDTM` → `BTC` (alias `XBT` = `BTC` по соглашению).

---

## 6. Filter (USDT-M Perpetual)

```python
def is_usdt_m_perpetual(item: dict) -> bool:
    return (
        item["quoteCurrency"] == "USDT"
        and item["settleCurrency"] == "USDT"
        and item["type"] == "FFWCSX"
        and item["status"] == "Open"
    )
```

Поля:
- `type`: `"FFWCSX"` (Linear Perpetual) | `"FFICSX"` (Inverse) | `"FFWFC"` (Linear Futures).
- `quoteCurrency`, `settleCurrency`: `"USDT"`.
- `status`: `"Open"` | `"BeingSettled"` | `"Settled"` | `"Paused"`.
- `multiplier`: contract size (e.g. `0.001` для BTC).
- `baseCurrency`: e.g. `"BTC"`.

---

## 7. Parser

### 7.1 От `/api/ua/v1/market/open-interest`

```python
def parse_oi_history_item(item: list, symbol: str) -> ParsedOI:
    ts_ms, oi_value = item
    return ParsedOI(
        native_symbol=symbol,
        ts_exchange=datetime.fromtimestamp(ts_ms / 1000, tz=UTC),
        oi_raw=Decimal(oi_value),
        oi_unit_hint="base_asset",
        oi_notional_provided=None,
        price_provided=None,
        raw_extras={},
    )
```

### 7.2 От `/api/v1/contracts/active`

```python
def parse_contract_snapshot(item: dict) -> ParsedOI:
    return ParsedOI(
        native_symbol=item["symbol"],
        ts_exchange=utcnow(),
        oi_raw=Decimal(str(item["openInterest"])),
        oi_unit_hint="base_asset",
        oi_notional_provided=None,
        price_provided=Decimal(str(item["markPrice"])),
        raw_extras={
            "last_trade_price": item["lastTradePrice"],
            "index_price": item["indexPrice"],
            "multiplier": item["multiplier"],
        },
    )
```

---

## 8. Unit resolution

- `oi_unit_hint = "base_asset"` (KuCoin возвращает OI в базовом активе).
- `oi_coins = oi_raw` напрямую.
- `oi_notional_usdt = oi_coins × mark_price`, `valuation_status = "good_estimate"`.

NB: KuCoin Doc явно указывает что для futures multiplier = contract_size, и `openInterest` в API — уже в base asset (post-multiplied). Но это **только для public OI endpoint**. Если бы мы видели «contracts» поля — пришлось бы умножать на `multiplier`.

---

## 9. Price resolution

- Primary: `markPrice` из `/contracts/active`.
- Fallback: `indexPrice`.
- Last fallback: `lastTradePrice`.

Bulk `/contracts/active` за 1 запрос/цикл — все нужные поля сразу.

---

## 10. Watermark logic (для native_interval)

```python
async def fetch_range(self, instrument, interval, start, end):
    granularity_map = {"5m": "Min5", "15m": "Min15", "30m": "Min30"}
    granularity = granularity_map[interval]

    last_ts = await self.watermark_repo.get_watermark(self.exchange, instrument.native_symbol, interval)
    actual_start = last_ts or start

    response = await self._http.get("/api/ua/v1/market/open-interest", params={
        "symbol": instrument.native_symbol,
        "granularity": granularity,
        "from": int(actual_start.timestamp() * 1000),
        "to": int(end.timestamp() * 1000),
    })

    items = response.json()["data"]["values"]
    if items:
        max_ts = max(item[0] for item in items)
        await self.watermark_repo.set_watermark(
            self.exchange, instrument.native_symbol, interval,
            datetime.fromtimestamp(max_ts / 1000, tz=UTC),
        )
    return [self._make_raw_event(item, ...) for item in items]
```

---

## 11. Edge cases

### 11.1 7-day retention

Если коннектор был down > 7 дней, native_interval история недоступна — gap не закрыть.

### 11.2 XBT alias

Legacy: `XBTUSDTM` для BTC. Может встречаться. В sync_instruments проверяем оба:
```python
canonical_symbol = (
    "BTC" if base in ("BTC", "XBT") else base
)
```

### 11.3 Multiplier как contract_size

Поле `multiplier` в KuCoin API это размер контракта в base asset:
- `BTCUSDTM`: `0.001` (1 контракт = 0.001 BTC).

Хранится в `instruments.contract_size`. Material change → version bump.

### 11.4 `BeingSettled` status

Контракт во время settlement. OI может быть нестабильным; в нашем scope игнорируем (фильтр `status == "Open"`).

### 11.5 Symbol whitelist note

Если пользователь использует `symbol_whitelist`, KuCoin API не имеет endpoint'а для нескольких символов одним запросом. Нужно делать 1 fetch per symbol — пробить rate limit.

Mitigation: всегда тянем bulk `/contracts/active` (для price + fallback OI), а native_interval только для нужных символов.

---

## 12. Test fixtures

```
tests/contract/kucoin/fixtures/
├── contracts_active.json
├── open_interest_min5_btcusdtm.json
├── open_interest_min15_btcusdtm.json
└── mark_price_btcusdtm.json
```

---

## 13. References

- KuCoin Futures API: https://www.kucoin.com/docs/rest/futures-trading/market-data/get-active-contract-list
- Open Interest: https://www.kucoin.com/docs/rest/futures-trading/market-data/get-open-interest-list
- Mark Price: https://www.kucoin.com/docs/rest/futures-trading/market-data/get-current-mark-price
