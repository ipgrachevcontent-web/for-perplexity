# 11.6 · MEXC Connector

> USDT-M Perpetual, snapshot polling.
> Base: `_BASE_CONNECTOR.md`.

---

## 1. Source kind

- **Primary:** `snapshot` через `/api/v1/contract/ticker` (bulk) или `/api/v1/contract/open_interest` per symbol.

---

## 2. URLs

- **Base:** `https://contract.mexc.com`
- **Доки:** https://mxcdevelop.github.io/apidocs/contract_v1_en/

---

## 3. Endpoints

### 3.1 Список инструментов

`GET /api/v1/contract/detail`

- Возвращает все futures contracts с метаданными.
- Cadence: каждые 5 минут.

### 3.2 Open Interest (per symbol)

`GET /api/v1/contract/open_interest/BTC_USDT`

- Возвращает текущий OI для одного символа.
- Response:
```json
{
  "success": true,
  "code": 0,
  "data": {
    "symbol": "BTC_USDT",
    "holdVol": 12345.678,
    "timestamp": 1714478700000
  }
}
```

`holdVol` — в контрактах.

### 3.3 Ticker (bulk OI + price)

`GET /api/v1/contract/ticker`

- Возвращает все contract tickers с полями: `lastPrice`, `holdVol`, `fairPrice` (= mark price), `indexPrice`.
- **Удобно для bulk fetch.**

### 3.4 Mark price

Можно из `/contract/ticker` (`fairPrice`) или отдельно `/api/v1/contract/index_price/BTC_USDT`.

---

## 4. Rate limits

- 20 req / 2 sec для public endpoints.
- HTTP 429 при превышении.

**Конфиг:**
```python
RATE_LIMIT_CAPACITY = 20
RATE_LIMIT_REFILL_PER_SEC = 10
MAX_CONCURRENT_REQUESTS = 5
```

**Стратегия:** bulk `/contract/ticker` за 1 запрос/цикл.

---

## 5. Symbol naming

- Pattern: `BASE_USDT` (с подчёркиванием).
- Examples: `BTC_USDT`, `ETH_USDT`, `SOL_USDT`.

NB: на UI сайте MEXC может показываться `BTC/USDT`, но API использует `_`.

---

## 6. Filter (USDT-M Perpetual)

```python
def is_usdt_m_perpetual(item: dict) -> bool:
    return (
        item["quoteCoin"] == "USDT"
        and item["state"] == 0          # 0 = enabled
        and item["contractType"] == 1    # 1 = perpetual
    )
```

Поля:
- `quoteCoin`: `"USDT"`.
- `baseCoin`: e.g. `"BTC"`.
- `state`: `0` (enabled), `1` (delivering), `2` (completed), `3` (offline), `4` (pause).
- `contractType`: `1` (perpetual), `2` (delivery).
- `contractSize`: e.g. `0.0001` (для BTC_USDT).

---

## 7. Parser

### 7.1 От `/api/v1/contract/open_interest`

```python
def parse_open_interest(payload: dict) -> ParsedOI:
    data = payload["data"]
    return ParsedOI(
        native_symbol=data["symbol"],
        ts_exchange=datetime.fromtimestamp(data["timestamp"] / 1000, tz=UTC),
        oi_raw=Decimal(str(data["holdVol"])),
        oi_unit_hint="contracts",
        oi_notional_provided=None,
        price_provided=None,
        raw_extras={},
    )
```

### 7.2 От `/api/v1/contract/ticker`

```python
def parse_ticker(item: dict) -> ParsedOI:
    return ParsedOI(
        native_symbol=item["symbol"],
        ts_exchange=datetime.fromtimestamp(item["timestamp"] / 1000, tz=UTC),
        oi_raw=Decimal(str(item["holdVol"])),
        oi_unit_hint="contracts",
        oi_notional_provided=None,
        price_provided=Decimal(str(item["fairPrice"])),
        raw_extras={
            "last_price": item["lastPrice"],
            "index_price": item["indexPrice"],
        },
    )
```

---

## 8. Unit resolution

- `oi_unit_hint = "contracts"`.
- `oi_coins = oi_raw × contract_size` (где `contract_size` — из `instruments.contract_size`).
- Для большинства MEXC USDT-M контрактов `contract_size` = `0.0001` (BTC), `0.01` (ETH), `1` (альты).

```python
def resolve_unit(parsed, instrument):
    contract_size = instrument.contract_size
    oi_coins = parsed.oi_raw * contract_size
    return ResolvedUnit(oi_coins=oi_coins, method="contracts_to_base",
                        details={"contract_size": contract_size})
```

`oi_notional_usdt = oi_coins × fair_price`.
`valuation_status = "good_estimate"`.

---

## 9. Price resolution

- Primary: `fairPrice` из `/contract/ticker`.
- Fallback: `indexPrice` оттуда же.
- Last fallback: `lastPrice`.

---

## 10. Edge cases

### 10.1 contract_size variations

Material change в `contract_size` (в API поле `contractSize`) → `instrument_version` bump.

### 10.2 state codes

- `0` (enabled) — нормальная торговля, опрашиваем.
- `1` (delivering) — во время settlement, OI может быть нестабильным.
- `2` (completed) — закрыт навсегда, `delisted_at = now()`.
- `3` (offline) — временно выключен, не опрашиваем но не делистим.
- `4` (pause) — пауза торгов; OI стабилен.

В нашей системе:
- `0` → активный.
- `1`, `4` → активный, но добавляем warning `unstable_state`.
- `2`, `3` → не активный, sync_job ставит `delisted_at` (для `2`) или skip (для `3`).

### 10.3 holdVol units

Документация MEXC явно указывает: `holdVol` is in **contracts**. Без contract_size конверсия неправильная.

### 10.4 v1 vs v2 API

MEXC contract API всё ещё на v1 (актуально на момент написания). Spot API имеет v3, но для futures — v1.

---

## 11. Test fixtures

```
tests/contract/mexc/fixtures/
├── contract_detail.json
├── contract_open_interest_btc_usdt.json
├── contract_ticker.json
└── contract_index_price.json
```

---

## 12. References

- MEXC Contract API V1: https://mxcdevelop.github.io/apidocs/contract_v1_en/
- Open Interest: https://mxcdevelop.github.io/apidocs/contract_v1_en/#get-contract-open-interest
- Tickers: https://mxcdevelop.github.io/apidocs/contract_v1_en/#get-all-contract-information
