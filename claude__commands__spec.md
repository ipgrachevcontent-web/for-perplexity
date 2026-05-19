---
description: Найти и зачитать релевантные документы из docs/ в основной контекст. Использовать в начале любой задачи, чтобы загрузить нужные спеки.
argument-hint: [topic | filename | keyword]
---

Тема: $ARGUMENTS

## Инструкции

1. Если `$ARGUMENTS` пустой — прочитать `docs/README.md` и `docs/00_DECISIONS_LOG.md`, вывести оглавление всех файлов из `docs/` с одной строкой описания каждого.

2. Если `$ARGUMENTS` совпадает с именем файла (полностью или частично) — прочитать файл целиком.

3. Если `$ARGUMENTS` — это тема/keyword:
   - Запусти `rg -l --no-ignore "<keyword>" /var/www/oi-tracker/docs/` чтобы найти релевантные файлы.
   - Прочитай найденные файлы целиком.
   - Если найдено больше 4 файлов — прочитай только 4 самых релевантных (сначала `00_DECISIONS_LOG.md`, потом основные спеки, потом per-exchange).

4. После чтения — кратко резюмируй (3–5 пунктов) что только что было загружено и какие решения релевантны теме.

5. Если тема касается архитектурного вопроса — обязательно упомянуть, какие решения C/Q/F из `00_DECISIONS_LOG.md` применимы.

## Карта тем → файлы

- alert / state machine / cooldown → `09_ALERT_ENGINE.md`
- schema / DDL / table / column → `05_DATA_CONTRACTS.md`, `08_TIME_SERIES_STORAGE.md`
- time / freshness / ts_exchange → `04_TIME_MODEL.md`
- normalize / mapping / canonical → `07_NORMALIZER.md`, `06_INSTRUMENT_REGISTRY.md`
- exchange / connector / adapter → `11_EXCHANGE_ADAPTERS/_BASE_CONNECTOR.md` + `<exchange>.md`
- delivery / SSE / Telegram → `10_DELIVERY_LAYER.md`
- metrics / SLO / Prometheus / Grafana → `12_OBSERVABILITY_SLO.md`
- ops / systemd / runbook → `13_OPERATIONS.md`
- test / coverage / fixture → `14_TEST_STRATEGY.md`
- glossary / term / oi_coins / valuation_status → `03_GLOSSARY.md`
- product / NFR / SLO / scope → `01_PRODUCT_SPEC.md`
- architecture / component / flow → `02_ARCHITECTURE.md`
