---
description: Прогнать новый/изменённый код против docs/05_DATA_CONTRACTS.md и связанных спек. Валидация типов, имён, инвариантов.
argument-hint: [file_or_dir | --staged | --branch]
---

Цель: $ARGUMENTS

## Инструкции

1. Определить scope:
   - Если `$ARGUMENTS` пустой — взять все изменения в working tree (`git status --short`).
   - `--staged` — взять `git diff --cached --name-only`.
   - `--branch` — взять `git diff main...HEAD --name-only`.
   - Иначе — рассматривать `$ARGUMENTS` как путь к файлу или директории.

2. Прочитать source-of-truth документы:
   - `docs/05_DATA_CONTRACTS.md`
   - `docs/00_DECISIONS_LOG.md` (для точечных инвариантов)
   - `docs/04_TIME_MODEL.md` (для всего, что касается времени)
   - `docs/03_GLOSSARY.md` (для имён)

3. Для каждого затронутого Python-файла проверить:

   **Имена и термины (из 03_GLOSSARY.md / 00_DECISIONS_LOG.md):**
   - `canonical_symbol` — это BASE (например `"BTC"`), не `BTCUSDT`. Любое присвоение `canonical_symbol = "<X>USDT"` — нарушение C1.
   - Поля `oi_coins`, `oi_notional_usdt`, `quote_asset`, `settle_asset`, `market_type`, `contract_type`, `source_kind`, `valuation_status` — используются именно в этих именах.
   - `market_type == 'usdt_perp'`, `contract_type == 'perpetual'`.
   - `valuation_status ∈ {authoritative, good_estimate, low_confidence}`.
   - `source_kind ∈ {snapshot, native_interval, backfill, replay, degraded_fallback}`.

   **Типы данных:**
   - Время — всегда `datetime` с tzinfo (UTC) или `TIMESTAMPTZ` в DDL. Никогда naive.
   - Numeric — `Decimal` в Python; в pydantic `condecimal` или явный `Decimal`. Не `float`.
   - DDL: coins/price → `NUMERIC(38, 18)`; USDT notional → `NUMERIC(38, 8)`.

   **Инварианты:**
   - Δ всегда считается по `oi_coins` (C5). Если видишь `delta_pct = (notional_new - notional_old) / notional_old` — нарушение.
   - Пороги — только относительные (%). Никаких абсолютных Δ.
   - `min_oi_notional_usdt` — это floor-фильтр, не порог.
   - Backfill / replay не создают retro-fire алертов: ищем, что reprocessing path не пишет в `alert_state` / `delivery_queue`.
   - Smart cooldown: re-fire допустим при Δ ≥ 1.5× предыдущего.

4. Для DDL / миграций — проверить:
   - Hypertable на `oi_samples` партиционирован по `ts_exchange`.
   - Compression `segmentby = (exchange, canonical_symbol)`.
   - Retention 90 дней.
   - Reversibility (наличие `downgrade()`).

5. Для API / SSE / Telegram payloads — проверить, что поля соответствуют контрактам в `05_DATA_CONTRACTS.md` и `10_DELIVERY_LAYER.md`.

## Формат отчёта

```
## Verify-Spec Report

### Scope
- <files>

### Найденные нарушения
1. [CRITICAL] <file>:<line> — <описание> (нарушает <C/Q/F-id или раздел docs>)
2. [HIGH] ...
3. [MEDIUM] ...

### Соответствие контрактам
- 05_DATA_CONTRACTS.md: SAME / DIFFERENT
- 04_TIME_MODEL.md: SAME / DIFFERENT
- 00_DECISIONS_LOG.md (C1, C5, C9, C11, C12): SAME / DIFFERENT

### Verdict
PASS / FAIL

### Рекомендации
- ...
```

Если найдены CRITICAL — настоятельно предложить вызвать `decisions-guardian` для финальной проверки перед merge.
