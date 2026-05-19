# 11.3 · OKX Connector

> USDT-M Perpetual SWAP, snapshot polling.
> Base: `_BASE_CONNECTOR.md`.

---

## 1. Source kind

- **Primary:** `snapshot` — `/api/v5/public/open-interest` отдаёт текущее OI.
- OKX поддерживает bulk-запрос всех SWAP за один вызов через `instType=SWAP` без `instId` — рекомендуется использовать.

---

## 2. URLs

- **Base:** `https://www.okx.com`
- **Доки:** https://www.okx.com/docs-v5/en/

---

## 3. Endpoints

### 3.1 Список инструментов

`GET /api/v5/public/instruments?instType=SWAP`

- Возвращает все SWAP инструменты.
- Cadence: каждые 5 минут.

### 3.2 Open Interest (bulk snapshot)

`GET /api/v5/public/open-interest?instType=SWAP`

- Возвращает OI по всем SWAP инструментам за один запрос (рекомендуется).
- Опционально `&instId=BTC-USDT-SWAP` для одного инструмента.
- Response:
```json
{
  "code": "0",
  "data": [
    {
      "instType": "SWAP",
      "instId": "BTC-USDT-SWAP",
      "oi": "5000",
      "oiCcy": "5000",
      "ts": "1597026383085"
    },
    ...
  ]
}
```

`oi` — в контрактах (number of contracts).
`oiCcy` — в base currency, ИЛИ в quote, в зависимости от типа (см. ниже).

### 3.3 Mark price

`GET /api/v5/public/mark-price?instType=SWAP`

- Bulk запрос всех mark prices.
- Response:
```json
{
  "code": "0",
  "data": [
    {
      "instType": "SWAP",
      "instId": "BTC-USDT-SWAP",
      "markPx": "67420.5",
      "ts": "1597026383085"
    },
    ...
  ]
}
```

### 3.4 Index ticker (для index price, опционально)

`GET /api/v5/market/index-tickers?instId=BTC-USDT`

---

## 4. Rate limits

- Public market endpoints: 20 req / 2 sec на IP.
- При превышении: HTTP 429.

**Конфиг:**
```python
RATE_LIMIT_CAPACITY = 20
RATE_LIMIT_REFILL_PER_SEC = 10
MAX_CONCURRENT_REQUESTS = 5
```

Bulk-запрос делает rate limit некритичным: 3 запроса за цикл (instruments + open-interest + mark-price).

---

## 5. Symbol naming

- Pattern: `BASE-USDT-SWAP`.
- Examples: `BTC-USDT-SWAP`, `ETH-USDT-SWAP`, `SOL-USDT-SWAP`.
- `BASE-USDT-SWAP` — перпетуальный своп с маржой в USDT. Внутренне это эквивалент USDT-M perp; quote = USDT, SWAP в конце.

---

## 6. Filter (USDT-M Perpetual SWAP)

```python
def is_usdt_m_perpetual(item: dict) -> bool:
    return (
        item["instType"] == "SWAP"
        and item["settleCcy"] == "USDT"
        and item["state"] == "live"
    )
```

Поля:
- `instType`: `"SWAP"` | `"FUTURES"` | `"OPTION"` | `"SPOT"` | `"MARGIN"`.
- `settleCcy`: `"USDT"` для USDT-M.
- `ctType`: `"linear"` | `"inverse"` (для SWAP).
- `state`: `"live"` | `"suspend"` | `"preopen"` | `"test"`.

---

## 7. Parser

### 7.1 От `/api/v5/public/open-interest`

```python
def parse_open_interest(item: dict) -> ParsedOI:
    return ParsedOI(
        native_symbol=item["instId"],
        ts_exchange=datetime.fromtimestamp(int(item["ts"]) / 1000, tz=UTC),
        oi_raw=Decimal(item["oi"]),                # в контрактах
        oi_unit_hint="contracts",
        oi_notional_provided=None,
        price_provided=None,
        raw_extras={"oi_ccy": item.get("oiCcy")},
    )
```

`oi_ccy` — для linear SWAP это значение OI в base currency (BTC), но мы предпочитаем считать через `oi (contracts) × ctVal` для согласованности.

---

## 8. Unit resolution

- `oi_unit_hint = "contracts"`.
- Для linear USDT-M SWAP в OKX: `ctVal` (contract value в base asset) — обычно равен `0.01` или `0.001` для разных инструментов.
- `oi_coins = oi_raw × ctVal × ctMult` (где `ctMult` обычно `1`).

`ctVal`, `ctMult`, `ctType` хранятся в `instruments.raw_metadata` после sync_instruments.

```python
def resolve_unit(parsed, instrument):
    ct_val = Decimal(instrument.raw_metadata["ctVal"])
    ct_mult = Decimal(instrument.raw_metadata.get("ctMult", "1"))
    oi_coins = parsed.oi_raw * ct_val * ct_mult
    return ResolvedUnit(oi_coins=oi_coins, method="contracts_to_base",
                        details={"ct_val": ct_val, "ct_mult": ct_mult})
```

`oi_notional_usdt = oi_coins × mark_price`.
`valuation_status = "good_estimate"`.

---

## 9. Price resolution

- Primary: `markPx` из `/api/v5/public/mark-price`.
- Fallback: index price из `/api/v5/market/index-tickers`.
- Last fallback: ticker last (`/api/v5/market/ticker`).

Bulk-запрос `/api/v5/public/mark-price?instType=SWAP` рекомендуется — все mark prices за один вызов.

---

## 10. Edge cases

### 10.1 ctVal varies per instrument

В отличие от Binance/Bybit где OI всегда в base, OKX контракты имеют **разные** `ctVal`:
- `BTC-USDT-SWAP`: ctVal = "0.01" (1 контракт = 0.01 BTC).
- `ETH-USDT-SWAP`: ctVal = "0.1" (1 контракт = 0.1 ETH).
- `SOL-USDT-SWAP`: ctVal = "1" (1 контракт = 1 SOL).

**Material change в `ctVal` вызывает `instrument_version` bump** (см. `06_INSTRUMENT_REGISTRY.md §4`).

### 10.2 Inverse swap mistakenly included

OKX имеет также COIN-M inverse SWAP (`BTC-USD-SWAP`). Фильтр `settleCcy == "USDT"` отсекает.

### 10.3 PreLaunch / suspend

`state != "live"` → не активный, в опросе не участвует.

### 10.4 `oiCcy` semantics

Для linear: `oiCcy` обычно в base. Но мы используем `oi (contracts) × ctVal` для надёжности — это однозначно.

---

## 11. Test fixtures

```
tests/contract/okx/fixtures/
├── instruments_swap.json
├── open_interest_swap_bulk.json
├── mark_price_swap_bulk.json
└── index_ticker_btc_usdt.json
```

---

## 12. References

- OKX API V5: https://www.okx.com/docs-v5/en/
- Open Interest: https://www.okx.com/docs-v5/en/#public-data-rest-api-get-open-interest
- Mark Price: https://www.okx.com/docs-v5/en/#public-data-rest-api-get-mark-price
- Instruments: https://www.okx.com/docs-v5/en/#public-data-rest-api-get-instruments
