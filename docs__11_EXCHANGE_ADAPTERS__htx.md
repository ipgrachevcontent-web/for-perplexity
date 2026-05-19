# 11.8 · HTX (Huobi) Connector

> USDT-M Linear Swap, snapshot polling.
> Base: `_BASE_CONNECTOR.md`.

---

## 1. Source kind

- **Primary:** `snapshot` через `/linear-swap-api/v1/swap_open_interest`.

---

## 2. URLs

- **Base:** `https://api.hbdm.com`
- **Доки:** https://www.htx.com/en-us/opend/newApiPages/?id=8cb95dc1-77b5-11ed-9966-0242ac110003

---

## 3. Endpoints

### 3.1 Список инструментов

`GET /linear-swap-api/v1/swap_contract_info?support_margin_mode=cross&pair=*-USDT`

- Возвращает все linear swap contracts.
- Cadence: каждые 5 минут.

Альтернативно:
`GET /linear-swap-api/v1/swap_index?contract_code=BTC-USDT` — по одному.

### 3.2 Open Interest (per symbol or bulk)

`GET /linear-swap-api/v1/swap_open_interest?contract_code=BTC-USDT`

- Возвращает текущий OI для одного символа. Без `contract_code` возвращает все.
- Response:
```json
{
  "status": "ok",
  "data": [
    {
      "volume": 12345.0,                    // в контрактах
      "amount": 12.345,                     // в base asset
      "symbol": "BTC",
      "value": 832500000.0,                 // в USDT
      "contract_code": "BTC-USDT",
      "trade_amount": 100.5,
      "trade_volume": 100500.0,
      "trade_turnover": 6800000.0,
      "business_type": "swap",
      "pair": "BTC-USDT",
      "contract_type": "swap",
      "trade_partition": "USDT",
    },
    ...
  ],
  "ts": 1714478700000
}
```

`amount` — в base asset (используем как primary).
`value` — в USDT (authoritative notional).

### 3.3 Mark price

`GET /linear-swap-api/v1/swap_mark_price?contract_code=BTC-USDT`

- Возвращает текущую mark price.

Bulk: `/linear-swap-api/v1/swap_batchmerged?contract_code=BTC-USDT,ETH-USDT,...` — но есть лимит 49 символов на запрос.

Альтернативно: `/linear-swap-api/v1/swap_index_kline` для index price kline.

---

## 4. Rate limits

- 800 req / 10 sec для public endpoints.
- HTTP 429 при превышении.

**Конфиг:**
```python
RATE_LIMIT_CAPACITY = 800
RATE_LIMIT_REFILL_PER_SEC = 80
MAX_CONCURRENT_REQUESTS = 20
```

**Стратегия:** bulk OI endpoint (без contract_code) и bulk_merged для prices с лимитом 49 — несколько запросов.

---

## 5. Symbol naming

- Pattern: `BASE-USDT` (с дефисом).
- Examples: `BTC-USDT`, `ETH-USDT`, `SOL-USDT`.

Поле `contract_code` в API.

---

## 6. Filter (USDT-M Linear Swap)

```python
def is_usdt_m_perpetual(item: dict) -> bool:
    return (
        item["business_type"] == "swap"
        and item["contract_type"] == "swap"
        and item["trade_partition"] == "USDT"
        and item["contract_status"] == 1     # 1 = trading
    )
```

Поля:
- `business_type`: `"swap"` | `"futures"` | `"options"`.
- `contract_type`: `"swap"` (perpetual) | `"this_week"` | `"next_week"` | `"this_quarter"` | `"next_quarter"`.
- `trade_partition`: `"USDT"` | `"USDC"`.
- `contract_status`: `0` (delisting), `1` (trading), `2` (matching but not trading), `3` (suspended), `4` (settling), `5` (delivering), `6` (settlement done), `7` (delivery done).

---

## 7. Parser

### 7.1 От `/swap_open_interest`

```python
def parse_open_interest(item: dict, ts_ms: int) -> ParsedOI:
    return ParsedOI(
        native_symbol=item["contract_code"],
        ts_exchange=datetime.fromtimestamp(ts_ms / 1000, tz=UTC),
        oi_raw=Decimal(str(item["amount"])),         # в base asset
        oi_unit_hint="base_asset",
        oi_notional_provided=Decimal(str(item["value"])),  # в USDT — authoritative
        price_provided=None,
        raw_extras={"volume_contracts": item["volume"]},
    )
```

`value` (USDT notional) → `valuation_status = "authoritative"` (HTX сам считает с mark price).

---

## 8. Unit resolution

- `oi_unit_hint = "base_asset"`.
- `oi_coins = oi_raw` напрямую.
- `oi_notional_usdt`:
  - Если `value` в payload → используем как authoritative.
  - Иначе → `oi_coins × mark_price`, `valuation_status = "good_estimate"`.

---

## 9. Price resolution

- Primary: `mark_price` из `/swap_mark_price` (per symbol) или из bulk `swap_batchmerged`.
- Fallback: index price из `/swap_index_kline`.
- Last fallback: last price из ticker.

Если у нас уже есть `oi_notional_provided` (= `value`) → цена не критична для notional, но всё равно нужна для UI и Δ verification.

---

## 10. Edge cases

### 10.1 contract_code formats

HTX исторически имела разные форматы:
- Legacy: `BTC_USDT` (подчёркивание) — устарел.
- Current: `BTC-USDT` (дефис) — используем.

Если sync_instruments вернёт `_` formatted — конвертируем при записи в `instruments.native_symbol`.

### 10.2 USDC swap

`trade_partition == "USDC"` → out of scope.

### 10.3 contract_status transitions

- `1` (trading) → активный.
- `0`, `3`, `4`, `5`, `6`, `7` → не активный.
- `2` (matching) → переходное состояние, обычно очень короткое; не активный.

### 10.4 amount vs volume

- `amount` — OI в base asset (BTC).
- `volume` — OI в контрактах.

Для linear USDT-M HTX контракт = 1 USD notional, что означает специальную конверсию. **ВАЖНО:** для linear-swap `volume = amount × price` (примерно), но это не точно. Лучше использовать `amount` (base) как primary.

### 10.5 Renaming Huobi → HTX

Бренд изменился в 2023, но API URL'ы остались `hbdm.com`. Не нужно менять, просто учитываем.

---

## 11. Test fixtures

```
tests/contract/htx/fixtures/
├── swap_contract_info.json
├── swap_open_interest_bulk.json
├── swap_open_interest_btc_usdt.json
├── swap_mark_price_btc_usdt.json
└── swap_batchmerged.json
```

---

## 12. References

- HTX Linear Swap API: https://www.htx.com/en-us/opend/newApiPages/?id=8cb95dc1-77b5-11ed-9966-0242ac110003
- Open Interest: see "Get Contract Open Interest Information" in docs
- Contract Info: see "Get Swap Contract Information" in docs
