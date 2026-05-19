# 16 · Roadmap

> Структура milestone'ов разработки `oi-tracker`. Phase 0 (scope первого PR) — **TBD** до отдельного решения. Milestones M1–M5 фиксируют последовательность построения системы.
> Связано: `00_DECISIONS_LOG.md`, `01_PRODUCT_SPEC.md`, `15_DEPLOYMENT_INFRA.md` (deployment checklist), `14_TEST_STRATEGY.md`.

---

## 0. Принципы roadmap

1. **Production-grade с самого начала.** Это не MVP-проект, см. `01_PRODUCT_SPEC.md`. Каждый milestone закрывается полным DoD из `CLAUDE.md §4`.
2. **Vertical slices.** Каждый milestone доставляет работающую функциональность end-to-end (биржа → БД → API → UI/TG), а не «всё backend, потом всё frontend».
3. **Single-user, без feature-flag'ов.** Не катаем «частично включённую» функциональность. Готово = задеплоено и используется.
4. **DoD before move on.** Не идём в M(N+1), пока M(N) не закрыт по Definition of Done. Отказ от этого правила = накопление техдолга.
5. **Roadmap пересматривается** после каждого milestone в feedback-loop (см. `CLAUDE.md §2.5`).

---

## Phase 0 · Skeleton (первый PR)

**Status:** Decided 2026-05-01.

**Решение по 4 вопросам:**

- [x] **Scope первого PR:** vertical slice «sample collector + observability skeleton + deploy infra». Включает:
  - Установка TimescaleDB extension по `15_DEPLOYMENT_INFRA §6.1` (одноразовая host-операция; согласована с владельцами соседей `detector`, `grach-ege`).
  - Deployment checklist `15_DEPLOYMENT_INFRA §7` выполнен полностью (OS user, paths, env-файл `chmod 0600`, dedicated PG database+role, nginx site + cert на `oi-tracker.robot-detector.ru`).
  - `pyproject.toml` + структура backend по `02_ARCHITECTURE §4`.
  - **Только** Alembic-миграция #1: `oi_samples` hypertable + compression + retention (по шаблону `08_TIME_SERIES_STORAGE §13.3`). CA `oi_5m/15m/30m` — отдельная миграция в M1.
  - `BaseConnector` интерфейс + Binance connector (только `fetch_instruments` + `fetch_oi_snapshot`).
  - Scheduler async loop, polling 60s aligned to minute (см. `00_DECISIONS_LOG C2`).
  - Normalizer-skeleton: маппинг raw → `OiSample` без price attachment / quality assessor (всё это — M1, тут только pipeline-каркас).
  - Storage layer: bulk upsert по шаблону `08 §4.1`.
  - `GET /api/v1/health` (liveness + DB connectivity) на UDS `/run/oi-tracker/api.sock` (см. F14).
  - `/metrics` endpoint для Prometheus (заглушка — без бизнес-метрик, только `process_*` от prometheus_client).
  - systemd units `oi-tracker-scheduler` + `oi-tracker-api` (без `tg-sender` — TG в M3, без `cleanup` — он добавится в M1 c retention/CA refresh).
  - structlog в `/var/log/oi-tracker/*.log` + journald, logrotate.
  - pre-commit (ruff format + ruff check + mypy --strict + pytest --quick).
- [x] **Первая биржа:** Binance (самая документированная, snapshot-source, низкий риск schema drift).
- [x] **Инфра-команды:** полный `15_DEPLOYMENT_INFRA §7` checklist. Без сокращений.
- [x] **Тесты с самого начала:** да. Минимум для Phase 0:
  - Unit: `BaseConnector` + Binance parser (mapping, edge cases).
  - Contract: 1 happy-path фикстура из live-payload Binance.
  - Integration: один прогон scheduler-cycle на test-БД с testcontainers, проверка что row появилась в `oi_samples`.
  - Migration: `alembic upgrade head` → `alembic downgrade base` → `alembic upgrade head` на чистой test-БД (см. `08 §13.5`).

**Что НЕ входит в Phase 0** (всё это — M1+):
- Continuous aggregates (CA `oi_5m/15m/30m`, `oi_quality_5m`).
- Бизнес-метрики Prometheus (`oi_collector_*`, `oi_storage_*`). В Phase 0 — только `/metrics` skeleton, без custom collector'ов.
- Alert engine, TG sender, UI.
- Остальные 11 коннекторов.
- Instruments registry sync-job (в Phase 0 список инструментов один раз при старте, sync-cycle добавится в M2).

**Definition of Done (Phase 0):**
- [ ] DoD из `CLAUDE.md §4.1` выполнен.
- [ ] За 30 минут непрерывной работы scheduler собрал ≥ 25 циклов (≥ 83% coverage по времени; полный SLO 92% — в M1).
- [ ] `oi_samples` содержит данные по всем активным USDT-perp Binance-символам.
- [ ] `curl --unix-socket /run/oi-tracker/api.sock http://localhost/api/v1/health` → 200 OK.
- [ ] `curl https://oi-tracker.robot-detector.ru/` → 200 + `X-Robots-Tag: noindex, ...` (заглушка index.html достаточно).
- [ ] `alembic upgrade head` + `alembic downgrade base` отрабатывают на чистой БД.
- [ ] Contract test для Binance проходит.
- [ ] mypy --strict, ruff — clean.
- [ ] Соседи (`detector`, `grach-ege`) не пострадали (smoke test после nginx reload и после restart PG).

---

## M1 · Foundation: storage + 1 connector + observability skeleton

**Цель:** один коннектор Binance собирает OI каждую минуту, пишет в TimescaleDB, экспортирует базовые метрики, систему можно мониторить.

**Включает:**
- Деплой-инфра выполнена по `15_DEPLOYMENT_INFRA.md §7`.
- pyproject.toml + структура проекта по `02_ARCHITECTURE.md §4`.
- Alembic-миграции: `oi_samples` (hypertable + compression policy + retention), `instruments`, `connector_errors`, `settings`, `alembic_version`.
- `BaseConnector` интерфейс + Binance connector (только `fetch_instruments` + `fetch_oi_snapshot`).
- Scheduler как async loop, polling 60s aligned to minute (см. `00_DECISIONS_LOG.md C2`).
- Normalizer-лайт: маппинг raw → `OiSample` без alert engine.
- Storage layer: bulk insert oi_samples с UPSERT.
- Health endpoint `GET /api/v1/health` (только liveness + DB connectivity).
- Prometheus metrics endpoint на `/metrics` (FastAPI prometheus-client).
- systemd-юниты `oi-tracker-scheduler` + `oi-tracker-api` (без tg-sender).
- structlog с outputting в `/var/log/oi-tracker/*.log` + journald.
- Logrotate сконфигурирован.
- pre-commit (ruff + mypy + pytest --quick).

**Definition of Done (M1):**
- [ ] DoD из `CLAUDE.md §4.1` выполнен.
- [ ] За 1 час непрерывной работы scheduler собрал ≥ 55 циклов (≥ 92% coverage по времени).
- [ ] `oi_samples` содержит данные по всем активным USDT-perp Binance-символам, `valuation_status` корректен.
- [ ] `/metrics` отдаёт `oi_collector_request_total{exchange="binance"}`, `oi_collector_cycle_duration_seconds`, `oi_storage_insert_duration_seconds`.
- [ ] Contract test для Binance проходит.
- [ ] Coverage normalizer + storage ≥ 80%.
- [ ] mypy --strict, ruff — clean.
- [ ] Sample query `SELECT exchange, count(*), max(ts_exchange) FROM oi_samples GROUP BY 1` работает и показывает свежие данные.
- [ ] Соседи (`detector`, `grach-ege`) не пострадали.

---

## M2 · Все 12 коннекторов + normalizer полный + instrument registry

**Цель:** все 12 бирж активны, normalizer корректно конвертирует все типы OI (contracts × contract_size, native notional, oi_coins прямо), instruments registry синкается каждые 5 минут.

**Включает:**
- 11 оставшихся коннекторов по `docs/11_EXCHANGE_ADAPTERS/<exchange>.md` через subagent `exchange-adapter-builder`:
  - Bybit + KuCoin: native_interval primary path (см. `00_DECISIONS_LOG.md C12`).
  - XT: bulk-snapshot endpoint.
  - Остальные 8: snapshot per symbol.
- Полный normalizer pipeline (`07_NORMALIZER.md`) с `valuation_status` и `source_kind`.
- Instruments registry с sync-job каждые 5 минут (`F2`), `delisted` flag, `blacklisted`.
- Per-exchange circuit breaker (`F3`).
- Exchange-health detection (метрики `oi_collector_symbol_coverage_ratio`, `oi_collector_stale_seconds`).
- Replay-tool stub (`13_OPERATIONS.md §6.2`) — командой можно реплейнуть, retro-fire отсутствует.

**Definition of Done (M2):**
- [ ] Все 12 коннекторов проходят contract-тесты (минимум: 1 happy path + 1 error case + mapping test).
- [ ] Coverage по `app/exchanges/*` ≥ 90%.
- [ ] За 1 час `symbol_coverage_ratio ≥ 95%` для каждой биржи.
- [ ] decisions-guardian отчёт PASS.
- [ ] verify-spec PASS (валидация имён, типов, `canonical_symbol = BASE`, `oi_coins` как primary).
- [ ] mypy --strict / ruff чисто.
- [ ] Replay-tool работает на синтетическом backfill.

---

## M3 · Alert engine + Telegram delivery

**Цель:** все 5 типов алертов работают, smart cooldown, доставка в Telegram, deduplication, retry с backoff.

**Включает:**
- `alert_rules` table + 6 default-строк (up/down × 5/15/30) per `F6`.
- State machine `idle → armed → fired → cooldown → armed` (`C6`).
- `confirm_points` поддерживается (default 1).
- Smart cooldown (`Q1 + F7`): re-fire при Δ ≥ 1.5× previous.
- Min history: `max(30, 2 × window)` (`F8`).
- Все 5 типов сигналов (`Q2`):
  - Threshold + Confirmed.
  - Consensus: ≥ M бирж одновременно.
  - Divergence: OI ↑ + price ↓ (или наоборот).
  - Exchange-health: stale_seconds, coverage_ratio_drop.
- `delivery_queue` + `oi-tracker-tg-sender.service`.
- aiogram 3.x sender-only с retry policy (`10_DELIVERY_LAYER.md §2.5`).
- TG templates: `oi_threshold_cross`, `oi_threshold_cross_smart_refire`, `divergence_alert`, `exchange_health_alert` (`10 §3`). _consensus_alert sunset per `00 F40`._
- `tg_dry_run` toggle через settings.
- Reprocessing не создаёт retro-fire (`F9`) — тест на это.

**Definition of Done (M3):**
- [ ] Все 5 типов алертов проверены на синтетических потоках (unit + integration).
- [ ] State machine — coverage ≥ 95% (это критично).
- [ ] Smart cooldown — отдельный тест на каждый сценарий (re-fire разрешён, re-fire запрещён, истечение cooldown).
- [ ] TG sender — contract test с mocked aiogram, retry policy verified.
- [ ] `oi_alert_e2e_latency_seconds` histogram присутствует и обновляется.
- [ ] Boot test: chat_id = test группа, dry_run=false, `tg_threshold_cross` приходит ≤ 90s после induced delta.
- [ ] decisions-guardian PASS.

---

## M4 · Web UI: dashboard + symbol page + alerts log + settings

**Цель:** все 4 страницы UI работают, SSE-real-time обновления, full UI Settings.

**Визуальный язык и UX-требования:** `18_DESIGN_SYSTEM.md`. Все компоненты, цвета, типографика, motion и acceptance UX-критерии — оттуда. M4 не считается closed без выполнения §13 design-system (acceptance for M4).

**Включает:**
- React + Vite + TS strict.
- Дизайн-токены (OKLCH CSS variables, Tailwind-конфиг) по `18_DESIGN_SYSTEM.md §2, §12`.
- Типографика (Inter + JetBrains Mono, self-host) по `18_DESIGN_SYSTEM.md §3`.
- 4 страницы (`10 §6.1`): Dashboard, Symbol page, Alerts log, Settings.
- TanStack Table + virtualization для live-таблицы (3000+ строк).
- Recharts для графиков symbol page.
- SSE через `EventSource` + reconnect backoff (`10 §5.6`).
- PG NOTIFY/LISTEN bridge для push events (`10 §5.5`).
- REST endpoints (`10 §4`): `/live`, `/symbols/:c`, `/alerts`, `/alert-rules`, `/settings`, `/instruments`, `/health`.
- Full UI Settings (расширенный список — см. `10 §6.1` после M3-обновления): пороги, cooldown defaults, smart_cooldown_factor, freshness_budgets, min_oi_notional_default, blacklist/whitelist, chat_id, dry_run, force re-sync.
- Rate-limit на PUT /settings (см. M3-обновление 10).
- Frontend deployed в `/var/www/oi-tracker/frontend-dist`, отдаётся nginx'ом.
- noindex headers + meta + robots.txt verified live.
- Дашборд показывает SSE connection status badge.

**Definition of Done (M4):**
- [ ] Все 4 страницы проходят golden-path manual smoke test.
- [ ] `curl -I https://oi-tracker.robot-detector.ru/` → 200 + `X-Robots-Tag: noindex, ...`.
- [ ] SSE: 1-минутный smoke test без drop'ов.
- [ ] UI на 3000 строк дашборда — рендерит без UI-jank (TanStack virtualization работает).
- [ ] Frontend coverage (компонент-тесты) ≥ 70%.
- [ ] mypy / ruff / eslint / tsc чисто.
- [ ] verify-spec PASS.
- [ ] **Acceptance из `18_DESIGN_SYSTEM.md §13` пройден** (no layout shift, tabular alignment, connection status, freshness, density toggle, theme toggle, reduced-motion, keyboard nav, empty/loading/error states, bundle ≤ 250 KB gz).

---

## M5 · Observability complete + runbooks verified + hardening

**Цель:** Prometheus + Grafana + Loki полностью настроены, все Alertmanager rules активны, runbooks проверены drill'ом, security hardening завершён.

**Включает:**
- Все Prometheus метрики per `12 §3.1–3.7`.
- 5 Grafana dashboards (`12 §4`): System, Exchanges, Alerts, Latency, Symbols.
- Loki labels стандартизированы (`12 §5.2`).
- Alertmanager rules (`12 §7.2`): SLO breach, ConnectorDown, TGDeliveryBacklog, DBDiskUsageHigh, ExchangeStale, и т.д.
- Все 7 runbooks (`13 §5`) пройдены drill'ом — для каждого зафиксировано фактическое время реакции.
- Security hardening:
  - `pg_hba.conf` audit.
  - env-файл chmod/chown verified.
  - Все исходящие httpx без `verify=False` (grep'ом).
  - Logs не содержат secrets (sample audit).
  - PG `oi_tracker` role не имеет CONNECT на соседние БД.
- Backups skeleton с RPO/RTO target зафиксирован в `13 §9` (даже если не активирован).

**Definition of Done (M5):**
- [ ] Production метрики собираются 7 дней без gap'ов.
- [ ] SLO 90s P95 / 180s P99 выдерживается.
- [ ] Coverage по всему проекту: ≥ 80% (общее), ≥ 90% (alert engine, normalizer, exchanges).
- [ ] Каждый runbook отрепетирован один раз с фиксацией времени.
- [ ] Security audit (chmod, env, verify=False, logs) — PASS.
- [ ] decisions-guardian + verify-spec — PASS на полном кодбейсе.

---

## M6 · Integration test backfill (closed — path Y partial)

**Цель:** закрыть test debt по storage repos, для которых unit-моки не дают реалистичной гарантии (deduplication, ON CONFLICT, FOR UPDATE SKIP LOCKED, hypertable behaviour).

**Scope (path Y per `tasks/todo.md` M6 section).** Integration harness + 3 critical I/O repos. Read-only / CRUD repos (`live_dashboard`, `symbol_history`, `oi_bars`, `instruments`, `settings_kv`, `oi_samples`, `api/routes/*`) интенциально оставлены вне M6 — covered E2E через UI; полный 80% literal был оценён как сигнал «уважаем metrics» без реального ROI на single-user продукте.

**Сделано:**

- `testcontainers[postgres]>=4.8` + `psycopg2-binary` (dev-only) — sync alembic в test fixture.
- `tests/integration/conftest.py` — session-scope TimescaleDB container + alembic upgrade один раз; function-scope asyncpg pool + per-test `TRUNCATE RESTART IDENTITY CASCADE` + reseed defaults (4 instruments, 6 alert rules per F18). CA matviews пропускаются (re-materialise from source).
- 47 integration tests (`tests/integration/storage/`):
  - `test_alert_events_repo.py` (17): append + bulk_append + query_alerts (every filter axis + DESC + pagination) + count_alerts.
  - `test_alert_state_repo.py` (13): get + get_for_rule + get_or_create bootstrap + upsert + bulk_upsert + count + FK CASCADE on rule delete.
  - `test_delivery_queue_repo.py` (21): enqueue + dedupe ON CONFLICT + JSONB roundtrip + fetch_pending lifecycle (`FOR UPDATE SKIP LOCKED`, retry budget, `next_retry_at` gating, ORDER BY id, limit) + mark_sending/sent/failed + schedule_retry (incl. throttle no-increment branch) + pending_count + reset_stale_sending + oldest_pending_age.
- F-records: `00_DECISIONS_LOG.md F34` (harness strategy), F35 (psycopg2 dev-dep), F36 (CA skip in reset).
- `pyproject.toml`: `fail_under = 69` (locks new unit+integration baseline).

**Definition of Done (M6 — path Y):**

- [x] `pytest -m integration` PASS — 67/67 (3 alert_engine pipeline + 1 migrations + 4 harness smoke + 47 storage repo tests + 12 from earlier suites).
- [x] Coverage uplift: 65.88% → 69.36% total backend; alert_state 0%→100%, delivery_queue 99%, alert_events 86%.
- [x] mypy --strict app/ — clean (97 source files).
- [x] ruff check + format — clean.
- [x] M6 closure note + retrospective entry в `tasks/lessons.md`.

**Что НЕ покрыто и почему (path Y choice):**

- API routes integration (M6.C in original plan): отложено — covered E2E через UI и existing unit tests с mocked repos.
- Read-only repos (M6.B.5–B.10): отложено по той же причине.
- Scheduler integration (M6.D): отложено — `loop.py` и `supervisor.py` декомпозированы в unit tests; реальное E2E поведение verifies через runbook drills (M5.B).
- pytest-xdist: отложено до момента, когда test suite > 5 min (сейчас 68s full).

Эти пункты могут быть подняты в M7+ при появлении конкретного debug-инцидента, требующего воспроизведения через integration test, или если CI infrastructure добавится и потребует formal coverage gates.

---

## M7 · Multi-chat routing (closed — 2026-05-02)

**Цель:** заменить single-value `settings.tg_chat_id` на per-rule routing с обязательным дефолтным fallback'ом, sync getChat-валидацией на create + hourly background revalidation, observability surface для health/blocked/resolution-failed signals.

**Сделано (4 атомарных коммита):**

- **M7.A** (b0e4d14, ~6h): F37 ADR в `00_DECISIONS_LOG.md`. Migration `0012_tg_chats.py` — `tg_chats` таблица с partial unique index `WHERE is_default=true`, `alert_rules.chat_id` FK ON DELETE RESTRICT, `delivery_queue.chat_id_native` snapshot. `app/domain/chats.py` (TgChat + ChatValidationError + ChatResolutionFailReason). `app/storage/repositories/tg_chats.py` (CRUD + atomic set_default + default-protect). docs/05 §8a + §8/§10/§12 обновлены. 22 repo integration tests + 4 migration round-trip tests.
- **M7.B** (3f0def1, ~5h): `chat_validator.py` (sync getChat over aiogram, 4 error categories), `chat_resolver.py` (rule → default fallback chain), producer + delivery_queue + dispatcher refactor на per-row routing, `revalidation.py` hourly loop с jitter, 5 новых Prom метрик. 22 новых тестов (validator + resolver + revalidation + producer-routing).
- **M7.C** (7d459ee, ~7h): REST `/api/v1/chats` (GET / POST + sync-validation / PATCH / set-default / DELETE soft / test-send), `alert_rules` API c chat_id tri-state, `settings/tg_chat_id` legacy alias с RFC 8594 Deprecation/Sunset/Link headers. Frontend Settings → chats panel + Alerts → chat dropdown. 14 API integration tests (httpx ASGI + slowapi reset).
- **M7.D** (30ea05d, ~3h): docs/12 §3.6 — 5 новых метрик + interpretation. 2 Prom alerts (`OiTgChatUnhealthy` warning 10m, `OiTgChatResolutionFailed` critical 5m). Grafana panel `Telegram chats (M7)` в alerts.json. Runbooks §5.8 (chat unhealthy) + §5.9 (default chat missing).
- **M7.E** (~3h): decisions-guardian PASS, code-reviewer (2 HIGH fixed: 422 vs 404 для inactive set_default + missing TG error markers; 1 HIGH out-of-scope для pre-M7 schedule_retry SQL), verify-spec PASS на Pydantic↔DDL↔docs/05. Live deploy на prod БД: migration `0012` clean, default chat seeded из env (settings_kv не содержал `tg_chat_id` → manual INSERT, runbook §5.9 покрывает этот path), все 4 systemd services restarted и active, API health ok, test-send delivered.

**Definition of Done:**

- [x] `pytest` 745/745 PASS, mypy --strict (103 source files), ruff check + format clean.
- [x] Coverage 71.39% (gate 69%, +2.03pt от M6 baseline). New modules: chat_resolver 100%, tg_chats 97%, chat_validator 96%, chats domain 90%, revalidation 64%.
- [x] Frontend: pnpm typecheck + lint + build clean (gz bundles < 250 KB), vitest 62/62 PASS.
- [x] Migration up + down idempotent + tested round-trip + downgrade preserves default to settings.tg_chat_id.
- [x] Prom rules валидируют через yaml load (promtool недоступен на host).
- [x] Runbook coverage: 2 новых runbooks (chat unhealthy, default missing).
- [x] F-record F37 в 00_DECISIONS_LOG.md.
- [x] Live deploy + smoke PASS — alert engine cycles 30k candidates / 0 errors, test delivery sent с правильным chat_id_native снапшотом.

**Surprises (см. tasks/lessons.md):**

- `from __future__ import annotations` несовместим с slowapi-декорированными FastAPI handler'ами (body-inference ломается). Lesson зафиксирован.
- Migration backfill на category-A KV settings (`F23` env-инициализируется) пропускает default-create когда ключ хранится **только** в env. Manual seed через legacy alias или прямой INSERT — стандартный путь восстановления, runbook §5.9 покрывает.

**Estimate vs actual:** ~24h plan vs ~24h actual (M7.A 6h + M7.B 5h + M7.C 7h + M7.D 3h + M7.E 3h). Совпадение точное — единственный milestone без overestimate с момента M5.

---

## Что после M7 (не roadmap, а future work)

- Бэкапы (pg_dump nightly + WAL-G), активация по `13 §9` (`Q7`, `J6`).
- USDC-M / COIN-M.
- Дополнительные биржи.
- Funding rate, liquidations.
- См. `01_PRODUCT_SPEC.md §10`.

---

## Зависимости между milestones

```
[Infra checklist 15 §7]
        │
        ▼
   [Phase 0]  ──── skeleton + Binance + deploy + tests
        │
        ▼
       [M1]
        │
        ▼
       [M2] ──── зависит от M1 (storage + normalizer базовый)
        │
        ▼
       [M3] ──── зависит от M2 (нужны все 12 source'ов и качество данных)
        │
        ▼
       [M4] ──── зависит от M3 (UI показывает алерты + SSE из NOTIFY)
        │
        ▼
       [M5] ──── зависит от M4 (полная observability + drill всех путей)
```

Параллелизация **внутри** milestone — допустима (например, в M2: 12 коннекторов в параллельных PR через `exchange-adapter-builder`). Параллелизация **между** milestone — не рекомендуется, теряется DoD-дисциплина.

---

## Pre-milestone prerequisites (закрыть до старта milestone)

Серые зоны из архитектурного аудита 2026-05-01 — не блокируют M1, но должны быть зафиксированы решениями (ADR / правкой docs) **до старта** соответствующего milestone, чтобы избежать improvisation в коде:

| Pre-Milestone | Что закрыть | Обоснование | Где зафиксировать |
|---|---|---|---|
| **pre-M1** | ORM-стратегия: asyncpg для hot-path bulk insert, SQLAlchemy Core для не-hot мутаций в API; **без** SQLAlchemy ORM session. | `mypy --strict` + Pydantic v2 + SQLAlchemy 2.x ORM требует декораций `Mapped[]` на каждую колонку. Нужно избежать двух стилей в коде. | `02_ARCHITECTURE §2.1` ADR. |
| **pre-M2** | Scheduler concurrency model: один supervisor, 12 connector tasks через `asyncio.TaskGroup`, изоляция падений per task. | В M1 один connector — модель не критична. С M2 (12 параллельных) выбор `gather` vs `TaskGroup` vs ручной supervisor влияет на error isolation (`F3`). | `02_ARCHITECTURE §5.4` ADR. |
| **pre-M2** | Backpressure: bounded `asyncio.Queue(maxsize=...)` между normalizer и storage + метрика `oi_storage_write_queue_depth`. | При медленной БД 60s-цикл может накрыть предыдущий → cycle_overlap. Bounded queue + drop-oldest политика — контрмера. | `02_ARCHITECTURE §5.5` + метрика в `12 §3.4`. |
| **pre-M2** | Connection pool sizes per-service: scheduler 20, api 10, tg-sender 5, cleanup 2 (= 37 connections + запас для соседей при PG `max_connections=100`). | 4 процесса × default pool без согласования = риск исчерпания. | `13_OPERATIONS §1.x` или `02 §5.3`. |
| **pre-M3** | aiogram throttle: leaky-bucket перед `bot.send_message` (1 msg/sec per chat, 25/sec global). | Telegram API global rate limit не учтён в `10 §2.5`; одновременные threshold-fire'ы на 12 биржах × 3 окна могут породить burst 36+ алертов и серию 429. | `10_DELIVERY_LAYER §2.5` дополнение. |
| **pre-M3** | Settings hot-reload: PG NOTIFY `settings_changed` или TTL cache 30s в каждом сервисе. | Без механизма scheduler/alert-engine не видят правки `tg_chat_id`/`tg_dry_run` из UI до рестарта. | `10_DELIVERY_LAYER §4.8` + механизм в `02 §5.3`. |
| **pre-M4** ✓ Decided | SSE no-replay disclaimer: «при reconnect клиент re-fetches `/api/v1/live` snapshot, gap не воспроизводится». | `10 §5.3` упоминает per-subscriber queue maxsize 1000, но не описывает поведение при disconnect. | `00_DECISIONS_LOG F27` + `10_DELIVERY_LAYER §5.5.2`. |
| **pre-M4** ✓ Decided | PG NOTIFY payload limit (8 KB): «payload содержит только keys, full record fetched отдельным API-запросом». | Превышение поднимает `ERROR: payload string too long` и прерывает INSERT — следующий разработчик мог бы расширить payload и сломать hot path. | `00_DECISIONS_LOG F28` + `10_DELIVERY_LAYER §5.5.1`. |
| **pre-M5** | Trace context propagation: `trace_id = cycle_id from connector`, наследуется через sample → alert → tg-sender. Каждый log entry содержит `trace_id`. | Дебаг «почему этот алерт пришёл с задержкой 87s» без сквозного trace_id занимает часы. | `12_OBSERVABILITY_SLO §5` дополнение. |

Ответственный: владелец проекта (single-user) принимает решение каждой строки одним документационным PR на старте соответствующего milestone.

---

## Cross-references

- `00_DECISIONS_LOG.md` — источник истины по всем архитектурным решениям, на которые опирается roadmap.
- `01_PRODUCT_SPEC.md §3, §10` — что в scope, что в future work.
- `14_TEST_STRATEGY.md` — coverage targets per module.
- `15_DEPLOYMENT_INFRA.md §7` — pre-M1 infra checklist.
- `CLAUDE.md §2.4 + §4` — что значит DoD.
