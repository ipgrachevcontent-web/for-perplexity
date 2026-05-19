# OI Tracker — База знаний проекта

> Полная техническая документация проекта `oi-tracker`. На этой базе ведётся вся разработка.
> При конфликте между документами и первичными файлами (`/var/www/oi-tracker/*.md`) — приоритет у документов в этой папке.

---

## Как читать

Документы пронумерованы в порядке зависимостей: каждый последующий опирается на предыдущие.

### Уровень 1 — Foundation (что и почему)

| # | Файл | Что внутри |
|---|---|---|
| 00 | [DECISIONS_LOG](00_DECISIONS_LOG.md) | **Источник истины** для всех архитектурных решений. C1–C17, Q1–Q8, F2–F18. |
| 01 | [PRODUCT_SPEC](01_PRODUCT_SPEC.md) | Mission, scope, non-goals, FR/NFR, KPI, риски. |
| 02 | [ARCHITECTURE](02_ARCHITECTURE.md) | Компоненты, потоки данных, стек, deployment topology. |
| 03 | [GLOSSARY](03_GLOSSARY.md) | Все термины проекта в одном месте. |
| 04 | [TIME_MODEL](04_TIME_MODEL.md) | Три временные оси (`ts_exchange / ts_ingested / ts_processed`), freshness формулы, late-arriving data policy. |

### Уровень 2 — Core Domain (как устроены данные)

| # | Файл | Что внутри |
|---|---|---|
| 05 | [DATA_CONTRACTS](05_DATA_CONTRACTS.md) | Pydantic-модели и DDL всех событий: RawExchangeEvent, NormalizedOISample, AlertEvent, AlertRule, AlertState, DeliveryRequest, Instrument, Settings, ConnectorErrorEvent, NormalizationError. |
| 06 | [INSTRUMENT_REGISTRY](06_INSTRUMENT_REGISTRY.md) | Master data об инструментах. Sync каждые 5 мин. Versioning через `instrument_versions` для replay. |
| 07 | [NORMALIZER](07_NORMALIZER.md) | 5-стадийный pipeline: parser → instrument resolver → unit resolver → price attachment → quality assessor. Per-exchange правила. |
| 08 | [TIME_SERIES_STORAGE](08_TIME_SERIES_STORAGE.md) | Полный DDL: hypertable `oi_samples`, CA `oi_5m/15m/30m`, retention 90 days, compression after 7 days, reprocessing процедуры. |
| 09 | [ALERT_ENGINE](09_ALERT_ENGINE.md) | 4-state machine, 4 типа сигналов (threshold, confirmed, divergence, exchange-health; consensus sunset per `00 F40`), smart cooldown 1.5×, no-retro-fire. |

### Уровень 3 — Edges (как связь с внешним миром)

| # | Файл | Что внутри |
|---|---|---|
| 10 | [DELIVERY_LAYER](10_DELIVERY_LAYER.md) | Telegram sender (aiogram), FastAPI REST + SSE, 4 страницы UI, nginx config. |
| 11 | [EXCHANGE_ADAPTERS](11_EXCHANGE_ADAPTERS/) | Базовый `_BASE_CONNECTOR.md` + 12 файлов per-exchange. |

#### Биржи (`11_EXCHANGE_ADAPTERS/`)

| Биржа | Source kind | Файл |
|---|---|---|
| Binance | snapshot | [binance.md](11_EXCHANGE_ADAPTERS/binance.md) |
| Bybit | **native interval** | [bybit.md](11_EXCHANGE_ADAPTERS/bybit.md) |
| OKX | snapshot (bulk) | [okx.md](11_EXCHANGE_ADAPTERS/okx.md) |
| Bitget | snapshot (bulk via tickers) | [bitget.md](11_EXCHANGE_ADAPTERS/bitget.md) |
| Gate.io | snapshot (bulk via contracts) | [gateio.md](11_EXCHANGE_ADAPTERS/gateio.md) |
| MEXC | snapshot (bulk via tickers) | [mexc.md](11_EXCHANGE_ADAPTERS/mexc.md) |
| KuCoin | **native interval** | [kucoin.md](11_EXCHANGE_ADAPTERS/kucoin.md) |
| HTX | snapshot (bulk OI) | [htx.md](11_EXCHANGE_ADAPTERS/htx.md) |
| Hyperliquid | snapshot (info endpoint) | [hyperliquid.md](11_EXCHANGE_ADAPTERS/hyperliquid.md) |
| Aster | snapshot (binance-like) | [aster.md](11_EXCHANGE_ADAPTERS/aster.md) |
| Bitmart | snapshot (bulk via /contract/public/details) | [bitmart.md](11_EXCHANGE_ADAPTERS/bitmart.md) |
| XT | snapshot (bulk one-shot) | [xt.md](11_EXCHANGE_ADAPTERS/xt.md) |

### Уровень 4 — Operations (как эксплуатировать)

| # | Файл | Что внутри |
|---|---|---|
| 12 | [OBSERVABILITY_SLO](12_OBSERVABILITY_SLO.md) | Prometheus метрики, Grafana дашборды, Loki, Alertmanager, SLO P95/P99 latency, runbook references. |
| 13 | [OPERATIONS](13_OPERATIONS.md) | Bare-metal deployment, systemd units, runbooks (биржа упала, БД упала, TG stuck, disk full, ...), reprocessing процедуры. |
| 14 | [TEST_STRATEGY](14_TEST_STRATEGY.md) | Test pyramid, contract tests с реальными payload-фикстурами, integration с testcontainers, e2e, coverage targets. |

### Уровень 5 — Deployment & Roadmap (как развернуть и в какой последовательности)

| # | Файл | Что внутри |
|---|---|---|
| 15 | [DEPLOYMENT_INFRA](15_DEPLOYMENT_INFRA.md) | Inventory соседних проектов, домен `oi-tracker.robot-detector.ru`, TLS-стратегия (отдельный certbot), noindex (3-layer), изоляция (UDS, dedicated PG db+role, OS user, paths), TimescaleDB install procedure с maintenance window. |
| 16 | [ROADMAP](16_ROADMAP.md) | Milestones M1–M5 с DoD по каждому. Phase 0 = TBD до отдельного решения. |
| 17 | [TOOLING](17_TOOLING.md) | `pyproject.toml`, `package.json`, pre-commit, CI minimal (GitHub Actions). |
| 18 | [DESIGN_SYSTEM](18_DESIGN_SYSTEM.md) | Дизайн-система UI: принципы, цветовые токены (OKLCH), типографика, spacing, motion, компоненты, per-page UX, a11y. |

---

## Quick navigation

### По задаче

| Хочу... | Читать |
|---|---|
| Узнать, что система делает | `01_PRODUCT_SPEC.md` |
| Понять, почему именно так | `00_DECISIONS_LOG.md` |
| Реализовать новый коннектор | `11_EXCHANGE_ADAPTERS/_BASE_CONNECTOR.md` |
| Изменить формат алерта в TG | `10_DELIVERY_LAYER.md §3` |
| Добавить новое правило алерта | `09_ALERT_ENGINE.md §5, §7` |
| Добавить новую метрику в нормализатор | `07_NORMALIZER.md` + `12_OBSERVABILITY_SLO.md §3.3` |
| Сделать миграцию БД | `08_TIME_SERIES_STORAGE.md §2` + `05_DATA_CONTRACTS.md` |
| Развернуть на новом сервере | `15_DEPLOYMENT_INFRA.md §7` (checklist) → `13_OPERATIONS.md §2` (commands) |
| Понять, что в первом PR / какие milestones | `16_ROADMAP.md` |
| Настроить pyproject.toml / pre-commit / CI | `17_TOOLING.md` |
| Узнать, что выносится в UI Settings | `10_DELIVERY_LAYER.md §4.8` (категории A/B/C) |
| Сделать UI-компонент в едином стиле | `18_DESIGN_SYSTEM.md` (токены, типографика, motion, паттерны) |
| Понять, почему алерт не сработал | `09_ALERT_ENGINE.md §10` (audit log) + `12_OBSERVABILITY_SLO.md §9` |
| Сделать backfill / replay | `08_TIME_SERIES_STORAGE.md §6` + `13_OPERATIONS.md §6` |
| Написать тест | `14_TEST_STRATEGY.md` |

### По термину

| Термин | Определение | Использование |
|---|---|---|
| `ts_exchange` | `04_TIME_MODEL.md §2` | partitioning axis, источник истины |
| `canonical_symbol` | `00_DECISIONS_LOG.md C1`, `03_GLOSSARY.md` | "BTC" формат |
| `source_kind` | `03_GLOSSARY.md`, `00_DECISIONS_LOG.md F11` | snapshot / native_interval / ... |
| `valuation_status` | `03_GLOSSARY.md`, `00_DECISIONS_LOG.md F10` | authoritative / good_estimate / low_confidence |
| `normalization_version` | `07_NORMALIZER.md §9` | для replay при изменении формул |
| smart cooldown | `09_ALERT_ENGINE.md §6`, `00_DECISIONS_LOG.md Q1, F7` | re-fire при 1.5× |
| 4-state machine | `09_ALERT_ENGINE.md §3` | idle / armed / fired / cooldown |
| min_history | `00_DECISIONS_LOG.md F8`, `09_ALERT_ENGINE.md §4` | `max(30, 2 × window)` |
| min_oi_notional | `00_DECISIONS_LOG.md C4 + F5` | floor-фильтр default 5M USDT |

---

## Принципы документации

1. **Source of truth — единственный.** При конфликте `00_DECISIONS_LOG.md` побеждает.
2. **Ссылки крестом.** Каждый документ ссылается на смежные. Обнаружили несинхронность — фикси.
3. **Изменения — через PR.** Любое изменение этих документов = ревью.
4. **Версионируем major/minor.** При значимом изменении scope/контракта — bump в шапке файла.
5. **Не дублируем.** Если что-то описано в `00_DECISIONS_LOG.md`, в других файлах — только ссылка.
6. **Не теоретизируем.** Каждое решение в `00_DECISIONS_LOG.md` имеет конкретное обоснование.

---

## Версия документации

- **v1.0** (2026-04-30) — первичный релиз. Все 27 файлов согласованы заказчиком.
- **v1.1** (2026-04-30) — добавлены `15_DEPLOYMENT_INFRA.md`, `16_ROADMAP.md`, `17_TOOLING.md`. Записи `F12–F17` в `00_DECISIONS_LOG.md`. Расширен UI Settings scope в `10_DELIVERY_LAYER.md §4.8 + §6.1`. Добавлены backups RPO/RTO targets и security hardening rules в `13_OPERATIONS.md §9, §10.0`. Добавлен UX-сценарий single-user в `01_PRODUCT_SPEC.md §3.3`.
- **v1.2** (2026-05-01) — добавлен `18_DESIGN_SYSTEM.md` (дизайн-система UI). Кросс-ссылки в `10_DELIVERY_LAYER.md §6` и `16_ROADMAP.md M4`.
- **v1.3** (2026-05-01) — pre-development readiness PR:
  - `08_TIME_SERIES_STORAGE.md §13` — Alembic + TimescaleDB migrations template (закрывает основной блокер B1 для M1).
  - `02_ARCHITECTURE.md §2.1.1, §5.4, §5.5` — ADR по storage access, scheduler concurrency, backpressure.
  - `02_ARCHITECTURE.md §5.1, §6.1, §6.2`, `13_OPERATIONS.md §5.5, §7.1, §8`, `12_OBSERVABILITY_SLO.md §8.1` — выровнены под `F13/F14` (UDS вместо `:8000`, `frontend-dist/`, удалено упоминание Basic Auth, watchdog-based liveness).
  - `16_ROADMAP.md` — Phase 0 закрыт (status `Decided`), добавлен раздел `Pre-milestone prerequisites` для серых зон.
  - `00_DECISIONS_LOG.md F18` — зафиксированы default thresholds (5%/8%/12%) для 6 base alert rules. Версия decisions-log → 1.2.
- **v1.4** (2026-05-01) — Phase 0.A execution findings (F-DOC-1..4) — docs hygiene без policy changes:
  - `13_OPERATIONS.md §2.1.2`, `15_DEPLOYMENT_INFRA.md §6.1` — `apt-key` (deprecated на Ubuntu ≥22.04) заменён на `signed-by` + `/etc/apt/keyrings/timescaledb.gpg`.
  - `13_OPERATIONS.md §2.1.3`, `15_DEPLOYMENT_INFRA.md §4.2` — добавлен `REVOKE CONNECT ... FROM PUBLIC` для соседних БД (literal `REVOKE FROM <role>` не закрывает CONNECT через PUBLIC inheritance — расходилось с `13 §10.0` security audit).
  - `15_DEPLOYMENT_INFRA.md §5` — `http2 on;` (требует nginx ≥1.25.1) заменён на legacy `listen 443 ssl http2;` (host nginx 1.24).
  - `15_DEPLOYMENT_INFRA.md §4.3` — добавлена строка таблицы для env-директории (owner `oi-tracker:oi-tracker`, иначе user не может войти).
- Дальнейшие версии — при значимых изменениях scope или архитектуры.

---

## Связь с исходными файлами

В корне `/var/www/oi-tracker/` лежат **первичные** черновики ТЗ:
- `OI_TRACKER_SPEC.md`, `SPEC.md`, `NORMALIZER.md`, `TIME_SERIES_STORAGE.md`, `ALERT_ENGINE.md`, `ANALYZE.md`.

Эти файлы — исторический контекст. Они **не являются** актуальной спецификацией. Все противоречия и недоопределённости из них зафиксированы и закрыты в `00_DECISIONS_LOG.md`.

Решения, отходящие от исходного ТЗ, явно помечены в `00_DECISIONS_LOG.md` как «Отход от исходного ТЗ» с обоснованием.
