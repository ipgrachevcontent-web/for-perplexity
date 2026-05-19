# 11.12 · XT Connector

> USDT-M Perpetual, **bulk snapshot** (один endpoint = все контракты).
> Base: `_BASE_CONNECTOR.md`.

---

## 1. Source kind

- **Primary:** `snapshot` через bulk endpoint `/future/market/v1/public/cg/contracts`.
- Один запрос возвращает все perpetual contracts с current state и `open_interest`.

---

## 2. URLs

- **Base:** `https://fapi.xt.com`
- **Доки:** https://doc.xt.com/

---

## 3. Endpoints

### 3.1 Bulk contracts list + OI (один запрос на всё)

`GET /future/market/v1/public/cg/contracts`

- Cadence: каждую минуту (для OI) + каждые 5 минут (для metadata через тот же endpoint).
- Response (массив):
```json
{
  "rc": 0,
  "mc": "SUCCESS",
  "result": [
    {
      "symbol": "btc_usdt",
      "productType": "PERPETUAL",
      "underlyingType": 1,                 // 1 = USDT-M, 2 = COIN-M
      "contractSize": "0.0001",
      "open_interest": "2419347630",        // в контрактах
      "indexPrice": "67400.10",
      "markPrice": "67420.50",
      "fundingRate": "0.00012",
      "state": 0,                           // 0 = enabled
      ...
    },
    ...
  ]
}
```

### 3.2 Mark price (опциональный)

`GET /future/market/v1/public/q/symbol-mark-price?symbol=btc_usdt`

- Используем при необходимости, обычно `markPrice` из bulk достаточно.

### 3.3 Symbol detail (опциональный)

`GET /future/market/v1/public/symbol/detail?symbol=btc_usdt`

- Доп. метаданные.

---

## 4. Rate limits

- 50 req / 10 sec для public market endpoints.
- HTTP 429 при превышении.

**Конфиг:**
```python
RATE_LIMIT_CAPACITY = 50
RATE_LIMIT_REFILL_PER_SEC = 5
MAX_CONCURRENT_REQUESTS = 3
```

**Стратегия:** один bulk-запрос за цикл. С учётом 1 запроса/мин = 1/60 от лимита, плюс sync_instruments тот же endpoint = переиспользуем.

---

## 5. Symbol naming

- Pattern: `base_usdt` (lowercase, с подчёркиванием).
- Examples: `btc_usdt`, `eth_usdt`, `sol_usdt`.

NB: lowercase! При сравнении делаем `.lower()`. Canonical mapping:
- `btc_usdt` → `BTC` (uppercase).

---

## 6. Filter (USDT-M Perpetual)

```python
def is_usdt_m_perpetual(item: dict) -> bool:
    return (
        item["productType"] == "PERPETUAL"
        and item["underlyingType"] == 1     # 1 = USDT-M
        and item["state"] == 0              # 0 = enabled
        and item["symbol"].endswith("_usdt")
    )
```

Поля:
- `productType`: `"PERPETUAL"` | `"DELIVERY"`.
- `underlyingType`: `1` (USDT-M) | `2` (COIN-M).
- `state`: `0` (enabled), `1` (delisting), `2` (offline).
- `contractSize`: обязательное поле, разное per-symbol.

---

## 7. Parser

### 7.1 От `/future/market/v1/public/cg/contracts`

```python
def parse_contract_item(item: dict) -> ParsedOI:
    return ParsedOI(
        native_symbol=item["symbol"].lower(),
        ts_exchange=utcnow(),                    # bulk endpoint без per-asset ts
        oi_raw=Decimal(str(item["open_interest"])),
        oi_unit_hint="contracts",
        oi_notional_provided=None,
        price_provided=Decimal(str(item["markPrice"])),
        raw_extras={
            "contract_size": item["contractSize"],
            "index_price": item["indexPrice"],
            "underlying_type": item["underlyingType"],
        },
    )
```

---

## 8. Unit resolution

- `oi_unit_hint = "contracts"`.
- XT docs: `open_interest` is in **contracts**, not USDT.
- `oi_coins = oi_raw × contract_size`.

```python
def resolve_unit(parsed, instrument):
    contract_size = instrument.contract_size  # из instruments.contract_size
    oi_coins = parsed.oi_raw * contract_size
    return ResolvedUnit(oi_coins=oi_coins, method="contracts_to_base",
                        details={"contract_size": contract_size})
```

`oi_notional_usdt = oi_coins × mark_price`.
`valuation_status = "good_estimate"`.

---

## 9. Price resolution

- Primary: `markPrice` из bulk endpoint.
- Fallback: `indexPrice`.
- Last fallback: отдельный mark-price endpoint.

---

## 10. Edge cases

### 10.1 contract_size critical

Без `contract_size` нельзя правильно конвертировать. Если поле отсутствует или меняется → instrument_version bump (см. `06_INSTRUMENT_REGISTRY.md §4`).

### 10.2 ts_exchange отсутствует

В bulk payload нет per-asset timestamp. Используем `fetched_at` (server clock), помечаем `ts_exchange_inferred` warning.

### 10.3 Coin-margined contracts

`underlyingType == 2` — COIN-M, не наш scope. Фильтруем.

### 10.4 Lowercase symbol

В UI XT показывает `BTC/USDT`, в API — `btc_usdt`. Стандартизируем lowercase в `native_symbol`. Canonical всё равно uppercase (`BTC`).

### 10.5 state codes

- `0`: enabled, опрашиваем.
- `1`: delisting (контракт скоро удалится). Опрашиваем, но добавляем warning.
- `2`: offline. Не опрашиваем; sync_job ставит `delisted_at`.

### 10.6 Schema variations

XT API имеет историю изменений (поле `open_interest` vs `openInterest`). Текущая версия использует snake_case. При парсинге fallback на оба варианта:

```python
oi_raw = Decimal(str(item.get("open_interest") or item.get("openInterest")))
```

### 10.7 Bulk endpoint size

С ~300 USDT-M символов размер response ~50–100 KB. Это один большой запрос, не ужасно, но decode может занимать ~10 ms. Учитываем в latency budget.

---

## 11. Test fixtures

```
tests/contract/xt/fixtures/
├── cg_contracts.json
├── cg_contracts_with_delisting.json
├── symbol_mark_price_btc_usdt.json
└── symbol_detail_btc_usdt.json
```

---

## 12. References

- XT Futures API: https://doc.xt.com/
- Bulk Contracts: see "GET /future/market/v1/public/cg/contracts" in docs
- Symbol detail: see "GET /future/market/v1/public/symbol/detail" in docs
- Mark price: see "GET /future/market/v1/public/q/symbol-mark-price" in docs
