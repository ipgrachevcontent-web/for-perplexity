# CLAUDE.md — oi-tracker

> Этот файл — основной набор правил для Claude Code в проекте `oi-tracker`. Любые
> личные глобальные правила пользователя (`~/.claude/rules/**`) применяются
> поверх стандартного поведения, но **этот файл имеет приоритет** над ними при
> конфликте, потому что описывает специфику этого продукта.

---

## 0. Иерархия приоритетов при конфликте правил

При противоречии любых правил между собой — побеждает то, что выше в этом списке:

1. **Безопасность** (секреты, валидация, инъекции, TLS).
2. **Корректность** (поведение соответствует спецификации в `docs/`).
3. **SLO / Performance** (бюджеты latency и throughput из `docs/01_PRODUCT_SPEC.md` NFR).
4. **Поддерживаемость** (читаемость, тесты, observability).
5. **Элегантность** (минимум абстракций при той же ясности).
6. **Краткость** (минимум кода).

Это решает конфликты вида «минимум кода vs элегантность», «починить молча vs уточнить
допущение», «не рефакторить заодно vs элегантнее переписать».

---

## 1. База знаний проекта (`docs/`)

`docs/` — единственный источник истины по архитектуре и решениям. Код должен ему
соответствовать; при расхождении — **docs приоритетнее кода**.

### 1.1 Карта документов

| Файл                                  | О чём                                                                  |
| ------------------------------------- | ---------------------------------------------------------------------- |
| `docs/00_DECISIONS_LOG.md`            | Финальные решения C1–C17 + Q1–Q8 + Fxx. **Читать в начале любой задачи.** |
| `docs/01_PRODUCT_SPEC.md`             | Что строим, NFR, SLO.                                                  |
| `docs/02_ARCHITECTURE.md`             | Компоненты и потоки данных.                                            |
| `docs/03_GLOSSARY.md`                 | Термины (canonical_symbol, oi_coins, valuation_status и др.).          |
| `docs/04_TIME_MODEL.md`               | `ts_exchange` / `ts_ingested` / `ts_processed`, freshness формулы.     |
| `docs/05_DATA_CONTRACTS.md`           | Pydantic / DDL контракты. **Менять — через PR в этот файл.**           |
| `docs/06_INSTRUMENT_REGISTRY.md`      | Реестр инструментов, sync 5 минут.                                     |
| `docs/07_NORMALIZER.md`               | Pipeline нормализации.                                                 |
| `docs/08_TIME_SERIES_STORAGE.md`      | TimescaleDB hypertable, compression, CA, retention.                    |
| `docs/09_ALERT_ENGINE.md`             | State machine `idle→armed→fired→cooldown→armed`, smart cooldown.       |
| `docs/10_DELIVERY_LAYER.md`           | SSE на дашборд, Telegram (групповой чат), без публичного API.          |
| `docs/11_EXCHANGE_ADAPTERS/*.md`      | Per-exchange адаптеры (12 бирж + `_BASE_CONNECTOR.md`).                |
| `docs/12_OBSERVABILITY_SLO.md`        | Prometheus / Grafana / Loki, метрики, SLO.                             |
| `docs/13_OPERATIONS.md`               | Runbooks, systemd, инциденты.                                          |
| `docs/14_TEST_STRATEGY.md`            | Coverage, contract-тесты, фикстуры.                                    |

### 1.2 Workflow с docs/

- В начале задачи: прочитать `00_DECISIONS_LOG.md` целиком и релевантные разделы профильных файлов.
- При архитектурном изменении: **сначала PR в `docs/`**, потом код.
- При расхождении кода и docs: уточнить у пользователя, привести код к docs (по умолчанию) или зафиксировать новое решение в `00_DECISIONS_LOG.md` как `Fxx`.
- Любой отход от уже принятого решения C1–C17 / Q1–Q8 — добавить запись `Fxx` с обоснованием.

---

## 1.5 Архитектура с высоты птичьего полёта

Полная схема — `docs/02_ARCHITECTURE.md`. Здесь — карта runtime-процессов для быстрой ориентации.

### 1.5.1 Топология (bare-metal, без Docker — Q6)

Шесть независимых systemd-юнитов в `deploy/systemd/`, каждый — отдельный Python-процесс из `backend/app/`:

| Unit                                | Entrypoint                 | Роль                                                                  |
| ----------------------------------- | -------------------------- | --------------------------------------------------------------------- |
| `oi-tracker-scheduler.service`      | `app.scheduler_main`       | 60s loop: пуллит 12 бирж параллельно, нормализует, bulk-insert в TS.  |
| `oi-tracker-alert-engine.service`   | `app.alert_engine_main`    | State machine `idle→armed→fired→cooldown`, hot lane для Hyperliquid.  |
| `oi-tracker-tg-sender.service`      | `app.tg_sender_main`       | aiogram 3.x sender-only, читает delivery_queue, дросселирует, шлёт в TG. |
| `oi-tracker-api.service`            | uvicorn → `app.api.main`   | FastAPI: REST + SSE для дашборда. Без публичного API (Q4).            |
| `oi-tracker-cleanup.service/.timer` | `app.cleanup_main`         | Retention 90 дней, без downsampling.                                  |

Связь между процессами — **только через PostgreSQL** (event tables + delivery_queue), без брокеров.

### 1.5.2 Поток данных (одна итерация)

```
Exchange REST/WS  →  exchanges/<exchange>.py    (per-connector)
                  →  normalizer/pipeline.py     (parsers/ → канонические OISample события)
                  →  storage/repositories/*     (asyncpg bulk insert в `oi_samples` hypertable)
                  →  alert_engine/state_machine (читает свежие samples, считает Δ%, переходит по FSM)
                  →  delivery_queue (DB-table)  →  tg_sender (TG group) + api/sse (dashboard)
```

`asyncpg` — только в hot path (bulk insert), всё остальное — SQLAlchemy Core, **без ORM session** (см. `docs/02_ARCHITECTURE.md §2.1.1` ADR).

### 1.5.3 Слои в `backend/app/`

| Слой               | Где                                    | Что внутри                                                            |
| ------------------ | -------------------------------------- | --------------------------------------------------------------------- |
| Exchange adapters  | `exchanges/`                           | 12 коннекторов (`binance`, `bybit`, …) + `_base.py` интерфейс, `_circuit_breaker`, `_rate_limiter`, `_ring_buffer` (Hyperliquid hot lane). |
| Normalizer         | `normalizer/pipeline.py` + `parsers/`  | Per-exchange парсеры → канонические `OISample` события.               |
| Storage            | `storage/` + `db/`                     | Repository pattern. asyncpg в hot path, SQLAlchemy Core — остальное.  |
| Alert engine       | `alert_engine/`                        | `state_machine.py`, `evaluation_cycle.py`, `hot_lane.py`, `signals/`. |
| Delivery           | `delivery/`, `tg_sender/`, `api/sse/`  | dispatcher, templates, throttle, SSE-стримы.                          |
| API                | `api/`                                 | `main.py`, `routes/` (alerts, alert_rules, chats, instruments, live, settings, symbols, tg_test), `schemas/`, `middleware.py`. |
| Instruments        | `instruments/`                         | Реестр инструментов, 5-минутный sync (`docs/06_INSTRUMENT_REGISTRY.md`). |
| Scheduler          | `scheduler/loop.py`, `supervisor.py`   | sd_notify watchdog (180s), per-exchange изоляция.                     |
| Observability      | `observability/`                       | Prometheus metrics, structlog JSON.                                   |
| Tools              | `tools/`                               | CLI: replay фикстур, capture_fixtures для contract-тестов.            |

### 1.5.4 Frontend (`frontend/`)

React 18 + Vite + TS strict. Раздаётся nginx из `frontend-dist/` (см. `package.json` script `deploy`). API-типы генерируются из OpenAPI (`pnpm gen:api`).

`src/`: `pages/` (dashboard + symbol), `components/`, `state/`, `api/`, `hooks/`, `format/`, `layout/`. SSE через `api/`, без 5s polling (см. §13).

---

## 1.6 Команды

`uv` — основной runner для backend (см. `docs/17_TOOLING.md §1.1`). Frontend — `pnpm`.

### Backend (из `backend/`)

```bash
# Setup
uv sync --extra dev

# Tests
uv run pytest -m "not slow"                # default
uv run pytest -m unit                      # быстрые unit
uv run pytest -m integration               # требует Docker для testcontainers
uv run pytest -m contract                  # replay фикстур из tests/fixtures/
uv run pytest --cov                        # с coverage (gate: fail_under=69)
uv run pytest tests/unit/test_state_machine.py::test_armed_to_fired -xvs   # одиночный тест

# Lint / types (всё должно быть чисто перед коммитом)
uv run ruff check .
uv run ruff format --check .
uv run mypy app                            # --strict, см. pyproject.toml [tool.mypy]
uv run pre-commit run --all-files          # всё за раз

# Миграции (требует поднятый PG+TS)
uv run alembic upgrade head
uv run alembic downgrade -1
uv run alembic revision -m "<msg>"         # autogenerate отключён — пишем DDL руками
```

### Frontend (из `frontend/`)

```bash
pnpm install
pnpm dev                                   # vite dev server
pnpm build                                 # tsc --noEmit + vite build
pnpm typecheck
pnpm lint                                  # --max-warnings 0
pnpm test                                  # vitest run (одноразово)
pnpm test:ui                               # vitest watch UI
pnpm gen:api                               # пересобрать src/api/types.gen.ts из http://127.0.0.1:8010/openapi.json
pnpm deploy                                # build + rsync в /var/www/oi-tracker/frontend-dist/ (prod)
```

### Production (на сервере)

```bash
# Перезапуск отдельного процесса
sudo systemctl restart oi-tracker-scheduler
sudo systemctl status oi-tracker-alert-engine
sudo journalctl -u oi-tracker-tg-sender -f --since "10 min ago"

# Все юниты — в deploy/systemd/, runbooks — в docs/13_OPERATIONS.md.
```

---

## 2. Анализ задачи и планирование

### 2.1 Анализ перед планом (обязательно)

Прежде чем писать план, явно для себя ответить:

- **Цель:** какую пользовательскую/системную задачу закрываем.
- **Ограничения:** ссылка на конкретные пункты `docs/` и `00_DECISIONS_LOG.md`.
- **Вход / выход:** какие данные приходят, что уходит и куда.
- **Edge cases:** пустые/null OI, биржа недоступна, native vs snapshot, gap во времени, валюта settle ≠ USDT (отсекается), reorg/replay.
- **Где может сломаться:** failure modes, race conditions, partial failures.
- **Что НЕ покрыто:** явный список того, что не делаем в этой итерации.

### 2.2 Plan-mode triggers (когда план обязателен)

План в `tasks/todo.md` обязателен в любом из случаев:

- Изменение DDL, `05_DATA_CONTRACTS.md`, `00_DECISIONS_LOG.md`.
- Касание state machine alert engine (`09_ALERT_ENGINE.md`).
- Касание формул времени и freshness (`04_TIME_MODEL.md`).
- Новый или изменяемый exchange-адаптер (`11_EXCHANGE_ADAPTERS/*`).
- Изменение схемы хранения / TimescaleDB CA / compression policies.
- Любая работа, затрагивающая >2 файла или >50 строк диффа.
- Любая задача из 3+ шагов или с архитектурным выбором.

Если задача казалась простой, но в процессе всплыло ≥1 из триггеров — **остановиться,
вернуться в plan mode**.

### 2.3 Управление неопределённостью

- Требования неясны — зафиксировать **assumptions** в `tasks/todo.md` перед реализацией.
- Допущение критично (влияет на схему/контракты/state machine) — **остановиться и уточнить** у пользователя.
- Не выдумывать поведение, влияющее на архитектуру.
- На вопрос «как лучше?» — давать конкретную senior-инженерную рекомендацию + 2–3 предложения обоснования; альтернативы упоминать только если они сопоставимы.

### 2.4 `tasks/todo.md`

- Перед началом реализации — пишем план чекбоксами.
- Сверяемся с пользователем до старта (если задача выше plan-mode trigger).
- Отмечаем пункты на ходу.
- На каждом шаге — короткое резюме изменения.
- В конце задачи — секция «Review» с итогами и ссылкой на `lessons.md` (если применимо).

### 2.5 Feedback loop после плана

После завершения задачи явно ответить:

- Что было неверно в плане?
- Где оценка трудоёмкости/риска была плохой?
- Что упустили в edge cases?

Если ответ нетривиален — добавить запись в `tasks/lessons.md`.

---

## 3. Реализация

### 3.1 Базовые принципы

- **Минимальное воздействие**: трогаем только нужное.
- **Без лени**: ищем корневую причину; временные фиксы — только как явный workaround с TODO и issue.
- **Senior-стандарт**: код должен пройти ревью с акцентом на безопасность, SLO, observability, edge cases.
- **Простота**: каждое изменение максимально простое для решения текущей задачи.
- **Элегантность (в меру)**: на нетривиальном изменении — пауза «есть ли путь элегантнее?»; на простых очевидных фиксах не оверинженирить.
- **Без сайд-эффектов**: ничего не «заодно».

Иерархия §0 решает спор простоты и элегантности: корректность и поддерживаемость
важнее краткости.

### 3.2 Code style и type safety

- Python 3.12. Type hints на 100% backend-кода. `mypy --strict`.
- `ruff format` + `ruff check`: ноль warning'ов перед коммитом.
- Pydantic v2 для всех API I/O и event contracts.
- `structlog` (JSON) для логов; никаких `print` и `logging.info(f"...")` без структурированных полей.
- `# noqa` / `# type: ignore` — только с комментарием-обоснованием на той же строке.
- Frontend: TS strict, ESLint + Prettier, React функциональные компоненты, Vite.

### 3.3 Тесты

- TDD обязателен для: normalizer pipeline, alert state machine, time/freshness утилит.
- Coverage targets — `docs/14_TEST_STRATEGY.md`.
- Contract-тесты на реальных payload-фикстурах — обязательны для каждого нового коннектора (`11_EXCHANGE_ADAPTERS/*`).
- Тесты детерминированы: никаких `datetime.now()` без `freezegun` / контролируемого clock.
- Никогда не «фиксить» тест, если ломается реализация (если только тест действительно неверный).

### 3.4 База данных

- Все schema-изменения — через **Alembic**, не ручные `ALTER`.
- Каждая миграция reversible (`downgrade`); one-way помечается с причиной в docstring.
- Никаких прямых `SELECT/INSERT` в бизнес-логике; **repository pattern**.
- Никогда f-string в SQL — только параметризация через SQLAlchemy.
- TimescaleDB-специфика (compression policies, CA refresh, retention) — изолирована в migration / service layer, не разбросана.
- Numeric: `NUMERIC(38, 18)` для coins/price, `NUMERIC(38, 8)` для USDT notional (`00_DECISIONS_LOG.md` C9).

### 3.5 Безопасность

- Никаких секретов в коде или коммитах. `.env` с правами `0600`.
- Telegram bot token — env-only; `chat_id` редактируется из UI (`Q8`).
- User input на API boundary — Pydantic-валидация без `try/except` обхода.
- Outbound HTTPS — TLS verified; `verify=False` запрещено.
- Логи не содержат токены, ключи, авторизационные payload'ы.
- Любая работа с user-controlled данными — review через линзу OWASP Top 10 (injection, SSRF, IDOR, secret leak).

### 3.6 Error handling

- Никогда не подавлять exception без логирования.
- Все ожидаемые ошибки — типизированные доменные классы.
- На границе слоя — конвертация internal exceptions в API/protocol errors.
- При оборачивании — всегда `raise NewError(...) from original` (сохраняем причину).
- Изоляция per-exchange: одна биржа не валит остальные (`00_DECISIONS_LOG.md`, ветка «Сбор данных»).

### 3.7 Performance budgets

См. `docs/01_PRODUCT_SPEC.md` NFR-1 + `docs/12_OBSERVABILITY_SLO.md`.

| Метрика                                | Бюджет                            |
| -------------------------------------- | --------------------------------- |
| E2E latency (тик биржи → TG)           | P95 ≤ 90s, P99 ≤ 180s             |
| Bulk insert `oi_samples` за цикл       | < 500ms                           |
| API endpoint                           | < 100ms p95                       |
| Alert engine cycle                     | < 5s                              |
| Poll cycle per exchange                | укладывается в 60s окно           |

Если изменение угрожает бюджету — обосновать в плане и измерить (бенчмарк/нагрузка) до merge.

---

## 4. Верификация и Definition of Done

Задача не «done», пока не доказано, что работает.

### 4.1 Базовый чек-лист (production)

- [ ] Код работает на golden path и на edge cases из §2.1.
- [ ] Тесты добавлены: unit + (integration или contract там, где применимо).
- [ ] Coverage не упал (см. `14_TEST_STRATEGY.md`).
- [ ] `mypy --strict` чисто.
- [ ] `ruff check` + `ruff format` чисто.
- [ ] Если schema change — Alembic миграция reversible.
- [ ] Если новый компонент — Prometheus метрики добавлены.
- [ ] Если новая failure mode — runbook в `docs/13_OPERATIONS.md` обновлён.
- [ ] Если архитектурное решение — запись `Fxx` в `00_DECISIONS_LOG.md`.
- [ ] Затронутые `docs/*.md` обновлены (см. §6).
- [ ] Поведение объяснимо в одном предложении.
- [ ] Не введены новые dependencies без обоснования.

### 4.2 Метод верификации

- Запустить тесты, посмотреть логи, продемонстрировать корректность.
- Сравнить поведение `main` и текущей ветки на репрезентативном входе, когда релевантно.
- Перед показом результата пользователю — самопроверка через линзы: безопасность, SLO, observability, edge cases.

---

## 5. Git и коммиты

- **Conventional commits**: `feat / fix / refactor / docs / test / chore / perf / ci`.
- Атомарные коммиты: один коммит = одно логическое изменение.
- **Никогда** `git push --force` на `main` или shared branch.
- **Никогда** `--no-verify`. Если pre-commit упал — починить, не обходить.
- PR description: что изменилось, почему, как тестировалось, какие docs затронуты.
- Никаких коммитов с секретами / `.env` / большими бинарями. Если попало случайно — ротация секрета + history rewrite только по явному указанию пользователя.

---

## 6. Документация при изменениях

| Что меняешь                    | Что обязан обновить                    |
| ------------------------------ | --------------------------------------- |
| Public API / SSE / TG payload  | `docs/10_DELIVERY_LAYER.md` + OpenAPI   |
| DB schema / DDL                | `docs/05_DATA_CONTRACTS.md` + `08_TIME_SERIES_STORAGE.md` |
| Alert state machine            | `docs/09_ALERT_ENGINE.md`               |
| Time/freshness формулы         | `docs/04_TIME_MODEL.md`                 |
| Exchange-адаптер               | `docs/11_EXCHANGE_ADAPTERS/<exchange>.md` |
| Любой отход от docs            | новая запись `Fxx` в `00_DECISIONS_LOG.md` |
| Новый failure mode / инцидент  | `docs/13_OPERATIONS.md` (runbook)       |

---

## 7. Субагенты — когда да, когда нет

### 7.1 Когда использовать

- Параллельный ресёрч / разведка по нескольким независимым вопросам.
- Анализ большого объёма выдачи, который не нужен в основном контексте.
- Отдельные изолированные задачи: написание тестов, генерация per-exchange адаптера, security-review.
- Когда длинная цепочка туллинга может «загрязнить» контекст.

Доступные специализированные агенты — см. `~/.claude/agents/` + project-specific
из `.claude/agents/` (см. §11).

### 7.2 Когда НЕ использовать

- Чтение `CLAUDE.md` и `docs/*` — нужно в основном контексте.
- Задачи < 5 минут — overhead субагента не оправдан.
- Задачи с обязательной continuity (debug state machine, поэтапная отладка alert flow).
- Финальное ревью своей работы перед показом — нужно in-context для следующего шага.

---

## 8. Управление scope (boy scout, но без энтузиазма)

- Не расширяй scope без причины.
- Если расширение нужно — явно зафиксируй в `tasks/todo.md`, **почему**, и согласуй с пользователем.
- Нашёл багу/мусор/нелогичность вне scope — **НЕ чини в текущем PR**. Открой issue (или запиши TODO в `tasks/todo.md` секцию `Out-of-scope findings`), продолжай свою задачу.
- Это сохраняет атомарность ревью и снижает риск «5 PR из одного».

---

## 9. Самоулучшение — `tasks/lessons.md`

### 9.1 Когда писать

После **любой** правки от пользователя, выявившей системную ошибку моего поведения
(не разовую опечатку), — добавить урок.

### 9.2 Формат записи

```
### YYYY-MM-DD · <category> · <pattern-name>
- **Контекст:** что произошло.
- **Урок:** обобщённое правило.
- **Триггер:** как распознать, что правило применимо.
- **Действие:** что делать.
```

Категории: `bug-fix-pattern`, `design-mistake`, `communication-mistake`, `scope-creep`,
`premature-optimization`, `spec-violation`, `test-gap`.

### 9.3 Дисциплина

- Не дублировать. Перед записью — найти существующий близкий урок и обновить его.
- Обобщать до паттерна; не записывать «однажды Х».
- Удалять/объединять устаревшие правила.
- Перечитывать в начале сессии в этой области.
- Максимум сигнала, минимум шума.

---

## 10. Автономная работа

- Получил баг-репорт → просто чини. Не проси «вести за руку».
- Логи / упавшие тесты / стек — указываются и закрываются мной, не пользователем.
- Ноль переключений контекста со стороны пользователя для рутины.
- Если упавший CI-тест — иди чини без подсказок «как».

**Ограничение:** автономия отменяется, если допущение критично (см. §2.3) или задача
попадает под plan-mode trigger (§2.2).

---

## 11. Project-specific агенты и команды

### 11.1 Агенты (`.claude/agents/`)

| Имя                          | Когда вызывать                                           |
| ---------------------------- | -------------------------------------------------------- |
| `decisions-guardian`         | Перед merge / при крупных изменениях — проверка соответствия `00_DECISIONS_LOG.md`. |
| `exchange-adapter-builder`   | При написании/обновлении коннектора биржи — генерация по `11_EXCHANGE_ADAPTERS/<exchange>.md`. |
| `timescale-migrator`         | При schema-change — генерация Alembic-миграции с учётом hypertable / compression / CA. |

### 11.2 Slash-команды (`.claude/commands/`)

| Команда       | Что делает                                                                  |
| ------------- | --------------------------------------------------------------------------- |
| `/spec`       | Найти и зачитать релевантный документ из `docs/` в основной контекст.       |
| `/decision`   | Показать конкретное решение C1–C17 / Q1–Q8 / Fxx из `00_DECISIONS_LOG.md`.  |
| `/adapter`    | Открыть спеку биржи + предложить вызвать `exchange-adapter-builder`.        |
| `/verify-spec`| Прогнать новый код против `docs/05_DATA_CONTRACTS.md` (валидация типов).    |

---

## 12. Стек и инструменты (краткая шпаргалка)

- Python 3.12 + FastAPI + asyncio + httpx
- PostgreSQL 16 + TimescaleDB (compression, CA, retention 90 дней)
- React + Vite + TypeScript (strict)
- aiogram 3.x — sender-only, групповой чат
- Prometheus + Grafana + Loki
- Bare-metal + systemd-юниты, **без Docker** (Q6)
- Alembic для миграций
- pytest + pytest-asyncio + freezegun + hypothesis (где уместно)
- ruff + mypy --strict
- Single-user, без авторизации, без публичного API (Q4)
- Бэкапы — future work (Q7), сейчас не настраиваются

---

## 13. Что не делаем (явные «нет»)

- Нет Docker.
- Нет retro-fire алертов при backfill/replay (`00_DECISIONS_LOG.md`).
- Нет абсолютных порогов Δ — только относительные в %.
- Нет авторизации / multi-user.
- Нет публичного API.
- Нет downsampling — retention 90 дней для всего.
- Нет 5s polling на дашборде — только SSE с reconnect.
