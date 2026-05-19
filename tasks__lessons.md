# tasks/lessons.md

> Уроки, извлечённые из правок пользователя. См. CLAUDE.md §9.
> Формат записи строго регламентирован — иначе не добавлять.

---

## Формат записи

```
### YYYY-MM-DD · <category> · <pattern-name>
- **Контекст:** что произошло.
- **Урок:** обобщённое правило.
- **Триггер:** как распознать, что правило применимо.
- **Действие:** что делать.
```

**Категории:** `bug-fix-pattern`, `design-mistake`, `communication-mistake`,
`scope-creep`, `premature-optimization`, `spec-violation`, `test-gap`.

## Дисциплина (см. CLAUDE.md §9.3)

- Не дублировать. Перед записью — найти существующий близкий урок и обновить его.
- Обобщать до паттерна; не записывать «однажды Х».
- Удалять или объединять устаревшие правила.
- Перечитывать в начале сессии в этой области.
- Максимум сигнала, минимум шума.

---

## Уроки

### 2026-04-30 · communication-mistake · plan-first-discipline
- **Контекст:** в первой сессии планирования начал создавать MD-файлы (`mkdir`, `Write docs/*`) сразу после согласования решений, без явной команды «реализуй».
- **Урок:** «спланируй / проанализируй / дай мнение» — это НЕ разрешение на создание файлов. Многократные раунды Q&A — тоже не разрешение.
- **Триггер:** пользователь обсуждает архитектуру / решения / задаёт уточняющие вопросы; нет глагола «реализуй / пиши / создавай / делай».
- **Действие:** оставаться в режиме обсуждения. Если все вопросы согласованы — представить consolidated план и явно запросить go-ahead. Не брать tool с `Write` в чёрный ящик.

### 2026-04-30 · communication-mistake · senior-opinion-direct-answer
- **Контекст:** на вопрос «как лучше?» давал перечисление вариантов с оговорками «зависит от...».
- **Урок:** пользователь — CTO-уровня заказчик, делегирующий технические решения. «Как лучше?» — не риторический вопрос. Ждёт ОДНУ рекомендацию + 2–3 предложения обоснования.
- **Триггер:** пользователь формулирует «как лучше / что бы предложил senior?».
- **Действие:** одна рекомендация, обоснование, явная пометка DIFFERENCE/SAME относительно ТЗ. Альтернативы — только если они сопоставимы и пользователь должен выбирать сознательно.

### 2026-05-01 · spec-violation · connector-spec-vs-reality
- **Контекст:** при работе над M2.B.5 Bitunix потратил 30 минут на capture'ы фикстур + connector skeleton, прежде чем обнаружил что Bitunix public API вообще не отдаёт OI (spec утверждал обратное). Аналогичные мелкие расхождения spec vs реальность были на Gate.io (`status` vs `trade_status`, `position_size` vs `total_size`), MEXC (`futureType` vs `contractType`), Bitget (V1/V2 schema differences). Применили урок 2026-05-04 при замене Bitunix → Bitmart (`F42`): сначала `curl /contract/public/details` в plan-фазе → подтвердили `open_interest` + `open_interest_value` в payload → только потом написали spec `bitmart.md`. Spec-vs-реальность delta = 0.
- **Урок:** для каждого нового exchange-коннектора **первый шаг** — это live capture всех endpoints (instruments + OI + price), валидация полей в реальных response'ах. Только потом читать spec для cross-check'а и видеть delta. Если delta crucial (e.g. OI вообще нет, как Bitunix) — лучше defer'нуть connector до момента когда API подтверждается, чем ловить расхождение в production.
- **Триггер:** написание нового exchange-коннектора по `docs/11_EXCHANGE_ADAPTERS/<exchange>.md`.
- **Действие:** перед `Write` в `app/exchanges/<name>.py` — обязательный live capture в `tests/contract/<name>/fixtures/` через httpx с инспекцией ВСЕХ ключей response'а. Если поля spec'а отсутствуют — НЕ городить guess-based парсер, а пинговать пользователя/документацию. Sub-finding F-DOC-N в tasks/todo.md если spec нужно обновить.

### 2026-05-01 · spec-violation · systemd-execstartpost-race-and-permission
- **Контекст:** при выполнении Phase 0.E install следовал docs/13 §3.2 шаблону `ExecStartPost=/bin/bash -c 'chgrp www-data /run/oi-tracker/api.sock && chmod 0660 ...'` — упало (1) chgrp до создания socket'а uvicorn'ом, (2) chgrp под user oi-tracker не member www-data, (3) parent dir 0750 oi-tracker:oi-tracker блокирует traverse www-data → 502.
- **Урок:** `ExecStartPost` для `Type=simple` сервисов с UDS требует трёх вещей одновременно: (a) `+` prefix для root-привилегий, (b) polling loop ожидающий socket файл, (c) chgrp на ПАРЕНТ-директорию (для traverse прав), не только на socket. Без всех трёх — production-deploy ломается.
- **Триггер:** написание systemd unit с UDS + чужим nginx-user (www-data) + ExecStartPost для socket-перм.
- **Действие:** copypaste pattern из `deploy/systemd/oi-tracker-api.service` ExecStartPost. Не следовать literal docs §3.2 — там пропуски (см. F-DOC-5).

### 2026-05-01 · spec-violation · infra-commands-validate-on-target-os
- **Контекст:** при выполнении Phase 0.A host infra setup нашёл 4 docs gaps (F-DOC-1..4): apt-key deprecated на noble, chmod без chown, http2 directive требует nginx ≥1.25.1, REVOKE на role не закрывает CONNECT через PUBLIC inheritance.
- **Урок:** инфра-команды в docs — не теоретические эскизы, они должны валидироваться на целевой OS / version stack. Особенно: deprecated CLI flags, ownership vs permission, минимальные версии ПО, security primitives с counterintuitive semantics (PUBLIC inheritance в Postgres).
- **Триггер:** написание docs с командами `apt`, `chmod`/`chown`/`useradd`, `nginx ...`, GRANT/REVOKE/ALTER ROLE, или любых системных операций на shared host.
- **Действие:** перед commit'ом infra-секций в docs — (a) проверь deprecated flags для current LTS (noble/jammy), (b) убедись что chmod дополнен chown где dir owner ≠ root, (c) укажи минимальную версию ПО если syntax-зависимый (nginx http2, openssl, postgres specific functions), (d) для security-related GRANT/REVOKE — вручную проверь achieved boundary, не доверяй literal command (PUBLIC inheritance, role membership, default privileges).

### 2026-05-01 · test-gap · smart-cooldown-edge-in-no-retro-fire-tests
- **Контекст:** в первой версии `tests/integration/alert_engine/test_no_retro_fire.py` тест «replay внутри cooldown с Δ=10%, last_fired=6%» ожидал SUPPRESS, но state_machine корректно делал smart re-fire (10 ≥ 6×1.5=9). Тест был неверный, не код.
- **Урок:** при написании cooldown-сценариев на state_machine — всегда явно проверять, не активирует ли тестовая Δ smart re-fire factor. Конкретно: smart fire bar = `last_fired_value × smart_cooldown_factor`. Δ выше этого бара = валидный re-fire, не баг. Если хотим тестировать SUPPRESS в active cooldown — выбирать Δ < smart bar или сетапить last_fired_value так, чтобы factor не давал re-fire.
- **Триггер:** написание тестов для F7 (smart cooldown), F9 (no retro-fire), 09 §3.4 (cooldown semantics) или любых multi-cycle state_machine-сценариев.
- **Действие:** перед asserting `result.fire is False` в COOLDOWN-state — посчитать `last_fired_value × smart_cooldown_factor` и убедиться, что Δ_test < этого. В docstring теста зафиксировать формулу. Так же поступить с Δ=0 / negative Δ для оператора `<=` (down rules).

### 2026-05-01 · spec-violation · cross-doc-tooling-consistency
- **Контекст:** при правке `13_OPERATIONS §7.1` обнаружил `npm install / npm run build`, хотя §2.1.6 того же файла использует `pnpm install --frozen-lockfile / pnpm build`. Поздним обнаружил при review фиксированной правки — пришлось вернуться и согласовать.
- **Урок:** при любой правке инфра-секций (deploy, build, install, paths, ports, command names) — ДО написания диффа сделать `grep` по всем файлам docs/ на упоминания смежных инструментов. Несогласованность копится в нескольких местах одновременно.
- **Триггер:** редактирование команд CLI / путей / имён сервисов / портов в любом из `02_ARCHITECTURE`, `13_OPERATIONS`, `15_DEPLOYMENT_INFRA`, `16_ROADMAP`, `17_TOOLING`.
- **Действие:** перед правкой — `grep -nE '(npm|pnpm|yarn|localhost:[0-9]+|/var/www/oi-tracker[^/]|frontend-dist)' docs/`. Все совпадения свести к единому стилю в одном PR; добавить tracking pattern в `decisions-guardian` запрос.

### 2026-05-02 · design-mistake · sparse-vs-dense-payloads
- **Контекст:** при M4.C.3 (Alerts log) обнаружил, что REST `AlertEvent` имеет 18 полей, а SSE `alert` event несёт только 7 (`rule_id`, `exchange`, `canonical_symbol`, `window`, `decision`, `delta_pct`, `ts_processed`). При попытке отрисовать одну таблицу из обоих источников — TS-несовместимость. Решено через synthetic `AlertRow` с `_origin: "rest" | "sse"` и null-заполнением недостающих полей; drawer показывает warning «partial payload only» когда `_origin === "sse"`.
- **Урок:** SSE и REST для «одного» доменного события почти никогда не одинаковы по форме. SSE листенер обычно несёт минимум для notification, REST — полный enriched payload. Перед написанием UI, который смешивает оба источника, явно сравнить shapes и спроектировать unified row type с `_origin` discriminator + warning для sparse-rows.
- **Триггер:** UI компонент агрегирует REST snapshot + SSE deltas одного домена (alerts, samples, deliveries).
- **Действие:** перед написанием UI — diff REST schema vs SSE event schema (поле в поле). Создать unified row type с обязательным `_origin` flag и nullable missing-fields. На UI явно показывать warning/banner когда происхождение sparse. Не пытаться скрыть разницу — пользователь должен понимать, почему JSON в drawer неполный.

### 2026-05-02 · communication-mistake · vite-ipv6-binding-tripwire
- **Контекст:** при M4.C smoke-тестах несколько раз ловил `curl: (7) Failed to connect to 127.0.0.1 port 5173`. Vite dev binding'ится на `[::1]:5173` (IPv6), а не на IPv4 loopback. Каждый раз тратил ~5min на диагностику пока не вспоминал. Произошло 3+ раза за сессию.
- **Урок:** dev-серверы (vite, esbuild, parcel) на современных Node.js биндятся на IPv6 loopback по умолчанию. `curl http://127.0.0.1:PORT` фейлится с exit 7 — нужно `http://[::1]:PORT` или `http://localhost:PORT` (последнее зависит от `/etc/hosts`).
- **Триггер:** smoke-тестируем dev-сервер через curl и получаем `Failed to connect`.
- **Действие:** для curl-смоков dev-серверов всегда сначала `ss -lntp | grep PORT`, посмотреть на binding (`127.0.0.1` vs `[::1]`), и адаптировать URL. Альтернатива — указать `--host 0.0.0.0` или `--host 127.0.0.1` в vite/dev-server CLI; но для одиночных smoke-проверок проще IPv6 URL.

### 2026-05-02 · spec-violation · coverage-scope-vs-overall-ambiguous
- **Контекст:** план в `tasks/todo.md` C.X.2 говорил «coverage ≥ 70%». При первом измерении overall coverage (`v8` provider, no scope) показал 18.54% — потому что считались все файлы репо, включая чистые view-обёртки (`Dashboard.tsx`, `layout/`, `OIChart.tsx`) без логики. Senior-решение: scoped на `hooks/** + format/** + FiltersBar.tsx` (logic-bearing modules), threshold 70%, exclude REST-обёртки (`useAlertRules`/`useAlertsLive`/`useSymbolData`) — они exercises через E2E.
- **Урок:** `coverage ≥ N%` без явного scope — двусмысленно. Overall vs scoped даёт ×3 разницу. Чистые JSX-обёртки — пустой эффект для coverage, но захламляют threshold; throwaway-тесты на них — anti-pattern (геймишь метрику без сигнала).
- **Триггер:** план содержит «coverage ≥ N%» без явного `include`/`exclude` scope.
- **Действие:** при настройке coverage gate — явно описать `include` (logic-bearing) и `exclude` (view-only, REST-обёртки exercising через E2E). Документировать выбор в `vite.config.ts` коммент-блоком + в Review/lessons. Не писать throwaway-тесты, чтобы дотянуть метрику.

### 2026-05-02 · scope-creep · vite-tsc-noEmit-required
- **Контекст:** при scaffold M4.A.1 использовал `"build": "tsc -b && vite build"` (как в стандартных vite-react templates). `tsc -b` без `noEmit:true` создаёт `.js` рядом с каждым `.ts` в `src/`, vitest подхватывает оба → 44 теста вместо 22.
- **Урок:** для не-library vite-проектов tsconfig обязан содержать `"noEmit": true`. Build script — `tsc --noEmit && vite build` (типчек без emit). `tsc -b` (project references) уместен только для monorepo либо когда библиотечный проект эмитит declarations.
- **Триггер:** scaffold нового frontend проекта на vite + TS, либо появление `.js` файлов в `src/` после `pnpm build`.
- **Действие:** в tsconfig.json — `"noEmit": true`. В package.json build → `"tsc --noEmit && vite build"`. Перед коммитом — `find src -name "*.js" -o -name "*.d.ts" -o -name "*.tsbuildinfo"` ноль результатов.

### 2026-05-02 · test-gap · coverage-path-c-decisions-not-lines
- **Контекст:** на M5.C закрытии измеренный coverage был 64% при docs/14 §2.1 target ≥80% overall + ≥85% repos + ≥80% API routes. Path A (literal 80%) = 20-30h работы по backfill integration tests на CRUD-обёртки (`oi_samples_repository.insert(samples)` это `INSERT ON CONFLICT DO NOTHING` через asyncpg `$N` binding — тестируется сам Postgres, не наша логика). Path B (zero new tests, F-record) — оставлял quick-wins на столе. Senior-decision Path C: 5h на decision-heavy modules → evaluation_cycle 17%→99%, producer 54%→100%, dispatcher 74%→98%, logging_config 54%→98%; CRUD-репозитории + API routes явно deferred в M6 через F31.
- **Урок:** «coverage ≥ N%» без указания **WHERE** покрытие имеет ценность — недостаточная спецификация. Tests should cover decisions, not lines. Pattern: backfill business logic (state machines, orchestration loops, retry handlers, payload builders) до 90+%, явно defer CRUD-wrappers + API endpoints на отдельный milestone с integration harness. F-record фиксирует решение «X% overall — это допустимый baseline когда decision-heavy code ≥98%».
- **Триггер:** milestone closure-gate на coverage; existing baseline сильно ниже target (>10pp gap); большая часть gap — на CRUD/I-O путях.
- **Действие:** (а) разделить coverage targets по слою decision-heavy / CRUD / I/O; (б) backfill только decision-heavy модули в текущем milestone; (в) F-record явно formalises «defer CRUD coverage to next milestone with integration harness plan»; (г) bump `fail_under` до фактического baseline (anti-regression gate).

### 2026-05-02 · spec-violation · spec-drift-additive-accumulates
- **Контекст:** на M5.C closure verify-spec audit нашёл 7 drift items в `docs/05_DATA_CONTRACTS.md` vs `app/domain/*.py`: `event_id: int → int | None` (BIGSERIAL relaxation), `oi_now_notional_usdt: Decimal → Decimal | None` в AlertEvent (NO_TRANSITION cooldown-ticks), `warnings: list → tuple` (frozen=True hashability), `AlertDecision.NO_TRANSITION` enum value не в spec, `TransitionResult` контракт не объявлен в spec, `AlertRule.extra` JSONB не в spec (хотя F24 это покрывает). Все additive, ни один не wire-breaking. Накопились между M3 и M5 — каждый refine типа в коде («давай Optional чтобы BIGSERIAL не мешал», «tuple вместо list для frozen») не сопровождался PR в `docs/05`.
- **Урок:** spec drift накапливается additively, когда implementation refine типы (BIGSERIAL nullability, hashability constraints, internal-only vocabulary). Каждый «давай немного relax`нём» без PR в spec — drift item, который копится до closure-gate audit. Pattern «code is source of truth / spec follows» проигрывает в multi-milestone проекте; правильно — «spec PR first when refining types».
- **Триггер:** менять Pydantic field type / nullability / collection kind в `app/domain/*.py` без явной ссылки на decision в `00_DECISIONS_LOG.md` или PR в `docs/05`.
- **Действие:** при refine типа в domain class — обязательный one-line PR в `docs/05` в том же commit (или явная F-запись если изменение архитектурное). Перед milestone closure — обязательный verify-spec subagent run; gap'ы либо закрывать spec-update'ом либо F-record acknowledgement. Fxx ADR пишется когда code shipped first (uncommon path), spec-update — когда decisions-first (default path).

### 2026-05-02 · design-mistake · drill-playbook-pre-flight-math-required
- **Контекст:** при написании drill 5.4 «Disk full» (M5.B PR-B-prep) изначально планировал рекомендовать «`dd if=/dev/zero of=balloon.bin count=N` где N подбирается empirically». Понял что это рецепт для catastrophic failure: target 82% disk usage без pre-flight math легко превращается в 95%+ если `df` baseline misread, а PostgreSQL write-protection kicks in вокруг 95%. Senior-rewrite: **обязательная pre-flight math секция** с формулами `total_kb`, `pct_now`, `target_used_kb = total_kb * 0.82`, `balloon_mb = (target_used_kb - used_kb) / 1024`, плюс **hard caps**: drill aborts если `pct_now ≥ 0.78` (no headroom), `balloon_mb < 500` (drill won't move needle), `target / total > 0.85` (recompute), `total < 50 GB` (host too small).
- **Урок:** drill playbook'и для resource-pressure тестов (disk, memory, network bandwidth) **обязаны содержать explicit pre-flight math + hard caps**. «Adjust empirically» — не приемлемо когда side-effect = прод-инцидент. Pattern: каждый math-driven induce шаг = formula + computed value записанный в drill log ДО первой destructive команды + hard caps как abort conditions.
- **Триггер:** написание drill / runbook procedure, которая induces resource pressure (disk fill, memory pressure, CPU saturation, connection exhaustion).
- **Действие:** в induce procedure — секция «Pre-flight math (REQUIRED)» с формулами + computed values + hard cap thresholds (с явными abort conditions). Operator пишет numbers в drill log до `dd` / equivalent. Никаких «pick N empirically» / «start small and increase».

### 2026-05-02 · design-mistake · asyncpg-pool-cross-loop-binding
- **Контекст:** при первой версии M6 integration harness попытался сделать session-scope asyncpg pool (через `pytest_asyncio.fixture(scope="session", loop_scope="session")`), чтобы амортизировать pool создание. Все тесты падали с `InterfaceError("another operation is in progress")` потому что pytest-asyncio default test loop scope = `function`: pool жил в session loop, тесты в function loops, asyncpg отвергает cross-loop usage. Function-scope pool (per-test create+close) против уже поднятого session-scope контейнера — ~30ms на pool-creation, в бюджете.
- **Урок:** asyncpg pool привязан к event loop, в котором был создан. Если pytest-asyncio loop scope ≠ pool scope — `InterfaceError` гарантирован. Сессионный контейнер (Docker) — fine, потому что не async; сессионный asyncpg pool — нет. Pattern для integration tests: контейнер session, pool function (или session pool ТОЛЬКО если все тесты явно `loop_scope="session"`).
- **Триггер:** интеграционные тесты на pytest-asyncio + asyncpg, default loop scope; желание амортизировать pool создание session-scope.
- **Действие:** оставить контейнер session-scope, asyncpg pool — function-scope с per-test create+close. Если возникнет нужда session-pool (>5min suite) — добавить project-wide `asyncio_default_test_loop_scope = "session"` в pyproject + рассмотреть последствия для unit-тестов (fixture sharing, mock state).

### 2026-05-02 · design-mistake · savepoint-rollback-incompatible-with-pool-api
- **Контекст:** план M6.A предусматривал savepoint-based per-test isolation (`BEGIN; ... ROLLBACK;` через одно соединение). Не сработало — repo API подписан на `asyncpg.Pool` и каждая repo function берёт fresh connection из pool через `async with pool.acquire() as conn`. Savepoint, открытый на одном соединении, не покрывает repo writes на других. Switched to TRUNCATE+RESTART IDENTITY+CASCADE per-test — тривиально, ~5ms на reset для типового seed.
- **Урок:** savepoint/transactional-rollback isolation работает только когда **все участники** разделяют одно соединение. Pool-based repo APIs (asyncpg `Pool.execute / Pool.fetchrow / Pool.acquire`) выдают разные соединения — savepoint бесполезен. Альтернативы для pool-based кода: (а) TRUNCATE+reseed (просто, дёшево), (б) per-test container (медленно), (в) рефакторить repo API на `Connection` (большое API change). Для существующего pool-based кода → TRUNCATE.
- **Триггер:** план integration harness содержит «savepoint per test» при существующем `pool: asyncpg.Pool` интерфейсе.
- **Действие:** не браться за savepoint pattern если repo функции принимают pool, а не connection. Сразу выбрать TRUNCATE+reseed; формализовать в F-record что hypertables на TimescaleDB ≥ 2.x поддерживают TRUNCATE напрямую, CA matviews пропускаются (re-materialise from source).

### 2026-05-02 · scope-creep · path-y-partial-milestone-discipline
- **Контекст:** M6 path Y (частичный — только critical I/O repos) был senior-recommendation на «как лучше?» вместо path Z (full backfill) или path X (skip). Доставил harness + 3 repos + F34/F35/F36 + retrospective за ~1 рабочий блок (~5h календарно), coverage uplift 65.88%→69.36%, без overengineering. Path Z был оценен в ~26h dev — литаральный 80% target требовал tests на CRUD-обёртки которые тестируют Postgres, не нашу логику.
- **Урок:** «частичный milestone» — валидный pattern когда (а) есть явная decision-heavy / CRUD граница, (б) decision-heavy уже covered в предыдущих милестоунах, (в) read-only paths covered E2E другими способами (UI, drills). Документировать в roadmap «что НЕ покрыто и почему» — это не technical debt, это intentional scope. Bump `fail_under` локирует фактический baseline без обязательства подниматься до literal target.
- **Триггер:** milestone с большим scope (>20h), большая часть которого — CRUD/repetitive coverage без decision logic; senior-question «как лучше?» с явными path X/Y/Z альтернативами.
- **Действие:** предлагать path Y как default для test-debt milestones; формализовать «что НЕ покрыто и почему» в roadmap; bumpить fail_under до actual; НЕ оставлять «coverage ≥ N% literal» как roadmap target если N не имеет business justification (signal vs noise).

### 2026-05-02 · design-mistake · slowapi-future-annotations-body-inference-break
- **Контекст:** при написании `app/api/routes/chats.py` (M7.C) добавил `from __future__ import annotations` для cleaner imports. Все POST/PATCH endpoints начали возвращать 422 с `{"loc": ["query", "body"], "msg": "Field required"}` — FastAPI трактовал body-Pydantic-параметр как query-param. Pre-existing `app/api/routes/settings.py` с PUT/SettingPut работал — у него **нет** `from __future__ import annotations`. Слой проблемы: `@limiter.limit("5/minute")` декоратор от slowapi оборачивает функцию, и FastAPI's body-inference резолвит type annotations через `get_type_hints`. С future annotations типы становятся string forwards и forward-resolve через wrapper-decorated function ломается. `Annotated[Body, Body()]` явно — не помогло. Простой фикс — drop `from __future__ import annotations` в этом файле.
- **Урок:** **`from __future__ import annotations` несовместим с FastAPI body-inference на slowapi-декорированных handler'ах.** Body-параметр трактуется как query-param из-за поломки в forward-reference resolution через wrapper. Pattern: routes с `@limiter.limit` — **без** future annotations; runtime imports (asyncpg etc.) — компенсируются `noqa: TC002`/`TC001` markers вместо TYPE_CHECKING block.
- **Триггер:** новый FastAPI route file с `@limiter.limit` декоратором; импорт `from __future__ import annotations` в начале файла.
- **Действие:** в `app/api/routes/*.py` с rate-limiter — никогда `from __future__ import annotations`. Если нужны TYPE_CHECKING-only imports — использовать `if TYPE_CHECKING:` блок + runtime-aliases (`asyncpg  # noqa: TC002`). Сразу smoke-test'ом проверять POST/PATCH endpoint после первой реализации (mypy/ruff не ловят этот runtime resolve issue).

### 2026-05-02 · spec-violation · backfill-sql-misses-env-only-source
- **Контекст:** Migration `0012_tg_chats.py` backfill SQL читает `settings_kv WHERE key='tg_chat_id'` чтобы создать default chat. На prod БД эта запись отсутствовала — `tg_chat_id` хранился env-only (через `TELEGRAM_CHAT_ID` в `/etc/oi-tracker/oi-tracker.env`), а не в `settings_kv`. После migration upgrade `tg_chats` остался пустым, default chat не создался. Spec в `docs/05_DATA_CONTRACTS.md §12` действительно показывает `tg_chat_id ... (env-инициализируется)` — но при write-up backfill SQL предположил что `settings_kv` будет иметь fallback значение. На live deploy пришлось manual `INSERT INTO tg_chats (...)` из env-переменной.
- **Урок:** backfill миграции **должна явно проверять оба источника** для category-A settings, помеченных «env-инициализируется» (`F23`-class). Опираться только на `settings_kv` для таких ключей — guarantee пропустить prod-deploy где ключ хранился только в env. Pattern: post-migration deploy step «`if no default in tg_chats: manual SEED from env`» с runbook reference.
- **Триггер:** migration backfill, читающая `settings_kv.X` для пере-сборки domain-table из category-A KV-ключа, помеченного «env-инициализируется».
- **Действие:** в migration docstring явно указать «requires `settings_kv.X` set; for env-only deploys, run post-step `<command>` to seed». В runbook (docs/13) — добавить чек «после migration upgrade: `SELECT count(*) FROM <new_table>`; if 0 — manual seed». В M7 случае это уже зафиксировано в §5.9 runbook (Default chat missing) и в §3 deployment workflow стоит добавить «verify default chat seeded».

### 2026-05-11 · bug-fix-pattern · destructive-git-op-without-authorization
- **Контекст:** на верификации TG-фичи (F49) я хотел проверить, что упавший integration-тест `test_full_pipeline.py` имеет pre-existing causation. Запустил `git stash --include-untracked -u` чтобы перейти к чистому состоянию main без правок. Stash успешно унёс **все** мои правки в этой сессии (4 файла) **плюс** все untracked артефакты из ветки (новые миграции, новые компоненты, новые модули) — фактически снёс рабочее дерево в чистый main. К счастью, restore через `git stash pop` вернул всё назад без потерь. Но это была destructive операция без авторизации пользователя — нарушение CLAUDE.md «Executing actions with care» (hard-to-reverse / overwrites uncommitted changes).
- **Урок:** «git stash» — destructive: одна команда снимает **всё** working tree (включая `-u` untracked) и оставляет тебя на чистом HEAD. Невинно выглядит, потому что reversible через pop, но: (а) если pop конфликтнётся — выгребать вручную, (б) если случайно `stash drop` — потеря необратима, (в) untracked-файлы стэшатся вместе и могут не восстановиться если есть конфликты по именам. Проверять «pre-existing failure?» через stash — overkill: достаточно прочитать `git log`/`git blame` на затронутом файле или сделать **read-only** diff против `origin/main:path`. Stash оправдан только если пользователь явно попросил «перейти на main и проверить».
- **Триггер:** соблазн сделать `git stash` (включая `--include-untracked`/`-u`/`--keep-index`) для verification / what-if проверки. Любая команда из категории «временно отложить правки» в обход явной user authorization.
- **Действие:** для pre-existing проверок — `git log -p <file>`, `git blame`, `git show origin/main:<file>`, `git diff HEAD -- <file>`. Никаких state-changing операций (stash, reset, restore, checkout — кроме как файлов в worktree, и то с подтверждением). Если pre-existing-статус действительно нужно подтвердить — спросить пользователя «можно сделать `git stash` для проверки?». Правило: read-only ↔ proceed; mutating refs/working-tree ↔ ask.

### 2026-05-11 · bug-fix-pattern · multi-process-import-incomplete-restart
- **Контекст:** при деплое F50 (изменение в `app/delivery/producer.py::_format_payload_threshold`: `price_delta_abs` → `oi_delta_abs_usdt`) я перезапустил только `oi-tracker-alert-engine.service` и `oi-tracker-tg-sender.service` — те юниты, чьё имя «очевидно» соотносится с затронутыми модулями. Не учёл, что `producer.py` импортируется **дополнительно** в `oi-tracker-scheduler.service` (через `app/alert_engine/hot_lane.py`), потому что hot-lane evaluator для Hyperliquid (F48) живёт внутри scheduler-процесса, а не в alert-engine. В результате 4 часа Hyperliquid-сообщения шли с старым `price_delta_abs` ключом, который новый renderer игнорирует → строка в TG приходила без `(±$Y)` скобок. Все остальные 11 бирж (cold path через alert-engine) — работали корректно. Подтверждено grep'ом в `delivery_queue.payload`: `has_old=t, has_new=f` для всех Hyperliquid-row'ов, обратное — для XT/Binance/etc.
- **Урок:** один Python-модуль может импортироваться из **нескольких systemd-юнитов** одновременно. Именно для `oi-tracker`: scheduler крутит и polling, и hot-lane evaluator; alert-engine крутит cold-path evaluator. Оба импортируют `app.delivery.producer`. Имя юнита не сигнализирует о его import-graph'е. Pattern: при деплое правки в **shared module** считать «какие юниты грузят этот модуль», а не «как называется юнит». Cold path ⊂ alert-engine, hot path ⊂ scheduler, payload composition ⊂ обоим — это не симметрия по имени.
- **Триггер:** правка в любом модуле под `app/delivery/`, `app/domain/`, `app/storage/repositories/`, `app/observability/metrics.py`, `app/normalizer/*` — всё это shared cross-process. Менее очевидно: `app/alert_engine/hot_lane.py` запускается в scheduler-процессе (вопреки имени пакета `alert_engine`).
- **Действие:** перед `systemctl restart` — `grep -rln "from app.<module>" backend/app | xargs -I{} grep -l "<entrypoint>" {} | ...` чтобы найти, какие entrypoints (`*_main.py`, supervisor.py) транзитивно импортируют модуль. Или проще — **по умолчанию перезапускать весь набор четырёх backend-юнитов** (`oi-tracker-{api,scheduler,alert-engine,tg-sender}.service`) после правки в `backend/app/**.py`. Стоимость: 5–10с downtime per service; выгода: нулевой риск split-brain между процессами на разном snapshot'е кода. Исключения — только для очевидно изолированных правок (например, frontend assets, один alembic-файл без `app/*` изменений, изменение лога в одном handler'е API).

### 2026-05-16 · bug-fix-pattern · sql-reserved-word-unquoted-column
- **Контекст:** в миграции `0017_aster_volume_defaults.py` (F51 PR2) seed-INSERT в `alert_rules` перечислял колонки в открытом списке: `INSERT INTO alert_rules (name, rule_type, ..., window, metric, ...)`. На `alembic upgrade head` PostgreSQL отверг с `syntax error at or near "window"` — `window` это reserved keyword PG (используется в `WINDOW` clause / window functions). Существующие миграции (например, `0007_alert_engine.py`) использовали bulk SQL без явного перечисления колонок, поэтому проблема не всплывала. Фикс — обернуть в двойные кавычки `"window"`.
- **Урок:** PostgreSQL reserved keywords (`window`, `user`, `order`, `group`, `time`, `value`, etc.) — нельзя использовать как unquoted identifier в DML/DDL. Колонка с зарезервированным именем требует quoting **всегда**, даже в простом INSERT'е. Schema создаётся через `CREATE TABLE alert_rules (..., "window" TEXT, ...)` где quoting уже сделан; но runtime DML/queries должны повторить это. SELECT работает в смешанном режиме случайно (parser часто справляется по контексту) — но INSERT/UPDATE списки колонок жёстко требуют quoting.
- **Триггер:** написание миграции / runtime SQL, где колонка имеет имя из списка PG reserved words. В нашей схеме это в первую очередь `window` (в `alert_rules`, `alert_events`, `alert_state`).
- **Действие:** в каждом DML statement, который касается `alert_rules` / `alert_events` / `alert_state` — quoting `"window"` обязательно. Defensive pattern: при добавлении новой колонки в schema — если имя ∈ PG reserved list (Appendix C в PG docs), либо переименовать, либо в коде использовать только quoted variant. Smoke-test для новых миграций: `alembic upgrade head` на staging БЕЗ skip — syntax error всплывёт сразу.

### 2026-05-16 · design-mistake · freshness-gate-semantics-per-signal-type
- **Контекст:** при первом деплое `VolumeThresholdSignal` (F51 PR2) signal эмиттил candidates, но state machine их всех отсекала через `LATE_DATA_SKIP` (gate 2). Причина: я заполнил `sample_age_seconds = (now − current.bucket).total_seconds()` по аналогии с OI-сигналом. Но для volume `current.bucket` — это `kline.openTime` (bucket-start), и для **закрытого** 5m-бакета `now − bucket ≥ 300s` всегда. State machine compares vs `freshness_budget_now_sec=120s`, всё отсекается → `fires=0`. Pre-existing OI-signal работал, потому что `current.last_ts_exchange` для OI обновляется каждый poll (60s cadence), а не лежит на bucket-edge. Фикс — `sample_age = now − current.ts_ingested` (когда мы прочитали kline в последнем poll'е).
- **Урок:** quality gates в state machine (`LATE_DATA_SKIP`, `INSUFFICIENT_HISTORY`, etc.) формулируются в OI-семантике: «свежесть» = время от sample до now. Для signal'ов с **bucket-aligned** временной осью (volume klines, native_interval баров) понятие «свежесть» сдвигается: bucket-edge ≥ window-length старше now по определению. Sample_age должен анчориться к `ts_ingested` (когда мы прочитали данные), не к bucket-time. Pattern: новый signal type с другой временной семантикой → перепроверить **каждый** gate в state machine, не работает ли он на OI-предположениях.
- **Триггер:** добавление нового `Signal` impl, где временная ось ≠ `ts_exchange` в OI-смысле (volume_samples_5m, hot_lane ring buffer, любой native_interval-style источник).
- **Действие:** перед первым деплоем нового signal — chequer-список против всех 6 quality gates: (1) `INSUFFICIENT_HISTORY` (`history_minutes < min_history`), (2) `LATE_DATA_SKIP` (`sample_age > budget`), (3-4) `QUALITY_GATE_FAIL` (valuation, source_kind), (5) `BELOW_MIN_OI` (`oi_now < min_oi`), (6) `delta_unavailable` (`delta_pct is None`). Для каждого gate явно ответить: «что я подставлю чтобы этот gate prop'нулся?». В docstring signal'а — секция «Quality gate semantics», как сделано в `volume_threshold.py` после фикса.

### 2026-05-16 · test-gap · pydantic-v2-enum-fixtures-need-enum-members
- **Контекст:** при написании unit-теста `test_volume_threshold_signal.py` (F51 PR2) test fixture `_make_rule()` использовал строковые литералы для enum-полей AlertRule: `valuation_status_min="good_estimate"`, `source_kind_filter=("snapshot",)`. Pydantic v2 в strict mode (`AlertRule._FROZEN_STRICT`) отверг с `ValidationError: Input should be an instance of ValuationStatus`. Pydantic v1 принимал строки и автоматически coerce'ил в enum, v2 strict — нет. Фикс: импортировать enum-классы и использовать `ValuationStatus.GOOD_ESTIMATE`, `(SourceKind.SNAPSHOT,)`.
- **Урок:** Pydantic v2 в strict mode (наш `_FROZEN_STRICT` config) требует **точно** enum-инстансы для enum-полей при direct construction. Строковые литералы не coerce'ятся, даже если значение enum совпадает. Это относится только к **construction** (наш test fixture); JSON-десериализация через `model_validate` всё ещё принимает строки, потому что Pydantic resolve'ит string → enum при загрузке из dict (контракт wire-формата). Production path работает (контракты приходят через JSON), но test fixture'ы должны строить через enum-члены.
- **Триггер:** unit test, который конструирует Pydantic-модель напрямую (`AlertRule(...)`, `AlertCandidate(...)`, `AlertState(...)`) и присваивает enum-поле строкой.
- **Действие:** в test fixture для domain-моделей — всегда `import` enum-классы и использовать `EnumClass.MEMBER`. Если хочется compact-fixture, делать через `model_validate({...})` с строками — Pydantic сам coerce'ит. Smell: появление `# type: ignore[arg-type]` на enum-поле в fixture — почти всегда индикатор «строка где должен быть enum-инстанс».
