# 11.11 · Bitmart Connector

> USDT-M Perpetual, snapshot bulk polling.
> Base: `_BASE_CONNECTOR.md`.
> Заменил Bitunix per `00_DECISIONS_LOG.md F42` (Bitunix public API не отдавал OI).

---

## 1. Source kind

- **Primary:** `snapshot` через единственный bulk endpoint `/contract/public/details`,
  возвращающий ВСЕ futures с OI + ценой + filter metadata в одном запросе.
- Pattern идентичен Binance/Bitget bulk-tickers.

---

## 2. URLs

- **Base:** `https://api-cloud-v2.bitmart.com`
- **Доки:** https://developer-pro.bitmart.com/en/futuresv2/

---

## 3. Endpoints

### 3.1 Список инструментов + OI + цена (один bulk)

`GET /contract/public/details`

- Возвращает все contract symbols с полями включая `open_interest`, `open_interest_value`,
  `index_price`, `last_price`, `funding_rate`, `status`, `expire_timestamp`.
- Query без параметров → все контракты. С `?symbol=BTCUSDT` → один контракт.
- **Используем bulk** (1 request / poll cycle). Cadence для instruments — каждые 5 мин,
  для OI — каждые 60s, оба используют тот же endpoint (in-memory cache на длительность цикла).

### 3.2 Response shape

```json
{
  "code": 1000,
  "message": "Ok",
  "data": {
    "symbols": [
      {
        "symbol": "BTCUSDT",
        "product_type": 1,
        "expire_timestamp": 0,
        "base_currency": "BTC",
        "quote_currency": "USDT",
        "last_price": "78901.2",
        "index_price": "78954.96391304",
        "contract_size": "0.001",
        "open_interest": "2802477",
        "open_interest_value": "223069591.48427733",
        "funding_rate": "-0.0000513",
        "status": "Trading",
        "delist_time": 0
      }
    ]
  }
}
```

Прочие поля (`volume_24h`, `turnover_24h`, `funding_time`, etc.) сохраняются в
`raw_extras` без обработки.

---

## 4. Rate limits

Bitmart contract public ≈ **12 req/sec/IP** (не зафиксировано жёстко в публичных доках,
оценено эмпирически + peer-aligned).

**Конфиг (консервативный, peer-aligned):**
```python
RATE_LIMIT_CAPACITY = 10
RATE_LIMIT_REFILL_PER_SEC = 10
MAX_CONCURRENT_REQUESTS = 5
```

**Стратегия:** один bulk `/contract/public/details` за poll cycle. Никаких per-symbol
запросов в hot path.

---

## 5. Symbol naming

- Pattern: `BASEUSDT` (без разделителей, как Binance).
- Examples: `BTCUSDT`, `ETHUSDT`, `BCHUSDT`.
- `canonical_symbol = base_currency` (см. `06_INSTRUMENT_REGISTRY.md`).

---

## 6. Filter (USDT-M Perpetual)

```python
def is_usdt_m_perpetual(item: dict) -> bool:
    return (
        item["product_type"] == 1
        and item["quote_currency"] == "USDT"
        and item["expire_timestamp"] == 0      # 0 = perpetual; >0 = delivery futures
        and item["status"] == "Trading"
    )
```

Поля:
- `product_type`: `1` (perpetual).
- `quote_currency`: `"USDT"` для нашего scope.
- `base_currency`: используется как `canonical_symbol`.
- `expire_timestamp`: `0` для perpetual, ms-timestamp для delivery — **строго `== 0`**.
- `status`: `"Trading"` для активных. Прочие (`Settling`, etc.) исключаем.
- `delist_time`: `0` для не делистнутых; `> 0` означает запланированный делистинг.
- `contract_size`: per-symbol множитель (`0.001` для BTC, `0.01` для ETH, etc.). **Критично** для конверсии в `oi_coins`.

---

## 7. Parser

### 7.1 От `/contract/public/details`

```python
from datetime import UTC, datetime
from decimal import Decimal

REQUIRED_FIELDS = {
    "symbol",
    "product_type",
    "quote_currency",
    "expire_timestamp",
    "status",
    "contract_size",
    "open_interest",
    "open_interest_value",
    "index_price",
}


def parse_symbol(item: dict, *, fetched_at: datetime) -> ParsedOI:
    missing = REQUIRED_FIELDS - item.keys()
    if missing:
        raise SchemaDriftError(f"Bitmart symbol missing fields: {missing}")
    return ParsedOI(
        native_symbol=item["symbol"],
        ts_exchange=fetched_at,  # Bitmart bulk endpoint не отдаёт per-symbol timestamp
        oi_raw=Decimal(str(item["open_interest"])),
        oi_unit_hint="contracts",
        oi_notional_provided=Decimal(str(item["open_interest_value"])),
        price_provided=Decimal(str(item["index_price"])),
        raw_extras={
            "last_price": item["last_price"],
            "funding_rate": item["funding_rate"],
            "contract_size": item["contract_size"],
        },
    )
```

**Замечание про `ts_exchange`.** Bitmart `/contract/public/details` не возвращает
per-symbol server timestamp. Используем `fetched_at` (наш wall-clock в момент HTTP
response) — это та же стратегия, что у Hyperliquid и Aster при bulk-snapshot без
embedded ts. `04_TIME_MODEL.md` это покрывает: `ts_exchange` — best-effort, freshness
считается от `ts_ingested`.

---

## 8. Unit resolution

- `oi_unit_hint = "contracts"` (поддерживаемое значение в `pipeline.py`; mirrors XT/OKX/Gate.io/MEXC pattern).
- `instrument.contract_size = open_interest_value / open_interest`-derived при build, либо напрямую из `contract_size` payload (predпочтительно).
- `instrument.multiplier = 1` (Bitmart не имеет 1000-prefix токенов).
- `oi_coins = oi_raw × instrument.contract_size × instrument.multiplier` (формула pipeline Stage 3 для `"contracts"` hint).
- `oi_notional_usdt`: `open_interest_value` provided биржей →
  `valuation_status = "authoritative"`. Estimate-fallback не нужен — поле всегда
  присутствует в наблюдённых ответах.

**Sanity check** (применяется в normalizer Stage 4, не в parser):
```
abs(oi_coins × index_price - open_interest_value) / open_interest_value < 0.02
```
≥ 2% drift → лог warning + degrade `valuation_status` → `good_estimate`. Это страховка
от инкоррентного `contract_size`.

---

## 9. Price resolution

- Primary: `index_price`.
- Fallback: `last_price` (если `index_price` пусто/`null` — на практике не наблюдалось).
- `mark_price` Bitmart в этом endpoint не отдаёт; index-based valuation peer-aligned
  (OKX тоже использует index).

---

## 10. Edge cases

### 10.1 Per-symbol `contract_size` вариативен

`BTCUSDT contract_size = 0.001`, `BCHUSDT = 0.001`, но альты могут иметь другие. **Без
парсинга `contract_size` `oi_coins` будет неправильным**. Фикстура для unit-тестов
обязана включать символы с разными `contract_size`. Sanity check в §8 — defensive layer.

### 10.2 Delivery contracts (не perpetual)

Bitmart хостит и quarterly delivery (`expire_timestamp != 0`). Filter `expire_timestamp == 0`
их отрезает. Гарантия: `06_INSTRUMENT_REGISTRY.md §5` — фильтр в `sync_instruments()`,
дублируется в normalizer как страховка.

### 10.3 Schema drift detection

В parser при отсутствии любого из `REQUIRED_FIELDS` → `SchemaDriftError` →
`normalization_errors` с `failed_stage = "parse"`, `error_type = "schema_drift"`. Это
ловит и переименование полей, и регрессию (например, `open_interest_value` исчезнет в
будущей версии API).

### 10.4 Большой response

Bulk `/contract/public/details` ≈ 700–800KB JSON (наблюдалось 764KB). httpx стримит без
проблем; никаких чанков не нужно. В `raw_extras` сохраняем только подмножество
(`last_price`, `funding_rate`, `contract_size`) — НЕ весь payload.

### 10.5 Меньшая ликвидность на части символов

Bitmart — mid-tier биржа. Часть символов будет иметь OI < 5M USDT и не пройдёт default
`min_oi_notional` фильтр. Это норма — не отдельный rule, а осознанная отсечка шума
(аналогично Aster, Hyperliquid).

### 10.6 `delist_time != 0`

Symbol с запланированным делистингом продолжаем читать до `status != "Trading"`. Никакой
специальной обработки не требуется — фильтр §6 справится автоматически.

---

## 11. Test fixtures

```
backend/tests/contract/bitmart/fixtures/
├── details.json              # full live snapshot (sanitized)
└── details_minimal.json      # 3 символа (BTC, ETH, низколикв. альт) для unit-тестов
```

---

## 12. References

- Bitmart Futures API: https://developer-pro.bitmart.com/en/futuresv2/
- Endpoint `/contract/public/details`: see "Get Contract Details" в docs.
- Decision: `00_DECISIONS_LOG.md F42` (replaces F20).
