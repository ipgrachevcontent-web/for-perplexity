# 11.10 · Aster Connector

> Binance-style derivatives exchange. USDT-M Perpetual, snapshot polling.
> Base: `_BASE_CONNECTOR.md`.

---

## 1. Source kind

- **Primary:** `snapshot` через `/fapi/v1/openInterest` (binance-like API).

---

## 2. URLs

- **Base:** `https://fapi.asterdex.com` (или аналогичный — фиксируется при разработке).
- **Доки:** Aster документирует API в стиле Binance Futures; следить за актуальной версией на их сайте.

---

## 3. Endpoints

Aster позиционирует свой futures API как Binance-compatible. Это означает, что endpoint paths и payload формат повторяют Binance.

### 3.1 Список инструментов

`GET /fapi/v1/exchangeInfo`

- Аналогично Binance.

### 3.2 Open Interest

`GET /fapi/v1/openInterest?symbol=BTCUSDT`

- Аналогично Binance.

### 3.3 Mark price

`GET /fapi/v1/premiumIndex?symbol=BTCUSDT`

- Bulk без symbol — все pairs.

### 3.4 5m trade volume (klines) — F51

`GET /fapi/v1/klines?symbol=BTCUSDT&interval=5m&limit=2`

- Binance-compatible. Возвращает массив 12-элементных списков. Поля по индексам: `[0] openTime ms`, `[5] volume base`, `[7] quoteAssetVolume USDT`, `[8] numberOfTrades`.
- Используется **только для Aster**, потому что 5m OI на Aster нерепрезентативен (OI cadence 5-25 мин — см. `F51`).
- `limit=2` отдаёт closed bucket + running bucket. Repo (`volume_samples` `bulk_upsert`) UPSERT'ит running bucket каждую минуту до его закрытия.
- Parser: `app.normalizer.parsers.aster_klines.parse_kline`.
- Storage: `volume_samples_5m` hypertable (мигр. `0016`).

---

## 4. Rate limits

Точные лимиты Aster документирует в `exchangeInfo.rateLimits`. На момент написания неизвестны точно публично, но binance-like структура предполагает аналогичные лимиты:

**Конфиг (стартовая оценка, корректировать после первого деплоя):**
```python
RATE_LIMIT_CAPACITY = 1200
RATE_LIMIT_REFILL_PER_SEC = 20
MAX_CONCURRENT_REQUESTS = 10
```

---

## 5. Symbol naming

- Pattern: `BASEUSDT` (без разделителей, как Binance).
- Examples: `BTCUSDT`, `ETHUSDT`, `SOLUSDT`.

---

## 6. Filter (USDT-M Perpetual)

```python
def is_usdt_m_perpetual(symbol_info: dict) -> bool:
    return (
        symbol_info["contractType"] == "PERPETUAL"
        and symbol_info["quoteAsset"] == "USDT"
        and symbol_info.get("status", "TRADING") == "TRADING"
    )
```

Поля идентичны Binance. Если структура отличается — поправить per-field.

---

## 7. Parser

Идентичен Binance (наследуется через `BinanceLikeConnector`):

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

---

## 8. Unit resolution

- `oi_unit_hint = "base_asset"`.
- `oi_coins = oi_raw`.
- `oi_notional_usdt = oi_coins × mark_price`, `valuation_status = "good_estimate"`.

---

## 9. Price resolution

- Primary: `markPrice` из `/fapi/v1/premiumIndex`.
- Fallback: `indexPrice`.
- Last fallback: ticker last.

---

## 10. Inheritance pattern

В коде:

```python
# app/exchanges/aster.py
from app.exchanges.binance import BinanceLikeBase

class AsterConnector(BinanceLikeBase):
    exchange = "aster"
    BASE_URL = "https://fapi.asterdex.com"
    primary_source_kind = SourceKind.SNAPSHOT
    connector_version = "1.0.0"

    # Override only при отличиях:
    # - error code mappings
    # - rate limit specifics
    # - field name differences (если есть)
```

`BinanceLikeBase` — общий класс, реализующий binance-style API logic. Сам `BinanceConnector` тоже от него наследуется.

---

## 11. Edge cases

### 11.1 Не 1:1 совместимость с Binance

Хотя API похож, не все эндпоинты могут быть идентичны:
- Коды ошибок могут отличаться.
- `exchangeInfo.symbols[].status` может иметь дополнительные значения.
- Rate limit headers могут называться по-другому.

**Mitigation:** при первом deploy — собрать суит реальных responses, сравнить с Binance fixtures, обновить parser при расхождениях.

### 11.2 Меньшая ликвидность

Aster — небольшая биржа. OI часто < 1M USDT. `min_oi_notional = 5_000_000` (default) **отфильтрует все Aster символы** для базовых правил.

Решение: рассмотреть отдельный default min_oi_notional для Aster (например `1_000_000`) или добавить как отдельное правило с пониженным фильтром (например, only top 20 Aster pairs).

### 11.3 Public listing not yet finalized

На момент написания Aster не имеет полного списка contracts в стабильном API. Список может меняться чаще обычного.

### 11.4 Trading 24/7

Like Binance, Aster torguets 24/7 без maintenance windows. Если cycle returns empty — это reason for circuit breaker, не ожидаемое.

---

## 12. Test fixtures

```
tests/contract/aster/fixtures/
├── exchange_info.json
├── open_interest_btcusdt.json
├── premium_index.json
└── README.md                   # Заметки о расхождениях с Binance
```

---

## 13. References

- Aster API docs (when available): aster's official site
- Reference: Binance Futures API (for compatibility)
