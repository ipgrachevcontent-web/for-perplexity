# 11.5 · Gate.io Connector

> USDT-M Perpetual (linear), snapshot polling.
> Base: `_BASE_CONNECTOR.md`.

---

## 1. Source kind

- **Primary:** `snapshot` через `/futures/usdt/contracts` (bulk endpoint).
- Этот endpoint возвращает все USDT-M perpetual контракты с полями `total_size` (OI in contracts) и `quanto_multiplier`.

---

## 2. URLs

- **Base:** `https://api.gateio.ws`
- **Доки:** https://www.gate.io/docs/developers/apiv4/

---

## 3. Endpoints

### 3.1 Список инструментов + OI (bulk)

`GET /api/v4/futures/usdt/contracts`

- Возвращает все USDT-M perpetual контракты.
- В ответе есть и метаданные, и `total_size` (текущее OI в контрактах).
- Cadence: каждые 5 минут (для instruments) + каждую минуту (для OI).
  - На самом деле один и тот же endpoint = можно объединить, но `instruments_sync_interval` отдельный для согласованности с другими биржами.
- Response:
```json
[
  {
    "name": "BTC_USDT",
    "type": "direct",
    "quanto_multiplier": "0.0001",
    "leverage_min": "1",
    "leverage_max": "100",
    "maintenance_rate": "0.005",
    "mark_type": "internal",
    "mark_price": "67420.5",
    "index_price": "67400.1",
    "last_price": "67425.3",
    "total_size": "12345678",
    "in_delisting": false,
    "trade_status": "tradable",
    ...
  },
  ...
]
```

### 3.2 OI history (опциональный backfill)

`GET /api/v4/futures/usdt/stats?contract=BTC_USDT&from=...&interval=5m`

- intervals: `5m`, `15m`, `30m`, `1h`, `4h`, `1d`.
- Используем при необходимости backfill.

### 3.3 Tickers (альтернатива)

`GET /api/v4/futures/usdt/tickers`

- Возвращает `last`, `mark_price`, `index_price`, `total_size` для всех контрактов.

---

## 4. Rate limits

- 200 req / 10sec для public futures endpoints.
- При превышении: HTTP 429.

**Конфиг:**
```python
RATE_LIMIT_CAPACITY = 200
RATE_LIMIT_REFILL_PER_SEC = 20
MAX_CONCURRENT_REQUESTS = 10
```

**Стратегия:** один bulk-запрос `/futures/usdt/contracts` за цикл — covers и OI, и цены, и метаданные. Огромная экономия rate limit.

---

## 5. Symbol naming

- Pattern: `BASE_USDT` (с подчёркиванием).
- Examples: `BTC_USDT`, `ETH_USDT`, `SOL_USDT`.

---

## 6. Filter (USDT-M Perpetual)

```python
def is_usdt_m_perpetual(item: dict) -> bool:
    return (
        item["type"] == "direct"          # linear
        and item["trade_status"] == "tradable"
        and not item.get("in_delisting", False)
        and item["name"].endswith("_USDT")
    )
```

Поля:
- `type`: `"direct"` (linear) | `"inverse"` (COIN-M).
- `trade_status`: `"tradable"` | `"untradable"`.
- `in_delisting`: bool — если true, контракт скоро удалится.
- `name`: `BASE_USDT` для нашего scope.

---

## 7. Parser

### 7.1 От `/futures/usdt/contracts` (bulk)

```python
def parse_contract(item: dict) -> ParsedOI:
    return ParsedOI(
        native_symbol=item["name"],
        ts_exchange=utcnow(),     # Gate не возвращает timestamp в bulk; берём server clock
        oi_raw=Decimal(item["total_size"]),         # в контрактах
        oi_unit_hint="contracts",
        oi_notional_provided=None,
        price_provided=Decimal(item["mark_price"]),
        raw_extras={
            "quanto_multiplier": item["quanto_multiplier"],
            "last_price": item["last_price"],
            "index_price": item["index_price"],
        },
    )
```

---

## 8. Unit resolution

- `oi_unit_hint = "contracts"`.
- `quanto_multiplier` — это `contract_size` (size of 1 contract in base asset).
- `oi_coins = oi_raw × quanto_multiplier`.

`quanto_multiplier` хранится в `instruments.contract_size` после sync_instruments. **Material change → instrument_version bump.**

```python
def resolve_unit(parsed, instrument):
    contract_size = instrument.contract_size  # = quanto_multiplier
    oi_coins = parsed.oi_raw * contract_size
    return ResolvedUnit(oi_coins=oi_coins, method="contracts_to_base",
                        details={"contract_size": contract_size})
```

`oi_notional_usdt = oi_coins × mark_price`.
`valuation_status = "good_estimate"`.

---

## 9. Price resolution

- Primary: `mark_price` из `/futures/usdt/contracts` (приходит вместе с OI).
- Fallback: `index_price` из того же endpoint'а.
- Last fallback: `last_price`.

---

## 10. Edge cases

### 10.1 quanto_multiplier varies

`quanto_multiplier`:
- `BTC_USDT`: `"0.0001"` (1 contract = 0.0001 BTC).
- `ETH_USDT`: `"0.01"` (1 contract = 0.01 ETH).
- Большая часть альтов: `"1"` или `"10"`.

### 10.2 in_delisting flag

Если `in_delisting == true`, контракт перестанет торговаться скоро. Пишем warning в `Instrument`, продолжаем опрашивать пока torgable.

### 10.3 Inverse contracts

Inverse (`type == "inverse"`) — COIN-M. Фильтруются.

### 10.4 BTC_USDT_PERP альтернативное naming

В некоторых docs встречается `BTC_USDT_PERP`. Текущий V4 API использует `BTC_USDT`. Проверить при разработке; если найдётся `_PERP` — добавить в filter.

### 10.5 Pre-launch contracts

Иногда Gate добавляет контракт за несколько часов до начала торгов. `trade_status != "tradable"` → не активный.

---

## 11. Test fixtures

```
tests/contract/gateio/fixtures/
├── futures_usdt_contracts.json
├── futures_usdt_tickers.json
├── futures_usdt_stats_btc_usdt.json
└── futures_usdt_contracts_with_delisting.json
```

---

## 12. References

- Gate.io API V4: https://www.gate.io/docs/developers/apiv4/
- Futures contracts: https://www.gate.io/docs/developers/apiv4/#list-all-futures-contracts
- Futures stats: https://www.gate.io/docs/developers/apiv4/#futures-stats
