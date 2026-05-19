# Implementation Plan: oi-tracker — End-to-End Development from Zero to Production

> Документ синтезирует карту разработки v1 проекта `oi-tracker` (single-user OI-трекер 12 USDT-M perp бирж, web + Telegram, bare-metal Python 3.12 + FastAPI + PG/TimescaleDB + React) на основе финализированной документации в `/var/www/oi-tracker/docs/` (00–18) и `CLAUDE.md`.
> Это не пересказ `16_ROADMAP.md`, а его углубление до уровня PR-tasks с оценкой по часам и идентификацией критического пути.
>
> Версия плана: 1.0 (2026-05-01).

---

## 1. Executive summary

### 1.1 Состояние документации

Документация v1.3 — **финализирована**. Источник истины:

- `/var/www/oi-tracker/docs/00_DECISIONS_LOG.md` — C1–C17 + Q1–Q8 + F2–F18 (39 решений).
- `/var/www/oi-tracker/docs/01_PRODUCT_SPEC.md` — FR-1..10, NFR-1..8, KPI, full Phase A–E UX-сценарий.
- `/var/www/oi-tracker/docs/02_ARCHITECTURE.md` — компоненты, стек, pre-M1 ADR storage strategy (§2.1.1), pre-M2 ADR concurrency (§5.4) и backpressure (§5.5).
- `/var/www/oi-tracker/docs/03_GLOSSARY.md`, `04_TIME_MODEL.md`.
- `/var/www/oi-tracker/docs/05_DATA_CONTRACTS.md` — 14 pydantic-контрактов с DDL.
- `/var/www/oi-tracker/docs/06_INSTRUMENT_REGISTRY.md`, `07_NORMALIZER.md` — pipeline.
- `/var/www/oi-tracker/docs/08_TIME_SERIES_STORAGE.md` — DDL + §13 готовые Alembic-шаблоны для hypertable / CA / compression / retention с idempotency и down-cycle.
- `/var/www/oi-tracker/docs/09_ALERT_ENGINE.md` — state machine, smart cooldown, 5 типов сигналов, default seed.
- `/var/www/oi-tracker/docs/10_DELIVERY_LAYER.md` — TG sender, REST endpoints, SSE, full UI Settings (категории A/B/C, rate-limit, audit).
- `/var/www/oi-tracker/docs/11_EXCHANGE_ADAPTERS/_BASE_CONNECTOR.md` + 12 файлов бирж.
- `/var/www/oi-tracker/docs/12_OBSERVABILITY_SLO.md` — все метрики §3.1–3.7, 5 Grafana-дашбордов, Alertmanager rules, SLO formulae.
- `/var/www/oi-tracker/docs/13_OPERATIONS.md` — bare-metal install, systemd-юниты, 7 runbooks §5.1–5.7, deploy workflow §7, backups skeleton §9.
- `/var/www/oi-tracker/docs/14_TEST_STRATEGY.md` — пирамида, coverage targets, contract-тесты с фикстурами.
- `/var/www/oi-tracker/docs/15_DEPLOYMENT_INFRA.md` — F12–F17 политика + полный nginx-config + §7 deployment checklist.
- `/var/www/oi-tracker/docs/16_ROADMAP.md` — Phase 0 (Decided 2026-05-01), M1–M5, pre-milestone prerequisites таблица.
- `/var/www/oi-tracker/docs/17_TOOLING.md` — pyproject skeleton, package.json skeleton, .pre-commit-config.yaml, CI minimal.
- `/var/www/oi-tracker/docs/18_DESIGN_SYSTEM.md` — OKLCH-токены, Inter+JBM, §13 acceptance for M4.

Ничего блокирующего в документации не найдено; ADR-границы (M1 storage strategy, M2 concurrency/backpressure/pool-sizes) уже зафиксированы в `02 §2.1.1, §5.4, §5.5`.

### 1.2 Что значит «production-ready» для этого проекта

По `01_PRODUCT_SPEC §3, §6, §7` + `CLAUDE.md §4.1`:

- 12 коннекторов USDT-M perp работают в flat 60s polling, изоляция per-exchange (`F3`).
- Normalizer pipeline (5 стадий) с `valuation_status` и `source_kind` для каждой точки.
- TimescaleDB hypertable `oi_samples` + 4 CA (`oi_5m/15m/30m/oi_quality_5m`) + compression after 7d + retention 90d.
- Alert engine: 4-state machine, smart cooldown, 5 типов сигналов, 6 default-rows F18 (5%/8%/12% × up/down).
- Delivery: TG sender (sender-only aiogram 3.x, retry policy, dry-run toggle), 5 templates.
- Web UI: 4 страницы (Dashboard, Symbol, Alerts, Settings) + SSE no-replay + full UI Settings (категории A+B `F15`).
- Bare-metal на shared-машине: 4 systemd-юнита, UDS `/run/oi-tracker/api.sock`, nginx с three-layer noindex `F13`, Let's Encrypt `oi-tracker.robot-detector.ru` `F12/F16`.
- Полный observability stack: Prometheus + Grafana (5 дашбордов) + Loki + Alertmanager.
- Coverage: ≥ 80% общий, ≥ 90% alert engine / normalizer / exchanges.
- SLO: e2e P95 ≤ 90s, P99 ≤ 180s; symbol coverage ≥ 95%.
- Безопасность: `verify=False` запрещён, f-string в SQL запрещён, env `0600`, role isolation, REVOKE на соседние БД.
- Все runbooks `13 §5.1–5.7` пройдены drill'ом; security audit PASS.

### 1.3 Общая структура

```
[Phase 0 Skeleton (Decided)]
         │
         ▼
   pre-M1 ADR (закрыт: storage strategy 02 §2.1.1)
         │
         ▼
       [M1 Foundation]      ← +CA + бизнес-метрики + cleanup + полный normalizer
         │
         ▼
   pre-M2 ADR (закрыт: TaskGroup 02 §5.4, backpressure 02 §5.5,
               pool sizes — открыт, нужно зафиксировать)
         │
         ▼
       [M2 12 connectors + registry sync + circuit breaker + replay-tool]
         │
         ▼
   pre-M3 ADR (открыт: aiogram throttle, settings hot-reload)
         │
         ▼
       [M3 Alert engine + TG delivery]
         │
         ▼
   pre-M4 ADR (открыт: SSE no-replay disclaimer, NOTIFY 8KB policy)
         │
         ▼
       [M4 Web UI]                ← включая full §13 design-system acceptance
         │
         ▼
   pre-M5 ADR (открыт: trace_id propagation)
         │
         ▼
       [M5 Observability complete + 7 runbook drills + hardening + backups skeleton]
```

### 1.4 Оценка общего объёма

Реалистичная оценка для одного senior dev в full-time режиме:

| Phase | Нижняя оценка | Верхняя оценка | Средняя |
|---|---|---|---|
| Phase 0 | 28h | 44h | **36h** |
| M1 | 24h | 36h | **30h** |
| M2 | 80h | 120h | **100h** |
| M3 | 56h | 84h | **70h** |
| M4 | 96h | 144h | **120h** |
| M5 | 48h | 72h | **60h** |
| Buffer (15–20%) | — | — | **~70h** |
| **Итого** | **332h** | **500h** | **~486h** |

В часах full-time (40h/неделя): **~12 недель / ~3 месяца**.
В режиме part-time (20h/неделя): **~24 недели / ~6 месяцев**.

Эти оценки предполагают, что инфраструктура хост-машины (TimescaleDB установка через restart кластера PG `F17`, DNS A-запись для subdomain `F12`) выполняется в рамках Phase 0 в согласованном maintenance-window.

---

## 2. Phase 0 — Skeleton (детализация)

> Phase 0 уже **Decided 2026-05-01** в `16_ROADMAP §Phase 0`. Здесь — разворот в последовательные задачи. Каждая задача в формате: `(Файл-источник docs, артефакт, оценка часов, риски, тесты)`.

### Phase 0.A — Infrastructure (host operations)

| # | Задача | Источник | Артефакт | Часы | Риск |
|---|---|---|---|---|---|
| 0.A.1 | DNS A-запись `oi-tracker.robot-detector.ru` → host IP | `15 §2.1, §7` | DNS resolves | 0.5h (внешняя задача владельца) | Low |
| 0.A.2 | Уведомление владельцев соседей `detector`, `grach-ege` о maintenance window для restart PG | `15 §6.1 step 1` | согласованное окно | 0.5h | Low |
| 0.A.3 | `pg_dump` соседних БД (`detector`, `grachege`, `grachege_shadow`) — страховка | `15 §6.1 step 2` | `.dump.gz` файлы | 0.5h | Med (большой объём) |
| 0.A.4 | Установка `timescaledb-2-postgresql-16` через apt + repo packagecloud | `15 §6.1 step 3` | пакет установлен | 0.5h | Low |
| 0.A.5 | Backup `postgresql.conf` + `timescaledb-tune --quiet --yes` | `15 §6.1 step 4` | tune применён | 0.5h | Med (меняет shared_preload_libraries) |
| 0.A.6 | `systemctl restart postgresql@16-main` + smoke test соседей | `15 §6.1 step 5–7` | соседи 200 OK | 1h | **High** — задевает `detector` + `grach-ege` |
| 0.A.7 | `CREATE DATABASE oi_tracker`, `CREATE ROLE oi_tracker`, `REVOKE ALL` на соседние БД, `CREATE EXTENSION timescaledb` | `15 §4.2` + `15 §6.3` | DB готова | 0.5h | Low |
| 0.A.8 | OS user `oi-tracker` (system, nologin) + директории `/opt/oi-tracker`, `/etc/oi-tracker`, `/var/log/oi-tracker`, `/var/www/oi-tracker/frontend-dist` с правильными chmod/chown | `15 §4.3`, `13 §2.1.4` | директории + user | 0.5h | Low |
| 0.A.9 | env-файл `/etc/oi-tracker/oi-tracker.env` `chmod 0600`, owner `oi-tracker` (DATABASE_URL, TELEGRAM_BOT_TOKEN placeholder, LOG_LEVEL) | `13 §2.1.7`, `F14` | env-файл | 0.5h | Med (секрет в файле) |
| 0.A.10 | nginx site-config `/etc/nginx/sites-available/oi-tracker` (UDS upstream, X-Robots-Tag, security headers, SSE buffering off, /api + /sse locations) — pre-cert версия | `15 §5`, `13 §2.1.11` | nginx site, `nginx -t` PASS | 1h | Med |
| 0.A.11 | Placeholder `index.html` + `robots.txt` в `/var/www/oi-tracker/frontend-dist/` | `15 §3.2, §3.3, §7` | static files | 0.5h | Low |
| 0.A.12 | Let's Encrypt cert `certbot --nginx -d oi-tracker.robot-detector.ru` | `15 §2.2`, `F16` | cert выпущен | 0.5h | Med (DNS должен резолвиться) |
| 0.A.13 | Smoke-tests: `curl -I` на subdomain → 200 + `X-Robots-Tag` headers; `/robots.txt` → `Disallow: /`; соседи `robot-detector.ru`, `grach-ege` всё ещё 200 | `15 §7 checklist` | все галочки `15 §7` пройдены | 0.5h | Low |
| 0.A.14 | logrotate `/etc/logrotate.d/oi-tracker` (daily, rotate 14, SIGHUP в сервисы) | `13 §2.1.10`, `15 §4.5` | logrotate config | 0.5h | Low |

**Phase 0.A итого:** ~7.5h.

### Phase 0.B — Code skeleton

| # | Задача | Источник | Артефакт | Часы | Риск |
|---|---|---|---|---|---|
| 0.B.1 | `pyproject.toml` (uv-managed, deps + dev extras + ruff + mypy strict + pytest config + coverage fail_under=80) | `17 §1.2` | pyproject.toml | 1h | Low |
| 0.B.2 | Структура `backend/src/oi_tracker/` (каталоги по `02 §4` + `17 §1.3`): `config.py`, `db/`, `exchanges/`, `normalizer/`, `alert_engine/`, `tg_sender/`, `api/`, `scheduler/`, `observability/`, `tools/`, `domain/` | `02 §4`, `17 §1.3` | пустые `__init__.py` | 0.5h | Low |
| 0.B.3 | `app/config.py` (pydantic-settings) — читает env: DATABASE_URL, TELEGRAM_BOT_TOKEN, LOG_LEVEL, APP_ENV | `13 §2.1.7` | Settings class | 1h | Low |
| 0.B.4 | `app/observability/logging_config.py` (structlog JSON, ts/level/service/event поля), вывод в `/var/log/oi-tracker/{service}.log` + journald | `12 §5.1`, `15 §4.5` | structlog setup | 1.5h | Low |
| 0.B.5 | `app/domain/events.py` — pydantic-контракты: `RawExchangeEvent`, `NormalizedOISample`, `OIBar`, `Instrument`, `ConnectorErrorEvent`, `SourceKind`, `ValuationStatus`, `PriceSource` enums | `05 §3, §4, §5, §11, §13` | модули с frozen=True | 2h | Low |
| 0.B.6 | `.pre-commit-config.yaml` (ruff format/check, mypy, gitleaks, pytest -m "not slow and not integration and not contract") | `17 §3` | pre-commit installed | 1h | Low |
| 0.B.7 | CI minimal `.github/workflows/ci.yml` (backend service postgres timescaledb, ruff, mypy, alembic upgrade head, pytest --cov-fail-under=80; secrets job gitleaks) | `17 §4` | CI yaml | 1h | Low |

**Phase 0.B итого:** ~8h.

### Phase 0.C — DB layer (Alembic migration #1)

| # | Задача | Источник | Артефакт | Часы | Риск |
|---|---|---|---|---|---|
| 0.C.1 | `alembic.ini` + `alembic/env.py` (async-aware через asyncpg sync wrapper для миграций) | `08 §13.1, §13.2` | alembic init | 1h | Low |
| 0.C.2 | Миграция `0001_init.py` — extension check + базовые non-hypertable таблицы (`instruments`, `connector_errors`, `settings`, `alert_rules`, `alert_state`, `alert_events`, `delivery_queue`, `settings_audit_log`, `normalization_errors`, `raw_exchange_events` hypertable) | `05` (вся секция DDL), `13.2 _ensure_timescaledb()`, `13.5 checklist` | revision 0001 | 2.5h | Med |
| 0.C.3 | Миграция `0002_oi_samples_hypertable.py` — `oi_samples` table + `create_hypertable` + compression (segmentby + orderby) + retention 90d + 3 индекса (idx_oi_samples_canonical_time / exchange_time / recent_native), полный `downgrade()` | `08 §13.3` | revision 0002 | 1.5h | Med |
| 0.C.4 | Migration test `tests/integration/test_migrations.py::test_upgrade_downgrade_cycle` — двойной upgrade (idempotency) + downgrade -1 + upgrade +1 на чистой test-БД | `08 §13.5`, `14 §5.2` | passing test | 1.5h | Med |
| 0.C.5 | `app/db/session.py` — `asyncpg.create_pool(min_size, max_size)` factory + dependency для FastAPI; **без AsyncSession ORM** (см. ADR `02 §2.1.1`) | `02 §2.1.1` | pool factory | 1h | Low |
| 0.C.6 | `app/db/repositories/oi_samples.py` — `bulk_insert(samples)` через `pool.executemany(INSERT_SAMPLE_SQL)` | `02 §2.1.1`, `08 §4.1, §4.2` | repository module | 1.5h | Med |

**Phase 0.C итого:** ~9h.

### Phase 0.D — Connector + scheduler skeleton (Binance only)

| # | Задача | Источник | Артефакт | Часы | Риск |
|---|---|---|---|---|---|
| 0.D.1 | `app/exchanges/_base.py` — `BaseConnector` abstract class (sync_instruments, fetch_snapshot, fetch_prices абстрактные; healthcheck, execute_cycle, get_rate_limit_policy конкретные) | `11/_BASE_CONNECTOR §2, §3` | base.py | 2h | Med |
| 0.D.2 | `app/exchanges/_rate_limiter.py` — token bucket per-exchange (общий для последующих 11 коннекторов) | `12 §3.1 oi_collector_rate_limit_remaining` | rate limiter | 1h | Low |
| 0.D.3 | `app/exchanges/_circuit_breaker.py` — N=5 fails → 30s pause (упрощённо, full в M2) | `00 F3`, `02 §3.2` | breaker stub | 1h | Low |
| 0.D.4 | `app/exchanges/binance.py` — реализация только `sync_instruments` + `fetch_snapshot` (без `fetch_range`); парсер `parse_open_interest` | `docs/11_EXCHANGE_ADAPTERS/binance.md` | binance connector | 3h | Med (наиболее задокументированный API; см. также contract test ниже) |
| 0.D.5 | `app/normalizer/pipeline.py` — skeleton 5 stages, **без price attachment / quality assessor** (Phase 0): только parser → instrument resolver → unit resolver (примитивный) → конструирует `NormalizedOISample` с `valuation_status="good_estimate"` placeholder | `07 §2`, Phase 0 scope `16` | pipeline | 2.5h | Med |
| 0.D.6 | `app/scheduler/loop.py` — async loop, выровненный по минуте (`next_minute_aligned_at`), один tick → execute_cycle on Binance connector → bulk_insert | `02 §5.4` (упрощённый, full TaskGroup в M2), `00 C2` | scheduler loop | 2h | Med |
| 0.D.7 | `app/scheduler_main.py` — entrypoint: load config → create pool → instantiate connector → run loop, signal handling (SIGTERM grace shutdown) | `02 §4`, `13 §3.1` | entrypoint | 1h | Low |
| 0.D.8 | `app/api/main.py` — FastAPI app skeleton: `/api/v1/health` (DB ping + scheduler heartbeat), `/metrics` (prometheus_client base, без custom metrics) | `10 §4.11`, `12 §3.7` | minimal API | 1.5h | Low |

**Phase 0.D итого:** ~14h.

### Phase 0.E — systemd + tests + DoD verification

| # | Задача | Источник | Артефакт | Часы | Риск |
|---|---|---|---|---|---|
| 0.E.1 | `deploy/systemd/oi-tracker-scheduler.service` (Type=notify, WatchdogSec=120, MemoryMax=4G, CPUWeight=150, ProtectSystem=strict) | `13 §3.1` | unit-файл | 0.5h | Low |
| 0.E.2 | `deploy/systemd/oi-tracker-api.service` (Type=simple, RuntimeDirectory=oi-tracker, `--uds /run/oi-tracker/api.sock`, ExecStartPost chgrp www-data + chmod 0660 на socket, MemoryMax=2G) | `13 §3.2`, `F14` | unit-файл | 0.5h | Med (UDS socket permissions) |
| 0.E.3 | Установка юнитов на хост, enable+start, smoke test через UDS: `sudo -u oi-tracker curl --unix-socket /run/oi-tracker/api.sock http://localhost/api/v1/health` → 200 | `13 §2.1.10`, `13 §8` | сервисы запущены | 1h | Med |
| 0.E.4 | Unit tests `tests/unit/exchanges/test_binance_parser.py` — mapping native_symbol → canonical_symbol, parse_open_interest на synthetic payload, edge cases (missing field, type changed) | `14 §3.1, §4.5` | passing tests | 2h | Low |
| 0.E.5 | Contract test `tests/contract/binance/` — capture одного live payload в `fixtures/exchange_info.json` + `fixtures/open_interest_btcusdt.json`, тест `test_binance_parser_open_interest_happy_path` | `14 §4.2, §4.3` | passing contract test + fixtures | 2h | Med |
| 0.E.6 | Integration test `tests/integration/test_scheduler_one_cycle.py` — testcontainers PG+TS, run alembic upgrade head, run scheduler `execute_cycle()` once против Binance mock (respx), assert `SELECT count(*) FROM oi_samples > 0` | `14 §5.2, §5.3` | passing integration | 3h | Med |
| 0.E.7 | DoD verification per `16 §Phase 0 DoD`: 30-минутный run scheduler → ≥ 25 циклов → 83% time-coverage; oi_samples заполнен на всех активных USDT-perp Binance-символах; `curl https://oi-tracker.robot-detector.ru/` → 200 + X-Robots-Tag; alembic upgrade/downgrade на чистой БД; mypy/ruff clean; соседи 200 OK после reload nginx и restart PG | `16 §Phase 0 DoD`, `CLAUDE.md §4.1` | DoD checklist passed | 1h | Low |

**Phase 0.E итого:** ~10h.

**Phase 0 итого: ~36h** (4–5 рабочих дней full-time, ~9 дней при 4ч/день).

---

## 3. Pre-M1 prerequisites (закрыто)

> `16 §Pre-milestone prerequisites` фиксирует ровно один пункт: **storage strategy** (asyncpg для bulk insert, SQLAlchemy Core для не-hot мутаций, **без ORM session**).

**Статус:** **Decided** в `02 §2.1.1` (полная ADR-таблица hot-path / non-hot / read-only / migrations + явный список «что НЕ делаем»).

**Действие:** в Phase 0.C.5 уже опираемся на эту ADR. Никаких дополнительных ADR-PR перед стартом M1 не требуется.

**Verification перед стартом M1:**
- [ ] В коде нет `from sqlalchemy.orm import Session` или `AsyncSession` (grep CI step).
- [ ] Hot-path repositories (`oi_samples.bulk_insert`) используют `asyncpg.Pool` напрямую.
- [ ] Не-hot мутации (settings, alert_rules CRUD) используют `sqlalchemy.text(...)` или `select/insert/update` Core API.

---

## 4. M1 — Foundation (детализация)

> M1 расширяет Phase 0 до полноценной storage / observability / normalizer-foundation на одной бирже Binance. После M1 один Binance работает с CA, бизнес-метриками, retention, cleanup, полным normalizer pipeline.

### M1.A — Continuous aggregates (отдельная Alembic-миграция)

| # | Задача | Источник | Артефакт | Часы | Риск |
|---|---|---|---|---|---|
| M1.A.1 | Миграция `0003_oi_5m_ca.py` — `oi_5m` MV с `WITH NO DATA` + `add_continuous_aggregate_policy` (start=1h, end=30s, schedule=30s) + retention 90d + idx_oi_5m_lookup; полный `downgrade()` | `08 §3.2, §13.4` | revision 0003 | 1.5h | Med |
| M1.A.2 | Миграция `0004_oi_15m_ca.py` — hierarchical CA `FROM oi_5m`, refresh schedule 60s, retention 90d | `08 §3.3` | revision 0004 | 1h | Low |
| M1.A.3 | Миграция `0005_oi_30m_ca.py` — аналогично 15m, time_bucket 30 minutes | `08 §3.4` | revision 0005 | 1h | Low |
| M1.A.4 | Миграция `0006_oi_quality_5m.py` — quality view (sample_count, valuation distribution, source_kind distribution, avg_lag_sec) | `08 §7` | revision 0006 | 1h | Low |
| M1.A.5 | Migration test для всех 4 CA: upgrade idempotent + downgrade в правильном порядке (oi_30m → oi_15m → oi_5m → oi_quality_5m) | `08 §13.5`, `14 §5` | passing test | 1.5h | Med (порядок drop) |

**M1.A итого:** ~6h.

### M1.B — Полный normalizer pipeline + бизнес-метрики

| # | Задача | Источник | Артефакт | Часы | Риск |
|---|---|---|---|---|---|
| M1.B.1 | `app/normalizer/parsers/binance.py` — extracted из 0.D.5, parse_open_interest для Binance schema | `07 §3`, `docs/11_EXCHANGE_ADAPTERS/binance.md` | parser module | 1.5h | Low |
| M1.B.2 | `app/normalizer/instrument_resolver.py` — lookup `(exchange, native_symbol)` → Instrument в registry, error if delisted/blacklisted | `07 §4`, `06_INSTRUMENT_REGISTRY.md` | resolver | 1.5h | Low |
| M1.B.3 | `app/normalizer/unit_resolver.py` — определить oi_unit_hint, конвертировать в `oi_coins` (для Binance OI in base_asset — identity) | `07 §5` | unit converter | 2h | Med (логика для Binance simple, full в M2) |
| M1.B.4 | `app/normalizer/price_resolver.py` + `app/normalizer/_price_cache.py` — TTL-кэш mark/index price на 60s per (exchange, native_symbol) | `07 §6`, `11/_BASE §3` | price attachment | 2h | Med |
| M1.B.5 | `app/normalizer/quality_assessor.py` — назначение `valuation_status ∈ {authoritative, good_estimate, low_confidence}` по правилам F10 + warnings (clock_skew, fallback_to_last) | `07 §7`, `00 F10` | quality logic | 2h | Med |
| M1.B.6 | `app/observability/metrics.py` — регистрация бизнес-метрик: `oi_collector_request_total{exchange}`, `oi_collector_cycle_duration_seconds`, `oi_storage_insert_duration_seconds{table}`, `oi_collector_samples_collected_total`, `oi_normalize_duration_seconds{exchange,stage}`, `oi_normalize_errors_total{exchange,error_type,failed_stage}`, `oi_valuation_status_total{exchange,status}` | `12 §3.1, §3.3, §3.4` | metrics module | 2h | Low |
| M1.B.7 | Wire-up метрик через декораторы / context managers в connector + normalizer + storage | M1.B.6 | metrics emit | 1.5h | Low |

**M1.B итого:** ~12.5h.

### M1.C — Cleanup service + DoD

| # | Задача | Источник | Артефакт | Часы | Риск |
|---|---|---|---|---|---|
| M1.C.1 | `app/cleanup_main.py` — daily job: `refresh_continuous_aggregate` для oi_5m/15m/30m/quality, ANALYZE, DELETE expired sse_sessions | `08 §9.1` | cleanup entrypoint | 1.5h | Low |
| M1.C.2 | `deploy/systemd/oi-tracker-cleanup.timer` + `.service` (OnCalendar=03:15 UTC) | `13 §3.4` | timer + oneshot service | 0.5h | Low |
| M1.C.3 | `app/api/routes/health.py` — расширенный `/health`: db ok, scheduler last_cycle_age, last_sample_age per exchange (только Binance в M1) | `10 §4.11` | health route | 1h | Low |
| M1.C.4 | Unit tests `tests/unit/normalizer/` — quality_assessor matrix (авторитативный/оценка/low_confidence × warnings combo), unit_resolver, price_resolver TTL | `14 §3.1, §3.2` | passing unit suite | 3h | Low |
| M1.C.5 | M1 DoD verification: 1h непрерывный run → ≥ 55 циклов (92% coverage); `oi_samples` populated, `valuation_status` корректен; `/metrics` экспортирует `oi_collector_*` и `oi_storage_*`; coverage normalizer+storage ≥ 80%; mypy/ruff clean; соседи не пострадали | `16 §M1 DoD`, `CLAUDE.md §4.1` | DoD passed | 1h | Low |

**M1.C итого:** ~7h.

**M1 итого: ~25.5h** (+ buffer ~5h на отладку CA refresh policies = **30h**).

---

## 5. Pre-M2 prerequisites

> Из `16 §Pre-milestone prerequisites` — три ADR'а до старта M2:

### 5.1 Scheduler concurrency model

**Статус:** **Decided** в `02 §5.4` — `asyncio.TaskGroup` + supervisor + per-exchange isolation через nested TaskGroup. Не блокирует M2.

### 5.2 Backpressure

**Статус:** **Decided** в `02 §5.5` — bounded `asyncio.Queue` (raw 20k, sample 10k) + drop-oldest политика + метрики `oi_*_dropped_total`, `*_queue_depth`. Не блокирует M2.

### 5.3 Connection pool sizes per-service

**Статус:** **Open**. `16 §Pre-milestone prerequisites` упоминает «scheduler 20, api 10, tg-sender 5, cleanup 2 = 37 + запас при PG max_connections=100», но это рекомендация, не зафиксированное решение. Перед стартом M2 — небольшой документационный PR в `02 §5.3` или `13 §1.x` фиксирующий точные значения.

**Действие перед M2:**
- [ ] PR в `02_ARCHITECTURE.md §5.3` с явной таблицей pool sizes (scheduler=20, api=10, tg-sender=5, cleanup=2; обоснование = укладка в 37 коннектов + запас 30 для соседей при `max_connections=100`).
- [ ] Регистрация значений в `app/config.py` как env-overridable константы.
- [ ] Прометей-метрика `oi_db_pool_connections{service,state=acquired/idle}` — добавить в `12 §3.4`.

**Оценка:** ~1.5h на PR в docs + код.

---

## 6. M2 — All 12 connectors + normalizer полный + instrument registry

> Самый объёмный milestone. Декомпозиция: registry sync-job, circuit breaker полноценный, exchange-health detection, replay-tool stub, и **11 коннекторов** (Binance уже в Phase 0).

### M2.A — Registry sync + per-exchange machinery

| # | Задача | Источник | Артефакт | Часы | Риск |
|---|---|---|---|---|---|
| M2.A.1 | `app/instruments/registry.py` — repository поверх `instruments` таблицы (find_active, upsert, mark_delisted, blacklist toggle, instrument_version bump на изменении contract_size/multiplier) | `06_INSTRUMENT_REGISTRY.md`, `05 §11` | registry module | 2.5h | Low |
| M2.A.2 | `app/instruments/sync_job.py` — async task, каждые 5 мин (F2): для каждого активного коннектора → connector.sync_instruments() → diff с registry → upsert/delist/version bump | `00 F2`, `06`, `02 §5.4` (instruments_sync_loop) | sync job | 2.5h | Med |
| M2.A.3 | Метрики instruments registry: `oi_instruments_total{exchange,status}`, `oi_instruments_sync_duration_seconds{exchange}`, `oi_instruments_new_total{exchange}`, `oi_instruments_delisted_total{exchange}`, `oi_instruments_changed_total{exchange,change_type}`, `oi_instruments_sync_errors_total{exchange}` | `12 §3.2` | metrics emit | 1h | Low |
| M2.A.4 | `app/exchanges/_circuit_breaker.py` — полная реализация state machine (closed → half_open → open) с metrics `oi_collector_circuit_breaker_state{exchange}` (0/1/2) | `00 F3`, `12 §3.1`, `02 §3.2` | full circuit breaker | 2h | Med |
| M2.A.5 | Exchange-health metrics: `oi_collector_symbol_coverage_ratio{exchange}` (received/active per cycle), `oi_collector_stale_seconds{exchange}` (now - max(ts_exchange) per exchange), `oi_collector_up{exchange}` | `12 §3.1` | health metrics | 1.5h | Low |
| M2.A.6 | Полный `app/scheduler/loop.py` с `asyncio.TaskGroup` + supervisor + nested per-connector TaskGroup для error isolation (F3) + bounded queues `raw_queue/sample_queue` + drop-oldest политика | `02 §5.4, §5.5` | scheduler v2 | 4h | **High** — критическая логика для error isolation и backpressure |
| M2.A.7 | Backpressure metrics: `oi_storage_write_queue_depth`, `oi_normalize_input_queue_depth`, `oi_collector_raw_dropped_total{exchange}`, `oi_normalize_sample_dropped_total{exchange}`, `oi_collector_cycle_overlap_total{exchange}` | `02 §5.5`, `12 §3.3, §3.4` | metrics | 1h | Low |
| M2.A.8 | `app/tools/replay.py` — stub CLI с argparse: `--exchange`, `--since`, `--until`, `--version`. Читает raw_exchange_events, прогоняет normalizer с целевой `normalization_version`, INSERT ON CONFLICT DO UPDATE WHERE normalization_version < EXCLUDED, refresh CA range. **No retro-fire** (F9) — alert engine не вызывается | `08 §6.2`, `13 §6.2`, `00 F9` | CLI tool | 2.5h | Med |

**M2.A итого:** ~17h.

### M2.B — 11 коннекторов

> **Точка параллелизации:** все 11 коннекторов могут писаться параллельно через `exchange-adapter-builder` agent (см. `CLAUDE.md §11.1`) в отдельных PR. Зависят только от `BaseConnector` и normalizer — оба готовы после Phase 0 / M1.

Для каждого коннектора задача состоит из 4 подзадач:
- (a) `app/exchanges/<exchange>.py` — реализация sync_instruments + fetch_snapshot (или fetch_range) + fetch_prices.
- (b) `app/normalizer/parsers/<exchange>.py` — parser.
- (c) `tests/contract/<exchange>/fixtures/*.json` — реальные payload-фикстуры (capture через `app/tools/capture_fixtures.py`).
- (d) `tests/contract/<exchange>/test_*.py` — минимум 1 happy path + 1 error case + mapping test (≥ 80% coverage per file).

| # | Биржа | source_kind | Adapter doc | Особенности | Часы (a+b+c+d) |
|---|---|---|---|---|---|
| M2.B.1 | OKX | snapshot per symbol | `docs/11_EXCHANGE_ADAPTERS/okx.md` | OI в контрактах + ctVal + settleCcy; формулировки `C15` для market_type/contract_type | **8h** |
| M2.B.2 | Bitget | snapshot per symbol | `docs/11_EXCHANGE_ADAPTERS/bitget.md` | Стандартный snapshot, low risk | **6h** |
| M2.B.3 | Gate.io | snapshot per symbol | `docs/11_EXCHANGE_ADAPTERS/gateio.md` | Стандартный snapshot | **6h** |
| M2.B.4 | MEXC | snapshot per symbol | `docs/11_EXCHANGE_ADAPTERS/mexc.md` | Стандартный snapshot, иногда unstable (см. UX-сценарий Phase D) | **7h** |
| M2.B.5 | Hyperliquid | snapshot (info endpoint) | `docs/11_EXCHANGE_ADAPTERS/hyperliquid.md` | POST к `info` endpoint, специфический формат, `C15` корректуры | **9h** |
| M2.B.6 | Aster | snapshot (binance-like) | `docs/11_EXCHANGE_ADAPTERS/aster.md` | API mimics Binance, можно частично переиспользовать парсер; `C15` корректуры | **5h** (re-use) |
| M2.B.7 | Bitmart | snapshot bulk (single endpoint) | `docs/11_EXCHANGE_ADAPTERS/bitmart.md` | Один bulk endpoint = roster + OI + price + filter; pattern XT-клон. Заменил Bitunix per F42 (live capture 2026-05-04 подтвердил OI в public API). | **5h** (re-use XT pattern) |
| M2.B.8 | HTX | bulk OI endpoint | `docs/11_EXCHANGE_ADAPTERS/htx.md` | Bulk endpoint = один HTTP-запрос за весь рынок, проще rate limits, но требует выборки нужных пар | **7h** |
| M2.B.9 | XT | bulk snapshot | `docs/11_EXCHANGE_ADAPTERS/xt.md` | Один endpoint = все контракты сразу + отдельный contractSize, конверсия contracts × size; `C12` | **9h** |
| M2.B.10 | Bybit | **native_interval** primary | `docs/11_EXCHANGE_ADAPTERS/bybit.md` | Реализация `fetch_range` для 5m/15m/30m баров; degraded_fallback на snapshot при > 2 cycles деградации (`C12`); `unit=base_asset`; самый сложный коннектор | **14h** |
| M2.B.11 | KuCoin | **native_interval** primary | `docs/11_EXCHANGE_ADAPTERS/kucoin.md` | Аналогично Bybit, но другой API + другая логика token-based auth public endpoints; `C12` | **14h** |

**M2.B итого: ~93h** (если параллелить через 3 агентов по группам — реальное время ~35h на ведущего разработчика).

### M2.C — Integration + DoD

| # | Задача | Источник | Артефакт | Часы | Риск |
|---|---|---|---|---|---|
| M2.C.1 | Полный normalizer для всех 12 типов OI: contracts × contract_size, native notional (Binance authoritative), oi_coins прямой (Bybit base_asset), bulk-snapshot (XT/HTX), info-endpoint (Hyperliquid) — расширение unit_resolver и price_resolver | `07 §5, §6` | normalizer v2 | 4h | Med |
| M2.C.2 | E2E integration тест: запустить scheduler с mock-серверами всех 12 бирж, run 5 циклов, assert `SELECT exchange, COUNT(DISTINCT canonical_symbol) FROM oi_samples GROUP BY 1 → ≥ 1 для каждой биржи` | `14 §6.3, §6.4` | e2e test | 3h | Med |
| M2.C.3 | M2 DoD verification: все 12 contract-тестов PASS, coverage `app/exchanges/*` ≥ 90%, 1h run → `symbol_coverage_ratio ≥ 95%` per exchange, decisions-guardian PASS, verify-spec PASS (типы, canonical_symbol=BASE, oi_coins primary), replay-tool на синтетическом backfill OK | `16 §M2 DoD` | DoD passed | 2h | Low |

**M2.C итого:** ~9h.

**M2 итого: ~119h** (с параллелизацией 11 коннекторов: ~60h на ведущего dev). Buffer ~15h.

---

## 7. Pre-M3 prerequisites

> Из `16 §Pre-milestone prerequisites` — два **открытых** ADR'а до M3:

### 7.1 aiogram throttle

**Статус:** ✅ **DECIDED** — `00_DECISIONS_LOG.md F22` + `10_DELIVERY_LAYER.md §2.5.2`. Leaky-bucket: 25 msg/sec global + 1 msg/sec per chat; 429 fallback использует TG `retry_after` напрямую и фризит global bucket. Реализация — `app/tg_sender/throttle.py`, переиспользует `_rate_limiter.RateLimiter`. Land'ит в M3.B.

**Действие:** ~2h на код в M3.B + dry-run-friendly: throttle включён даже при `tg_dry_run=true`.

### 7.2 Settings hot-reload

**Статус:** ✅ **DECIDED** — `00_DECISIONS_LOG.md F23`: hot-reload **не реализуем в v1**. Все процессы читают settings при старте; UI Settings PUT возвращает `{"applied": false, "restart_required": [...]}` чтобы пользователь видел какие сервисы перезапустить. Reactivation criteria зафиксированы.

**Обоснование вкратце:** single-user (`Q4`) → ~5s downtime от `systemctl restart` приемлем; cost SIGHUP/poll-watcher/LISTEN-NOTIFY > benefit; YAGNI до multi-user setup.

**Действие:** комментарии в `app/config.py` + `app/storage/repositories/settings_kv.py` (placeholder), endpoint behavior в M5 (UI Settings) реализуется с `restart_required` массивом в response.

---

## 8. M3 — Alert engine + Telegram delivery

### M3.A — alert_rules + state machine + quality gates

| # | Задача | Источник | Артефакт | Часы | Риск |
|---|---|---|---|---|---|
| M3.A.1 | Миграция `0007_alert_rules_seed.py` — INSERT 6 default rows (5%/8%/12% × up/down × 5/15/30 min) согласно `F18` exact values | `09 §7`, `00 F18` | revision 0007 + 6 rows | 1h | Low |
| M3.A.2 | `app/alert_engine/state_machine.py` — pure function `transition(state, candidate, rule) → new_state` реализующая 4-state machine (idle → armed → fired → cooldown → armed) с переходами `09 §3.2` и pseudocode `09 §3.4` | `09 §3`, `00 C6` | state_machine.py | 4h | **High** — критическая корректность; coverage ≥ 95% |
| M3.A.3 | Unit tests state machine — параметризованные сценарии из `09 §13.1` + `14 §3.3`: armed_to_fired, no_cross, armed_already_above, cooldown_smart_re_fire, cooldown_no_smart_re_fire, cooldown_to_armed_after_ttl, insufficient_history, stale_data_skip, instrument_delisted (минимум 16 cases) | `14 §3.3`, `09 §13.1` | tests + ≥ 95% coverage | 3h | Med |
| M3.A.4 | `app/alert_engine/quality_gate.py` — фильтры: freshness (sample_age_seconds <= freshness_budget_now_sec), valuation_status_min, source_kind_filter, min_oi_notional_usdt, min_history_minutes (формула `max(30, 2×window)` = `F8`), delta_pct is not None | `09 §4`, `00 F8` | quality gate | 2h | Med |
| M3.A.5 | `app/alert_engine/cooldown.py` — smart cooldown algorithm (`09 §6.1`): re-fire if `|candidate.delta_pct| >= |state.last_fired_value| × smart_cooldown_factor` (default 1.5), extension cooldown timer | `09 §6`, `00 Q1, F7` | cooldown logic | 2h | Med |
| M3.A.6 | `app/config/runtime_settings.py` — TTL-cache 30s (Pre-M3 решение), async `RuntimeSettings.get(key)` через SELECT FROM settings | Pre-M3 §7.2 | runtime settings | 2h | Low |

**M3.A итого:** ~14h.

### M3.B — Signal types + delivery_queue

| # | Задача | Источник | Артефакт | Часы | Риск |
|---|---|---|---|---|---|
| M3.B.1 | `app/alert_engine/signals/threshold.py` + `confirmed.py` (confirm_points >= 2 через consecutive_above_count) | `09 §5.1, §5.2` | 2 signal modules | 2h | Low |
| M3.B.2 | `app/alert_engine/signals/consensus.py` — group bars by canonical_symbol, count exchanges with delta >= threshold, fire if ≥ M (default 3 из `consensus_min_exchanges`); cross-exchange bucket alignment | `09 §5.3` | consensus signal | 3h | Med |
| M3.B.3 | `app/alert_engine/signals/divergence.py` — type A: `delta_oi >= threshold AND delta_price <= -divergence_price_threshold` (или зеркальный); игнорирует degraded_fallback | `09 §5.4` | divergence signal | 2.5h | Med |
| M3.B.4 | `app/alert_engine/signals/exchange_health.py` — 4 sub-triggers: stale_data (max(ts_exchange) старше 5min), low_coverage (< 95% за 3 cycles), high_error_rate (> 5/min), instruments_sync_failing (3 fail подряд); cooldown 1h | `09 §5.5` | health signal | 3h | Med |
| M3.B.5 | `app/alert_engine/engine.py` — orchestrator: каждые `evaluation_cycle_sec` (default 30) → load enabled rules → for each rule: signal.evaluate(rule) → state.transition → write alert_event → if FIRE: enqueue delivery | `09 §8`, `09 §3, §10` | engine | 3h | Med |
| M3.B.6 | `app/alert_engine/delivery_producer.py` — `INSERT INTO delivery_queue ON CONFLICT (dedupe_key) DO NOTHING`; dedupe_key формат `rule:<id>|ex:<ex>|sym:<sym>|win:<w>|bucket:<ts_iso>` | `09 §9`, `05 §10` | producer | 1.5h | Low |
| M3.B.7 | Метрики alert engine: `oi_alert_engine_cycle_duration_seconds`, `oi_alert_engine_decisions_total{rule_id,decision}`, `oi_alert_engine_fires_total{rule_id,exchange,signal_type}`, `oi_alert_engine_state_transitions_total{from_state,to_state}`, `oi_alert_engine_smart_re_fires_total{rule_id}`, `oi_alert_e2e_latency_seconds{rule_type}` (histogram, ts_processed - ts_exchange), `oi_alert_quality_gate_fails_total{reason}`, `oi_alert_state_size` | `12 §3.5` | metrics | 1.5h | Low |

**M3.B итого:** ~16.5h.

### M3.C — TG sender service

| # | Задача | Источник | Артефакт | Часы | Риск |
|---|---|---|---|---|---|
| M3.C.1 | `app/tg_sender/templates.py` — 5 jinja-like templates: `oi_threshold_cross`, `oi_threshold_cross_smart_refire`, `consensus_alert`, `divergence_alert`, `exchange_health_alert` + `humanize_usd()` | `10 §3.1–3.6` | templates module | 2h | Low |
| M3.C.2 | `app/tg_sender/throttle.py` — leaky-bucket (1 msg/sec per chat, 25 msg/sec global) | Pre-M3 §7.1 | throttle | 2h | Med |
| M3.C.3 | `app/tg_sender/sender.py` — async loop: poll delivery_queue (every 2s, limit 10), for each pending → mark_sending → render → throttle.acquire() → bot.send_message → mark_sent OR retry с backoff (60s/120s/240s/480s/600s, max 5 retries → mark_failed) | `10 §2.3, §2.4, §2.5` | sender loop | 3h | Med |
| M3.C.4 | `app/tg_sender_main.py` — entrypoint: read TELEGRAM_BOT_TOKEN from env, runtime_settings.get("tg_chat_id") + tg_dry_run → connect aiogram bot → run sender loop | `10 §2.6` | entrypoint | 1h | Low |
| M3.C.5 | `deploy/systemd/oi-tracker-tg-sender.service` (Type=simple, MemoryMax=512M, CPUWeight=50, ProtectSystem=strict, ReadWritePaths=/var/log/oi-tracker) | `13 §3.3` | unit-файл | 0.5h | Low |
| M3.C.6 | TG metrics: `oi_tg_sent_total{template}`, `oi_tg_failed_total{template,reason}`, `oi_tg_retry_total{template}`, `oi_tg_pending_size`, `oi_tg_oldest_pending_age_seconds`, `oi_tg_throttle_wait_seconds` | `10 §8.1`, `12 §3.6` | metrics | 1h | Low |
| M3.C.7 | Contract test `tests/contract/tg/test_aiogram_send.py` — mocked aiogram bot; verify retry policy (TelegramRetryAfter → schedule_retry), verify dry_run пишет в лог не отправляя | `14 §4` | passing test | 2h | Med |

**M3.C итого:** ~11.5h.

### M3.D — No retro-fire test + DoD

| # | Задача | Источник | Артефакт | Часы | Риск |
|---|---|---|---|---|---|
| M3.D.1 | E2E test `tests/e2e/test_no_retro_fire_after_replay.py` — стейт фиксирует cooldown_until = past_time, replay создаёт исторический FIRE-кандидат с ts_exchange < state.cooldown_until, assert `SELECT count(*) FROM delivery_queue → 0` | `00 F9`, `09 §13.4`, `14 §6.2` | passing test | 2h | Med |
| M3.D.2 | E2E test `tests/e2e/test_smart_cooldown.py` — fire +5.2% → внутри cooldown +6% (suppress) → +9% (re-fire) → +10% (suppress, 10 < 9×1.5=13.5) | `09 §13.3` | passing test | 2h | Low |
| M3.D.3 | E2E test `tests/e2e/test_full_pipeline_threshold.py` + consensus + divergence + exchange_health (4 файла) | `14 §6.4` | 4 passing tests | 4h | Med |
| M3.D.4 | Boot test (manual): chat_id = test группа, dry_run=false, induce delta → tg_threshold_cross приходит ≤ 90s (P95 SLO) | `16 §M3 DoD` | manual verify | 1h | Low |
| M3.D.5 | M3 DoD verification: state machine coverage ≥ 95%, smart cooldown тесты для каждого сценария, TG sender contract test PASS, `oi_alert_e2e_latency_seconds` histogram активен и обновляется, decisions-guardian PASS | `16 §M3 DoD` | DoD passed | 2h | Low |

**M3.D итого:** ~11h.

**M3 итого: ~53h** + 4h pre-M3 ADR работа + buffer ~10h = **~67h**.

---

## 9. Pre-M4 prerequisites

> Оба ADR закрыты 2026-05-01.

### 9.1 SSE no-replay disclaimer

**Статус:** ✓ **Decided** (2026-05-01) — `00_DECISIONS_LOG F27` + `10_DELIVERY_LAYER §5.5.2`.

Контракт: на reconnect клиент re-fetches snapshot через
`event: connection_init`; пропущенные события в gap'е НЕ воспроизводятся.
Канонические FIRE сохраняются через TG (с retry до 5). Live OI пишется
заново следующим polling cycle (≤60s). UI обязан показывать
ConnectionStatusBadge (`18 §7.6`) — три состояния live/reconnecting/offline.
Метрика: `oi_sse_queue_full_total{channel}` (`12 §3.6`) — Grafana алерт
«>0 за 5 минут».

### 9.2 PG NOTIFY 8KB payload limit policy

**Статус:** ✓ **Decided** (2026-05-01) — `00_DECISIONS_LOG F28` + `10_DELIVERY_LAYER §5.5.1`.
Превышение 8000 байт поднимает `ERROR: payload string too long` и прерывает INSERT — недопустимо в hot path `oi_samples`. Payload зафиксирован keys-only:
- `oi_new_sample`: `exchange`, `canonical_symbol`, `ts_exchange`.
- `oi_new_alert`: `rule_id`, `exchange`, `canonical_symbol`, `ts_processed`.

Расширение запрещено без новой F-записи. Реализация триггеров — в M4.B.1
(миграция `0010_pg_notify_triggers.py`).

---

## 10. M4 — Web UI

### M4.A — Frontend skeleton + design tokens

| # | Задача | Источник | Артефакт | Часы | Риск |
|---|---|---|---|---|---|
| M4.A.1 | `frontend/package.json` (React 18, Vite 5, TS strict, TanStack Table v8, TanStack Virtual, Recharts, Tailwind, Lucide) + `tsconfig.json` strict + `vite.config.ts` (proxy /api и /sse → 127.0.0.1:8010 для local dev) | `17 §2.2, §2.3, §2.5` | scaffold | 2h | Low |
| M4.A.2 | `frontend/src/styles/tokens/{colors,typography,spacing,motion,elevation}.css` — OKLCH variables по `18 §2.3` (dark theme) + light theme | `18 §2, §12` | token files | 3h | Med (OKLCH без HSL fallback — Chrome 111+, см. `18 §15`) |
| M4.A.3 | Self-host fonts: `Inter.var.woff2` + `JetBrainsMono.var.woff2` в `public/fonts/`, `@font-face` с `font-display: swap`, `font-feature-settings` per `18 §3.1` | `18 §3.1` | fonts | 1.5h | Low |
| M4.A.4 | `frontend/src/styles/themes/{dark,light}.css` + `tailwind.config.ts` proxy CSS variables (`18 §12.2`) | `18 §12` | tailwind config | 1.5h | Low |
| M4.A.5 | Layout shell: `AppShell` (100dvh fixed), `Sidebar` (72px rail + hover overlay), `TopBar` (56px), `PageHeader` (sticky), routing через `react-router-dom` | `18 §4.3, §4.4` | layout | 3h | Low |
| M4.A.6 | Density toggle (`compact`/`comfortable` через `data-density` на html), persisted в localStorage + PUT `/settings/ui_density` (новый категория-A key) | `18 §4.2` | density mechanism | 2h | Low |
| M4.A.7 | Theme toggle (dark/light), persisted в localStorage + settings | `18 §2.6` | theme toggle | 1.5h | Low |
| M4.A.8 | `useSSE` hook с auto-reconnect (1s → 2s → 4s → ... → 30s cap) + connection status state | `10 §5.6` | hook | 2h | Med |
| M4.A.9 | `useApi` hook — типизированный fetch с error handling, OpenAPI codegen pipeline (или ручные types сгенерированные из pydantic schemas в `app/tools/export_schemas.py`) | `05 §16`, `10 §6.2` | hook + types | 2.5h | Med |
| M4.A.10 | Common components: `ExchangeChip`, `DeltaCell` (intensity-aware §2.4), `FreshnessIndicator` (§7.4), `ValuationBadge` (§7.5), `ConnectionStatusBadge` (§7.6) | `18 §7.3–7.6` | 5 components | 4h | Med |

**M4.A итого:** ~23h.

### M4.B — Backend SSE + REST endpoints

| # | Задача | Источник | Артефакт | Часы | Риск |
|---|---|---|---|---|---|
| M4.B.1 | DB триггеры `notify_new_oi_sample` + `notify_new_alert` (миграция `0010_pg_notify_triggers.py`) — payload только keys per `F28` | `00 F28`, `10 §5.5.1`, Pre-M4 §9.2 | revision 0010 | 1h | Med |
| M4.B.2 | `app/api/sse/hub.py` — `SSEHub` с `subscribe/unsubscribe/publish` + per-subscriber `asyncio.Queue(maxsize=1000)` + drop-on-full с `sse_queue_full` warn | `10 §5.4` | SSE hub | 2h | Low |
| M4.B.3 | `app/api/sse/listener.py` — постоянное `LISTEN oi_new_sample / oi_new_alert` через asyncpg + forward в SSE Hub | `10 §5.5` | listener | 2.5h | Med |
| M4.B.4 | Endpoints `/sse/v1/live` + `/sse/v1/alerts` — StreamingResponse, initial `connection_init` event с snapshot, heartbeat 30s, request.is_disconnected() loop | `10 §5.3, §5.6` | SSE routes | 2.5h | Med |
| M4.B.5 | REST endpoint `GET /api/v1/live` (filter exchange, min_oi_usdt, sort, limit; reads из `live_dashboard_mv` или dynamic CTE per `08 §5.4`) | `10 §4.3`, `08 §5.4` | route + repo | 3h | Med |
| M4.B.6 | REST endpoint `GET /api/v1/symbols/{canonical}` (period selector, exchanges filter, decimation per `08 §5.5`) | `10 §4.4`, `08 §5.5` | route + decimation logic | 3h | Med |
| M4.B.7 | REST endpoint `GET /api/v1/alerts` (since/until/exchange/symbol/window/decision filters, pagination) | `10 §4.5` | route | 2h | Low |
| M4.B.8 | REST endpoints `GET /alert-rules` + `PUT /alert-rules/{id}` (update threshold для default rules) | `10 §4.6, §4.7` | routes | 1.5h | Low |
| M4.B.9 | REST endpoints `GET /settings` + `PUT /settings/{key}` с pydantic-валидацией категория A+B (диапазоны из `10 §4.8`), audit log INSERT, **rate-limit 5/min per IP** через slowapi | `10 §4.8`, `00 F15` | routes + rate-limit | 3h | Med |
| M4.B.10 | REST endpoints `GET /instruments` + `POST /instruments/{id}/blacklist` + `POST /instruments/sync?exchange=...` (force sync) | `10 §4.9, §4.10`, `10 §6.1 Tab 2` | routes | 2h | Low |
| M4.B.11 | REST endpoint `POST /api/v1/tg/test` — отправляет дефолтное сообщение в текущий tg_chat_id (или dry_run в лог) | `10 §6.1 Tab 3` | route | 1h | Low |
| M4.B.12 | API metrics: `oi_api_requests_total{endpoint,method,status_code}`, `oi_api_request_duration_seconds{endpoint}`, `oi_sse_connections{channel}`, `oi_sse_events_published_total{channel,event_type}` | `12 §3.6` | metrics | 1h | Low |

**M4.B итого:** ~24.5h.

### M4.C — UI pages (могут параллелиться: 4 страницы × ~10h = 40h, но требуют связки на конец)

| # | Страница | Источник | Подзадачи | Часы |
|---|---|---|---|---|
| M4.C.1 | **Dashboard** `/` | `10 §6.1`, `18 §8.1` | Status strip (32px), Filters bar (40px), LiveTable (TanStack Table + virtualization, 9 колонок compact mode по `18 §8.1`), DeltaCell flash animation per ref (`18 §7.1`), row context menu, sort persistence localStorage, SSE event throttling 200ms, empty/loading state | **14h** |
| M4.C.2 | **Symbol page** `/symbol/:canonical` | `10 §6.1`, `18 §8.2` | Header + period chips segmented, main chart 12 lines (Recharts, `dot={false}`, `isAnimationActive={false}`, crosshair sync, hue per `EXCHANGE_HUE`), per-exchange table compact, recent alerts last 24h, toggle lines via chip click | **12h** |
| M4.C.3 | **Alerts log** `/alerts` | `10 §6.1`, `18 §8.3` | Table + sticky filters bar, Live mode toggle (SSE), decision/delivery badges, alert-row-enter animation §6.3, click row → side drawer JSON viewer | **8h** |
| M4.C.4 | **Settings** `/settings` | `10 §6.1 Tab 1-5`, `18 §8.4` | 5 tabs (Alerts/Filters/Telegram/Advanced/History), 6 default thresholds slider form (`F18`) с PUT /alert-rules/{id}, full UI Settings категория A+B per `10 §4.8`, autosave debounced 300ms, 429 toast handling, audit log read-only Tab 5 | **12h** |

**M4.C итого:** ~46h.

### M4.D — DoD acceptance + design system §13

| # | Задача | Источник | Артефакт | Часы | Риск |
|---|---|---|---|---|---|
| M4.D.1 | Production build → `/var/www/oi-tracker/frontend-dist/`, sourcemaps, gzip-compressed (Vite default), bundle ≤ 250 KB initial gz (verify через `webpack-bundle-analyzer` или `rollup-plugin-visualizer`) | `13 §2.1.6`, `18 §10`, `16 §M4 DoD` | bundle within budget | 2h | Med |
| M4.D.2 | noindex audit live: `curl -I https://oi-tracker.robot-detector.ru/` → 200 + `X-Robots-Tag: noindex,nofollow,noarchive,nosnippet`; `/robots.txt` → `Disallow: /`; HTML meta присутствует | `15 §3`, `16 §M4 DoD` | live verify | 0.5h | Low |
| M4.D.3 | SSE 1-минутный smoke без drop'ов (consume через `curl -N` + count events) | `16 §M4 DoD` | smoke OK | 0.5h | Low |
| M4.D.4 | Frontend tests Vitest: hooks (`useSSE`, `useApi`, `useDebounce`), formatters (`humanize_usd`, `formatDelta`), Settings form validation; coverage ≥ 70% | `14 §9`, `18 §3.5` | passing tests | 5h | Med |
| M4.D.5 | Design-system §13 acceptance чек-лист (10 пунктов): no layout shift on SSE update (CLS=0), tabular alignment во всех состояниях, connection status видим, freshness видим, density toggle работает + persistent, theme toggle работает + light contrast PASS, reduced motion уважается, keyboard nav dashboard полностью без мыши, empty/loading/error states у каждой страницы, bundle ≤ 250 KB gz | `18 §13` | acceptance PASS | 4h | Med |
| M4.D.6 | M4 DoD final: 4 страницы golden-path manual smoke OK, dashboard 3000 rows без UI-jank (TanStack virtualization работает), mypy/ruff/eslint/tsc clean, verify-spec PASS | `16 §M4 DoD` | DoD passed | 2h | Low |

**M4.D итого:** ~14h.

**M4 итого: ~107.5h** + 2h pre-M4 ADR + buffer ~15h = **~125h** (самый объёмный milestone).

---

## 11. Pre-M5 prerequisites

> Один **открытый** ADR до M5:

### 11.1 Trace context propagation

**Статус:** **Open**. `12 §5` не описывает сквозной `trace_id`. Дебаг «почему этот алерт пришёл с задержкой 87s» без trace_id занимает часы.

**Действие перед M5:**
- [ ] PR в `12_OBSERVABILITY_SLO.md §5` дополнение: «`trace_id = cycle_id from connector` (uuid4 generated at `execute_cycle` start). Propagated через: `RawExchangeEvent.request_id` → нормализация переносит в `NormalizedOISample.warnings` field как metadata, либо в structlog context → `AlertCandidate` → `AlertEvent` → `DeliveryRequest.payload._trace_id` → tg_sender логирует. Каждый log entry содержит `trace_id` поле».
- [ ] Имплементация: `structlog.contextvars.bind_contextvars(trace_id=...)` в начале каждого cycle.

**Оценка:** ~1.5h на ADR-PR + 3h на код.

---

## 12. M5 — Observability complete + runbooks + hardening

### M5.A — Метрики + дашборды

| # | Задача | Источник | Артефакт | Часы | Риск |
|---|---|---|---|---|---|
| M5.A.1 | Полная инвентаризация метрик per `12 §3.1–3.7` — verify что все 40+ метрик emit'ятся; добавить недостающие (например `oi_clock_skew_seconds{exchange}`, postgres_exporter wiring, timescaledb_exporter) | `12 §3` | metrics complete | 4h | Med |
| M5.A.2 | Trace context propagation реализация (Pre-M5 §11.1) | `12 §5` | trace_id в каждом log | 3h | Med |
| M5.A.3 | Loki labels стандартизация: `service ∈ {api, scheduler, tg-sender, cleanup}`, `level`, `exchange` (low cardinality); content fields для `request_id`, `canonical_symbol`, `dedupe_key`, `trace_id` | `12 §5.2, §5.3` | promtail config | 2h | Low |
| M5.A.4 | Grafana dashboard `system-overview.json` — status grid (12 ячеек), E2E latency P50/P95/P99, alerts/min, coverage (12 lines), disk usage, memory/CPU per service | `12 §4.1` | dashboard | 3h | Low |
| M5.A.5 | Grafana dashboard `exchanges.json` — per-exchange uptime, fetch latency, error rate, symbol coverage, clock skew, rate limit budget, top errors | `12 §4.2` | dashboard | 3h | Low |
| M5.A.6 | Grafana dashboard `alerts.json` — decisions distribution per rule, top firing symbols, fires per signal type, E2E latency histogram, smart re-fire rate, TG delivery success rate | `12 §4.3` | dashboard | 3h | Low |
| M5.A.7 | Grafana dashboard `latency.json` (storage + alert engine pipelines): hypertable size growth, compressed/uncompressed chunks, CA refresh duration, slow queries top-10, insert rate per table, connection pool utilization | `12 §4.4` | dashboard | 3h | Low |
| M5.A.8 | Grafana dashboard `business.json` (per `12 §4.5`): top movers hourly, total OI per exchange, active symbols per exchange, heatmap Δ × symbols × exchanges | `12 §4.5` | dashboard | 3h | Med |
| M5.A.9 | Alertmanager rules `oi-tracker.yml`: ConnectorDown, ConnectorHighErrorRate, SymbolCoverageLow, ClockSkewHigh, DBDiskUsageHigh, CARefreshFailing, AlertEngineSlow, E2ELatencyHigh, TGDeliveryBacklog, TGDeliveryFailing | `12 §7.2` | rules.yml | 2h | Low |
| M5.A.10 | Alertmanager routing → TG (либо тот же chat_id с тегом `[INFRA]`, либо dedicated chat) | `12 §7.1` | alertmanager.yml | 1h | Low |

**M5.A итого:** ~27h.

### M5.B — 7 runbook drills + hardening + backups skeleton

| # | Задача | Источник | Артефакт | Часы | Риск |
|---|---|---|---|---|---|
| M5.B.1 | Runbook drill 5.1 «Биржа упала» — induce DNS fail для одной биржи, verify circuit breaker open после 5 fails, проверить что остальные 11 продолжают, восстановить, замерить recovery time | `13 §5.1` | drill log | 1h | Low |
| M5.B.2 | Runbook drill 5.2 «БД упала» — `systemctl stop postgresql` 30s, verify scheduler/api/tg-sender падают и автоматически restart'уют через systemd, замерить recovery после `systemctl start postgresql` | `13 §5.2` | drill log | 1h | Med |
| M5.B.3 | Runbook drill 5.3 «TG sender stuck» — invalid TELEGRAM_BOT_TOKEN, induce fail, verify Alertmanager fires `TGDeliveryBacklog`, fix token, restart, verify recovery | `13 §5.3` | drill log | 1h | Low |
| M5.B.4 | Runbook drill 5.4 «Disk full» — fill /var/lib/postgresql to >80% (через `dd` в test directory), verify Alertmanager fires `DBDiskUsageHigh`, manual force compress old chunks | `13 §5.4` | drill log | 1.5h | Med (опасно — можно реально заполнить) |
| M5.B.5 | Runbook drill 5.5 «SSE отвалился» — kill nginx, verify UI reconnect badge, restart, verify реcоnnection | `13 §5.5` | drill log | 0.5h | Low |
| M5.B.6 | Runbook drill 5.6 «Фейл нормализации» — induce schema_drift в Binance fixture (replace field), verify `oi_normalize_errors_total` ↑, manual replay после fix | `13 §5.6` | drill log | 1.5h | Low |
| M5.B.7 | Runbook drill 5.7 «SLO breach» — induce slow normalize (artificial sleep), verify P99 latency > 180s, Alertmanager `E2ELatencyHigh`, mitigation (отключить consensus rule) | `13 §5.7` | drill log | 1h | Low |
| M5.B.8 | Security audit `13 §10.0`: `grep -RE 'verify\s*=\s*False' src/` → 0 строк; `grep -RE 'f["\'']' src/.*sql' --include='*.py'` или ruff S608 PASS; `journalctl -u oi-tracker-* | grep -iE '(token|password|secret|api[_-]?key)'` → пусто; `stat -c '%a' /etc/oi-tracker/oi-tracker.env` → 600; `pg_hba.conf` для `oi_tracker` без `trust`; `REVOKE ALL` на соседние БД verified через `psql -c "\du+ oi_tracker"` | `13 §10.0`, `15 §4`, `00 F14` | audit PASS | 2h | Med |
| M5.B.9 | Backups skeleton — `13 §9.1` план активации (RPO ≤ 24h / RTO ≤ 2h pg_dump nightly; RPO ≤ 1h WAL-G optional) — **зафиксировать в коде структуру** (`/var/backups/oi-tracker/` + cron-stub), но **не активировать** (Q7) | `13 §9`, `00 Q7` | backup playbook | 2h | Low |

**M5.B итого:** ~11.5h.

### M5.C — Final DoD

| # | Задача | Источник | Артефакт | Часы | Риск |
|---|---|---|---|---|---|
| M5.C.1 | 7 days production метрики без gap'ов — passive monitoring | `16 §M5 DoD` | metrics history clean | 168h calendar (~0h dev) | Low |
| M5.C.2 | Coverage final verification: ≥ 80% общий, ≥ 90% alert engine + normalizer + exchanges, ≥ 70% frontend | `16 §M5 DoD`, `14 §2.1` | coverage report | 1h | Low |
| M5.C.3 | decisions-guardian + verify-spec PASS на полном кодбейсе через subagents (`CLAUDE.md §11.1`) | `16 §M5 DoD` | reports PASS | 2h | Low |
| M5.C.4 | Final retrospective + `tasks/lessons.md` updates | `CLAUDE.md §2.5, §9` | lessons | 1h | Low |

**M5.C итого:** ~4h (в dev hours; calendar — 7 дней).

**M5 итого: ~42.5h** + 4.5h pre-M5 ADR + buffer ~10h = **~57h**.

---

## 13. Risk register (топ-12 для всего проекта)

| # | Риск | Вероятность | Impact | Контрмера | Где зафиксировано |
|---|---|---|---|---|---|
| R1 | **Schema drift одной из 12 бирж в M2 или после деплоя** | Высокая | Высокий | (a) Contract tests с реальными фикстурами. (b) `raw_exchange_events` хранятся 14 дней → replay при обновлении парсера. (c) `connector_version` + `normalization_version` версионирование. (d) Schema-drift тест per parser (`14 §4.5`). | `01 §9 R1`, `13 §5.6` |
| R2 | **TimescaleDB compression incompatibility / late insert в compressed chunk медленный** | Низкая (требуется TS ≥ 2.11) | Средний | `08 §13.2` `_ensure_timescaledb()` fail-fast на версии < 2.11. Late insert поддерживается из коробки (`08 §4.3`). Migration тесты проверяют compression up/down. | `08 §13.2, §4.3` |
| R3 | **aiogram rate-limit burst при consensus event (36+ алертов сразу)** | Средняя | Средний | Pre-M3 §7.1 leaky-bucket throttle (1 msg/sec per chat, 25/sec global) + retry policy с TelegramRetryAfter обработкой | `10 §2.5`, Pre-M3 |
| R4 | **SSE reconnect storms при network blip** | Средняя | Низкий | Frontend exponential backoff (1s → 30s cap, `10 §5.6`). PG NOTIFY не блокирующий — backend не падает от reconnect storm. | `10 §5.6`, `18 §7.6` |
| R5 | **60s polling cycle overlap при медленной БД (compression rebalance, vacuum)** | Средняя | Высокий | Pre-M2 ADR backpressure (`02 §5.5`): bounded queues + drop-oldest политика + метрики `oi_*_dropped_total`. Cycle_overlap detection в supervisor. | `02 §5.4, §5.5`, `13 §5.7` |
| R6 | **mypy --strict + asyncpg типизация (asyncpg не имеет статических типов)** | Средняя | Низкий | Wrapper модули в `app/db/` с явными pydantic-моделями на границе. `asyncpg.Record` → `NormalizedOISample.model_validate()`. Минимизировать прямой контакт с asyncpg API в бизнес-коде. | `02 §2.1.1` |
| R7 | **Frontend bundle размер > 250 KB gz target** | Средняя | Средний | (a) React.lazy для Settings и Symbol. (b) Recharts tree-shake (импорт only used components). (c) Без heavy date libs (date-fns instead of moment). (d) `rollup-plugin-visualizer` в CI как guard. | `18 §10`, `16 §M4 DoD` |
| R8 | **Single-user — нет резервного оператора при инциденте** | Высокая | Высокий | (a) 7 runbooks `13 §5` с конкретными шагами. (b) Phase D в UX-сценарии `01 §3.3` тренирует self-recovery. (c) Auto-restart через systemd `Restart=always`. (d) Alertmanager → TG с теми же caveat'ами. | `01 §3.3 Phase D`, `13 §5` |
| R9 | **Zero-downtime deploy без Docker** | Средняя | Низкий | Принимаем downtime ~10s при `systemctl restart` — single-user это переживёт. Frontend deploy через rsync — атомарен. Migrations совместимы (additive). Rollback — `git checkout` + restart (`13 §7.3`). | `13 §7`, `01 §6 NFR-3` |
| R10 | **Late-arriving data + retro-fire policy edge cases (smart_re_fire через minute boundary)** | Низкая | Средний | Тест `tests/e2e/test_no_retro_fire_after_replay.py` (M3.D.1). Alert engine работает только на «срезе вперёд» по `ts_exchange` (`F9`). Replay не вызывает alert engine. | `00 F9`, `09 §13.4` |
| R11 | **TimescaleDB extension установка задевает соседей `detector` + `grach-ege` (~30–60s downtime)** | Высокая (если не делается аккуратно) | Высокий | `15 §6.1` полный maintenance protocol: уведомление, pg_dump страховка, backup `postgresql.conf`, smoke test после restart. Откат через restore `postgresql.conf` (пакет inert без preload). | `15 §6`, `00 F17` |
| R12 | **TG bot token leak в логи / git** | Низкая | Высокий | (a) gitleaks в pre-commit + CI (`17 §3, §4`). (b) Sample audit `journalctl | grep -iE '(token|password|secret)'` перед merge (`13 §10.0`). (c) `.env chmod 0600` (F14). (d) Token в env, не в DB (Q8). | `13 §10.0`, `00 F14, Q8`, `17 §3` |

---

## 14. Critical path и параллелизация

### 14.1 Sequential dependencies

Между milestone'ами — строго последовательно, DoD-driven (`16 §0 принципы`). Нельзя начать M(N+1) без закрытия M(N) DoD:

```
Phase 0 ──▶ M1 ──▶ M2 ──▶ M3 ──▶ M4 ──▶ M5
   │         │      │      │       │      │
infra +    foundation │      │       │      │
binance    + CA + cleanup    │       │      │
                  12 connectors      │      │
                            alert engine    │
                                  + TG     │
                                       UI  │
                                       full obs
```

### 14.2 Параллелизация **внутри** milestone

| Milestone | Что параллелится | Сколько потоков |
|---|---|---|
| Phase 0 | Phase 0.A (infra) и Phase 0.B (code skeleton) — независимы | 2 |
| M1 | M1.A (CA migrations) и M1.B (normalizer) — независимы | 2 |
| **M2** | **11 коннекторов** через `exchange-adapter-builder` agent (см. `CLAUDE.md §11.1`); каждый коннектор изолирован, зависит только от готового `BaseConnector` (Phase 0) и normalizer (M1) | **до 4–5** (group: snapshot-similar = 7 connectors, native_interval = 2, bulk = 2, специфические = 2) |
| M3 | M3.A (state machine) + M3.B (signals) могут начать параллельно после M3.A.2 готов; M3.C (TG sender) требует только delivery_queue из M3.B.6 | 2 |
| M4 | **4 UI страницы** могут параллелиться (M4.C.1–4) при готовом M4.A (skeleton) и M4.B (backend SSE+REST); типичный split: 1 dev на dashboard + symbol, 1 dev на alerts + settings | 2 |
| M5 | 5 Grafana дашбордов M5.A.4–8 — независимы; 7 runbook drills M5.B.1–7 — независимы | 3 |

### 14.3 External dependencies (вне контроля dev)

| # | Зависимость | Когда | Mitigation |
|---|---|---|---|
| E1 | DNS A-запись `oi-tracker.robot-detector.ru` (владелец зоны) | Phase 0.A.1 | Запросить заранее, до старта Phase 0 |
| E2 | Maintenance window для restart PG (соседи `detector`, `grach-ege`) | Phase 0.A.6 | Согласовать за 24–48h, проводить ночью |
| E3 | TLS-cert через certbot — DNS должен резолвиться | Phase 0.A.12 | После E1 + 5 минут propagation |
| E4 | TG bot creation через @BotFather (owner action) | Phase 0.A.9 (placeholder), реально нужен в M3.C.4 | Сделать в Phase 0 параллельно с infra |
| E5 | Прогон 7 days production метрик для M5 DoD | M5.C.1 | Calendar time, 7 days после M5 кода-готов |

---

## 15. Total estimate

| Stage | Dev hours (estimate) | Calendar (full-time 40h/wk) | Calendar (part-time 20h/wk) |
|---|---|---|---|
| Phase 0 | 36h | ~5 дней | ~10 дней |
| Pre-M1 ADR (closed) | 0h | 0 | 0 |
| M1 | 30h | ~4 дня | ~8 дней |
| Pre-M2 ADR (pool sizes PR) | 1.5h | 0.5 дня | 1 день |
| M2 | 119h (60h при параллелизации 11 коннекторов в 2 потока) | ~7.5–15 дней | ~15–30 дней |
| Pre-M3 ADR (throttle + hot-reload) | 3.5h | 0.5 дня | 1 день |
| M3 | 67h | ~8.5 дней | ~17 дней |
| Pre-M4 ADR (SSE + NOTIFY policy) | 1.5h | 0.5 дня | 1 день |
| M4 | 125h | ~16 дней | ~32 дня |
| Pre-M5 ADR (trace_id) | 4.5h | 0.5 дня | 1 день |
| M5 | 57h (+ 7 calendar days passive monitoring) | ~7 дней + 7 calendar | ~14 дней + 7 calendar |
| Buffer 15–18% | ~70h | ~9 дней | ~18 дней |
| **Total** | **~516h** (~408h при максимальной параллелизации M2) | **~12 недель** | **~24 недели** |

**Реалистичный target для одного senior-dev в full-time:** ~3 месяца от Phase 0.A.1 до M5.C.4 (включая 7 days passive monitoring в M5.C.1).

В режиме part-time (20h/неделя): ~6 месяцев.

---

## 16. Open questions / explicit assumptions

### 16.1 Open (требуют решения до начала соответствующего milestone)

1. **Pre-M2 §5.3 connection pool sizes** — нужен микро-PR в `02 §5.3` фиксирующий точные значения (рекомендация: scheduler=20, api=10, tg-sender=5, cleanup=2). **Решение:** делается в Pre-M2 ADR-PR (1.5h работы).
2. **Pre-M3 §7.1 aiogram throttle** — leaky-bucket алгоритм + точные пороги (1/25 msg/sec). **Решение:** PR в `10 §2.5`.
3. **Pre-M3 §7.2 settings hot-reload** — TTL-cache 30s vs PG NOTIFY. **Senior-рекомендация:** TTL-cache (проще, для single-user 30s latency приемлема). PR в `10 §4.8`.
4. **Pre-M4 §9.1 SSE no-replay disclaimer** — явная фиксация поведения при reconnect. PR в `10 §5`.
5. **Pre-M4 §9.2 PG NOTIFY 8KB limit policy** — явный запрет расширения payload без F-номера. PR в `10 §5.5`.
6. **Pre-M5 §11.1 trace_id propagation** — формат (cycle_id из connector → end-to-end через все слои). PR в `12 §5`.

### 16.2 Assumptions (если не подтверждены — могут потребовать пересмотра)

- **A1.** Хост-машина имеет ≥ 200 GB SSD свободно (`13 §1.1`). При меньшем — ужесточить retention до 60 дней.
- **A2.** PG max_connections = 100 default (стандарт). Если соседи занимают > 60 — придётся менять `postgresql.conf` (требует maintenance window).
- **A3.** TG bot создаётся одним пользователем (single-user), без multi-bot routing.
- **A4.** OKLCH browser support не блокирует пользователя (Chrome 111+, Safari 16.4+, Firefox 113+ — current LTS-relevant). См. `18 §15.2`.
- **A5.** 12 бирж имеют публичные OI endpoints без обязательной авторизации в течение всего срока разработки (3 месяца). См. `01 §8 C-4`.
- **A6.** `app/tools/capture_fixtures.py` существует или пишется как часть Phase 0.E.5 — для capture реальных payloads под `tests/contract/<exchange>/fixtures/`.
- **A7.** `ruff` правило `S608` (f-string в SQL) — включено в `[tool.ruff.lint]` select list (см. `17 §1.2` уже включает `S` (bandit) набор).
- **A8.** Loki / Grafana / Prometheus — устанавливаются как обычные ubuntu пакеты (`13 §2.1.12`); если нужна dockerized stack — это **conflict** с `Q6` (no Docker), нужно решение.

---

## 17. Next action

После approval этого плана — **первый PR**:

**PR-001: «Phase 0.A.1–A.7 — host infrastructure preparation»**

Включает:
1. DNS A-запись `oi-tracker.robot-detector.ru` (E1 — внешняя задача владельца).
2. Согласование maintenance window с владельцами `detector` и `grach-ege` (E2).
3. `pg_dump` соседей в `/root/backups/`.
4. Установка TimescaleDB extension в PG16 кластер (`apt install` + `timescaledb-tune` + restart).
5. `CREATE DATABASE oi_tracker` + role + REVOKE на соседние БД + `CREATE EXTENSION timescaledb`.
6. Smoke test соседей.

**Артефакт PR-001:** `docs/13_OPERATIONS.md §2.1.2` команды выполнены, фиксация в `tasks/todo.md` секции «Phase 0 — Infra». Никакого кода `oi-tracker` ещё не пишется.

**После PR-001 → PR-002:** «Phase 0.A.8–A.14 — OS user, env, nginx, TLS, frontend placeholder» (без коду application'а).

**После PR-002 → PR-003:** «Phase 0.B — code skeleton (pyproject, structure, config, logging, contracts)» — первый PR с Python-кодом.

Дальше следовать декомпозиции из §2 этого плана, по одной задаче за PR (или группами 2–3 связанных задач), с обязательным DoD-чек-листом перед merge.

---

## Релевантные файлы (всё абсолютные пути)

- `/var/www/oi-tracker/docs/00_DECISIONS_LOG.md`
- `/var/www/oi-tracker/docs/01_PRODUCT_SPEC.md`
- `/var/www/oi-tracker/docs/02_ARCHITECTURE.md`
- `/var/www/oi-tracker/docs/05_DATA_CONTRACTS.md`
- `/var/www/oi-tracker/docs/07_NORMALIZER.md`
- `/var/www/oi-tracker/docs/08_TIME_SERIES_STORAGE.md`
- `/var/www/oi-tracker/docs/09_ALERT_ENGINE.md`
- `/var/www/oi-tracker/docs/10_DELIVERY_LAYER.md`
- `/var/www/oi-tracker/docs/11_EXCHANGE_ADAPTERS/_BASE_CONNECTOR.md` + 12 файлов
- `/var/www/oi-tracker/docs/12_OBSERVABILITY_SLO.md`
- `/var/www/oi-tracker/docs/13_OPERATIONS.md`
- `/var/www/oi-tracker/docs/14_TEST_STRATEGY.md`
- `/var/www/oi-tracker/docs/15_DEPLOYMENT_INFRA.md`
- `/var/www/oi-tracker/docs/16_ROADMAP.md`
- `/var/www/oi-tracker/docs/17_TOOLING.md`
- `/var/www/oi-tracker/docs/18_DESIGN_SYSTEM.md`
- `/var/www/oi-tracker/CLAUDE.md`

Этот план готов к сохранению в `/var/www/oi-tracker/tasks/development_plan.md` после approval.