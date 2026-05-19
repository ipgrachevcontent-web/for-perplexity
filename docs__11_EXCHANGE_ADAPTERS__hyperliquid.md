# 11.9 · Hyperliquid Connector

> Decentralized perpetual exchange. Snapshot polling через universe + meta endpoints.
> Base: `_BASE_CONNECTOR.md`.

---

## 1. Source kind

- **Primary:** `snapshot` через POST-based info endpoint `/info` с типом запроса `metaAndAssetCtxs` или `meta`.

---

## 2. URLs

- **Base:** `https://api.hyperliquid.xyz`
- **Доки:** https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/info-endpoint

---

## 3. Endpoints

Hyperliquid использует **POST-based JSON API**, не REST GET.

### 3.1 metaAndAssetCtxs (всё за один запрос)

`POST /info`
Body:
```json
{
  "type": "metaAndAssetCtxs"
}
```

Response: `[meta, ctxs]` где:
- `meta.universe` — список perpetual market'ов с metadata.
- `ctxs` — массив объектов с current state каждого market'а: `markPx`, `oraclePx`, `funding`, `openInterest`, `prevDayPx`, etc.

```json
[
  {
    "universe": [
      {
        "name": "BTC",
        "szDecimals": 5,
        "maxLeverage": 50,
        "onlyIsolated": false
      },
      {
        "name": "ETH",
        ...
      },
      ...
    ]
  },
  [
    {
      "funding": "0.0000125",
      "openInterest": "12345.67891",
      "prevDayPx": "67000.0",
      "dayNtlVlm": "5800000000.0",
      "premium": "0.0001",
      "oraclePx": "67380.5",
      "markPx": "67420.5",
      "midPx": "67421.0",
      "impactPxs": ["67419.5", "67421.5"]
    },
    ...
  ]
]
```

`openInterest` — в base asset (количество монет).

### 3.2 meta только

`POST /info` с `{"type": "meta"}` — только universe (метаданные), без current state. Для sync_instruments.

---

## 4. Rate limits

Точно не задокументированы публично; эмпирически:
- ~10 POST/sec на IP без проблем.
- При превышении — HTTP 429.

**Конфиг:**
```python
RATE_LIMIT_CAPACITY = 100
RATE_LIMIT_REFILL_PER_SEC = 10
MAX_CONCURRENT_REQUESTS = 5
```

**Стратегия:** один запрос `metaAndAssetCtxs` за цикл — содержит и instruments, и OI, и mark prices.

---

## 5. Symbol naming

- Pattern: `BASE` (просто base asset, без quote-suffix в universe).
- Examples в universe: `"BTC"`, `"ETH"`, `"SOL"`.
- В UI Hyperliquid показывает `"BTC-USD"` или `"BTC-USDT"`, но API использует только base name.

NB: Hyperliquid использует **USDC** под капотом для расчётов (collateral), но контракты квотируются в "USD". Для нашей системы канонизируем как USDT-M (с явным признаком в `Instrument.raw_metadata.note = "settled_in_usdc_internal"`).

**Это компромисс**: Hyperliquid технически не USDT-M (quote ≠ USDT), но цена deno в USD = USDT для практических целей. Решение зафиксировано: включаем Hyperliquid в систему как «эффективно USDT-M perp».

**Mapping:** `"BTC"` → `canonical_symbol = "BTC"` (1:1).

---

## 6. Filter

В Hyperliquid universe — **только perpetual contracts**. Все они линейные.

```python
def is_active_perp(item: dict) -> bool:
    # Hyperliquid universe всегда содержит активные perp markets
    # Field "isDelisted" — пометка делистинга (если есть)
    return not item.get("isDelisted", False)
```

Поля:
- `name`: e.g. `"BTC"`.
- `szDecimals`: размер lot'а в десятичных знаках.
- `maxLeverage`: max leverage.
- `onlyIsolated`: bool — поддерживает ли cross-margin.

---

## 7. Parser

### 7.1 От `metaAndAssetCtxs`

```python
def parse_meta_and_ctxs(payload: list) -> list[ParsedOI]:
    meta, ctxs = payload
    universe = meta["universe"]
    parsed_list = []
    fetched_at = utcnow()

    for inst_meta, ctx in zip(universe, ctxs):
        if inst_meta.get("isDelisted"):
            continue
        parsed_list.append(ParsedOI(
            native_symbol=inst_meta["name"],
            ts_exchange=fetched_at,                    # Hyperliquid не возвращает per-asset ts
            oi_raw=Decimal(str(ctx["openInterest"])),
            oi_unit_hint="base_asset",
            oi_notional_provided=None,
            price_provided=Decimal(str(ctx["markPx"])),
            raw_extras={
                "oracle_px": ctx["oraclePx"],
                "mid_px": ctx.get("midPx"),
                "funding": ctx.get("funding"),
            },
        ))
    return parsed_list
```

---

## 8. Unit resolution

- `oi_unit_hint = "base_asset"`.
- `oi_coins = oi_raw`.
- `oi_notional_usdt = oi_coins × markPx`, `valuation_status = "good_estimate"`.

---

## 9. Price resolution

- Primary: `markPx` из `metaAndAssetCtxs`.
- Fallback: `oraclePx` (oracle aggregated price).
- Last fallback: `midPx`.

Цена приходит вместе с OI в одном запросе — отдельный price fetch не нужен.

---

## 10. Edge cases

### 10.1 USDC vs USDT

Hyperliquid не использует USDT натурально. Но deno в USD (≈ USDT). Для системы:
- `quote_asset = "USDT"` (символьно).
- `settle_asset = "USDC"` (физически).
- В `Instrument.raw_metadata.note`: `"hyperliquid_usdc_settled_quoted_as_usd"`.

UI может показывать пометку «(USDC)» рядом с Hyperliquid OI для прозрачности.

### 10.2 Universe ordering

Order в `meta.universe` соответствует order в `ctxs` (zip обязателен). Order может меняться при добавлении новых perp.

### 10.3 ts_exchange отсутствует

В payload нет per-asset timestamp. Используем `fetched_at` (server clock), помечаем `ts_exchange_inferred` warning.

### 10.4 markPx может быть `null` для новых markets

Для только что добавленных perp `markPx == null` — пропускаем (idle state).

### 10.5 Delisting

Hyperliquid редко делистит. Пометка `isDelisted` бывает; при появлении — `delisted_at = now()`.

### 10.6 Special tokens

Hyperliquid торгует некоторые pre-launch tokens (например `kPEPE` — 1000-multiplied PEPE). Canonical mapping:
- `kPEPE` → `PEPE` (с `multiplier = 1000`).
- Аналогично Binance pattern с `1000`-prefix.

---

## 11. Test fixtures

```
tests/contract/hyperliquid/fixtures/
├── meta.json                     # only meta type
├── meta_and_asset_ctxs.json      # full snapshot
└── meta_with_delisted.json       # edge case
```

---

## 12. References

- Hyperliquid API: https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/info-endpoint
- Asset Contexts: https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/info-endpoint/perpetuals
- Universe: see "metaAndAssetCtxs" type in docs
