# 01 · Product Spec

> Что строим, для кого, и как мы поймём, что построили правильно.
> Связано: `00_DECISIONS_LOG.md` (источник всех решений), `02_ARCHITECTURE.md` (как).

---

## 1. Mission

`oi-tracker` — это персональная система мониторинга Open Interest (OI) на USDT-M perpetual фьючерсах 12 криптовалютных бирж, отправляющая алерты в Telegram при значимых изменениях OI и предоставляющая веб-интерфейс для просмотра истории и настройки правил.

**Назначение:** дать одному пользователю возможность вовремя замечать изменения позиционирования рынка по конкретным символам, чтобы реагировать на них в трейдинге.

---

## 2. Target user

**Кто:** один пользователь — заказчик проекта.

**Контекст использования:**
- Трейдинг криптовалютных перпетуальных фьючерсов.
- Telegram — основной канал получения сигналов (групповой чат, в котором заказчик участвует).
- Веб-интерфейс — для anal'iza истории и тонкой настройки порогов.
- Использование в одиночку, без команды.

**Что НЕ является пользователем:**
- Многопользовательская SaaS (нет пользовательских аккаунтов).
- Внешний API-потребитель (нет публичного API).
- Мобильное приложение (только web + TG).

---

## 3. In-scope

### 3.1 Функциональные возможности

**Сбор данных:**
- Опрос 12 бирж раз в минуту: Binance, Bybit, OKX, Bitget, Gate.io, MEXC, KuCoin, HTX, Hyperliquid, Aster, Bitmart, XT.
- Только USDT-M perpetual контракты.
- Автоматическое обновление списка торгуемых символов каждые 5 минут.
- Автоподхват новых листингов; делистнутые символы прекращают опрашиваться, но история сохраняется.
- Изоляция ошибок: одна биржа недоступна → остальные продолжают работать.

**Хранение:**
- Сырые нормализованные точки в TimescaleDB hypertable.
- Continuous aggregates 5m / 15m / 30m.
- Retention 90 дней (как для raw, так и для агрегатов).
- Compression после 7 дней.

**Расчёт метрик:**
- Δ_pct по `oi_coins` для окон 5/15/30 минут.
- `oi_notional_usdt` как производное display-поле.
- Valuation status (authoritative / good_estimate / low_confidence) для каждой точки.

**Алерты:**
- 4 типа сигналов: threshold, confirmed, divergence, exchange-health (consensus sunset per `00 F40`).
- Smart cooldown: повторный fire при усилении Δ ≥ 1.5×.
- Минимальная история `max(30, 2 × window)` минут перед включением алертов на новом символе.
- Whitelist бирж + blacklist символов.
- Floor-фильтр по абсолютному OI (default 5_000_000 USDT).

**Доставка:**
- Telegram групповой чат (sender-only бот).
- Web dashboard (live-таблица, страница символа, журнал алертов, settings).
- SSE для real-time обновлений UI.

### 3.2 Поддерживаемые биржи

| # | Биржа | Тип сбора |
|---|---|---|
| 1 | Binance | snapshot per symbol |
| 2 | Bybit | native interval (5m/15m/30m) |
| 3 | OKX | snapshot per symbol |
| 4 | Bitget | snapshot per symbol |
| 5 | Gate.io | snapshot per symbol |
| 6 | MEXC | snapshot per symbol |
| 7 | KuCoin | native interval (5m/15m/30m) |
| 8 | HTX | snapshot per symbol |
| 9 | Hyperliquid | snapshot per symbol |
| 10 | Aster | snapshot per symbol (binance-like) |
| 11 | Bitmart | snapshot bulk (один endpoint = все контракты + OI) |
| 12 | XT | bulk snapshot (один endpoint = все контракты) |

---

## 3.3 UX-сценарий (single-user, end-to-end)

Описывает как один пользователь живёт с системой от первого запуска до инцидента и восстановления.

### Phase A — Cold start (день 0)

1. Получает доступ к `https://oi-tracker.robot-detector.ru/` (домен — `15_DEPLOYMENT_INFRA.md F12`).
2. Открывает `/` (Dashboard) — пусто или почти пусто: data только начала собираться, `min_history_minutes` ещё не накоплено.
3. Идёт в `/settings` → tab Telegram → ставит свой `tg_chat_id` группового чата (где уже добавлен бот). Включает `tg_dry_run=true` на первый час, чтобы убедиться, что алерты осмысленны.
4. На странице Settings → tab Alerts видит 6 default-порогов из `00_DECISIONS_LOG.md F6`. Не меняет (defaults — это «безопасный» старт).
5. Через ~30 минут `min_history_minutes_floor` (default 30 мин, `F8`) накапливается, dashboard оживает: видно `delta_5m_pct` для большинства символов.

### Phase B — Tuning (первые 24 часа)

1. Получает первые алерты в TG в режиме dry-run (`oi-tracker-tg-sender.service` логирует, в чат не шлёт).
2. Просматривает `/alerts` — журнал `alerts_sent` со статусом `tg_delivery_status='sent'` (для `dry_run` это означает «успешно эмулировано»).
3. Понимает, что 6 алертов в час по 5-минутным окнам — слишком много. Идёт в Settings → Alerts:
   - Поднимает `up_5m / down_5m` пороги с default до более жёсткого значения (например, default 3% → 5%).
   - Поднимает `min_oi_notional_default_usdt` с 5М до 10М, чтобы отрезать шум на средних альтах.
4. Сохраняет — все правки идут через `PUT /api/v1/settings` (rate-limit 5/min, см. `10_DELIVERY_LAYER.md §4.8`).
5. Через час смотрит `/alerts` (новые fire-события) — частота приемлемая. Отключает `tg_dry_run`.
6. С этой минуты алерты идут в реальный TG. e2e latency P95 ≤ 90s (см. NFR-1).

### Phase C — Steady state (первая неделя)

1. Заходит в UI несколько раз в день, без TG не пользуется кроме как для алертов.
2. Иногда кликает по символу в dashboard → `/symbol/:canonical` → видит график OI на 24h по 12 биржам.
3. Замечает, что Hyperliquid даёт «странные» дельты (биржа специфическая) — добавляет её в symbol_blacklist для нескольких токенов: Settings → Filters → blacklist text input.
4. Раз в день получает `exchange-health` алерт от MEXC: «coverage_ratio < 95% последние 30 минут». Открывает `/alerts` — видит периодические `exchange_stale` от MEXC. Предполагает временные проблемы биржи, не вмешивается.

### Phase D — Incident (день 8)

1. Перестают приходить алерты вообще. Открывает UI — `/` показывает «disconnected» (SSE отвалился).
2. Refresh браузера → коннект восстановлен, но колонки `delta_*` все пусты.
3. Идёт в `/api/v1/health` — видит `{"db":"ok", "scheduler":"degraded"}`. Понимает, что scheduler жив (БД отвечает) — но cycles не идут.
4. Вспоминает runbook `13_OPERATIONS.md §5.2` (БД упала) — проверяет:
   ```bash
   ssh server
   sudo systemctl status oi-tracker-scheduler.service
   ```
5. Видит `Restart=always` отрабатывает, но падает каждые 10s. `journalctl -u oi-tracker-scheduler` — ошибка коннектора Bybit: «schema_drift». Биржа поменяла формат ответа.
6. Действие: следуя `13 §5.6`, обновляет коннектор Bybit, bump'ит `connector_version`, deploy через `13 §7.1`. Альтернатива (если фикс не сразу): убирает `bybit` из `exchange_whitelist` через UI Settings → Filters, чтобы scheduler перестал на ней падать.
7. Service зеленеет, dashboard оживает за 1-2 минуты.

### Phase E — Routine (день 30+)

1. Открывает раз в неделю `/symbol/:c` для нескольких ключевых тикеров на period=`30d` — оценивает структурные сдвиги OI.
2. Раз в месяц получает алерт `disk_usage > 80%` (`13 §5.4`) — проверяет, что compression policy жива:
   ```sql
   SELECT * FROM timescaledb_information.jobs WHERE proc_name='policy_compression';
   ```
   Если нужно — force compress (см. runbook).
3. Бэкапы пока **не активированы** (см. `01_PRODUCT_SPEC.md §10`, `13 §9`). Через ~3 месяца, когда история становится ценной, активирует pg_dump nightly по плану `13 §9.2` (RPO ≤ 24ч / RTO ≤ 2ч).

### Что в этом сценарии важно

- **Самостоятельность.** Пользователь решает 95% задач через UI, только в инцидентах лезет в SSH.
- **Безопасные настройки в UI.** Всё, что можно «крутить» без миграций и code changes, доступно (см. `10_DELIVERY_LAYER.md §4.8` категории A+B).
- **Опасные настройки скрыты.** `polling_interval_sec`, `chunk_time_interval` через UI не меняются (категория C).
- **Алерты как первичный канал.** UI второй по частоте использования, после TG.
- **Observability на проде.** journalctl, /metrics, /alerts page — всё, что нужно для самодиагностики.

---

## 4. Out-of-scope

Эти возможности явно НЕ входят в проект. Если они потребуются позже — это отдельный проект/фаза.

- ❌ **Трейдинг и ордера.** Система только наблюдает, не исполняет сделок.
- ❌ **Multi-user.** Один пользователь, без аутентификации, без user_id.
- ❌ **USDC-M / COIN-M / Options.** Только USDT-M perpetual.
- ❌ **Spot OI / Futures без OI.** Только OI на perpetual'ах.
- ❌ **Социальные функции** (комментарии, шаринг, копи-трейдинг).
- ❌ **Мобильное приложение.** TG-уведомления + web UI достаточны.
- ❌ **Публичный REST/GraphQL API.** Внутренний API только для собственного фронта.
- ❌ **Бэкапы в v1.** Будут добавлены в future-work, но не сейчас.
- ❌ **Cluster / HA.** Single-node bare-metal deployment.
- ❌ **Биржи вне списка из §3.2.** Добавление новой биржи = расширение proекта.
- ❌ **Анализ ленты ордеров, liquidations, funding rate.** Только OI и цена (для notional конверсии).
- ❌ **Бэктест стратегий, симулятор.** Только real-time мониторинг.
- ❌ **Кастомные индикаторы.** Только Δ OI в %.
- ❌ **i18n / RTL.** UI на русском или английском, единственный язык.

---

## 5. Functional requirements

### FR-1 · Сбор OI
Система получает текущий OI по всем активным USDT-M perpetual символам 12 бирж раз в минуту.

### FR-2 · Нормализация
Все OI приводятся к единой единице измерения (количество монет — `oi_coins`) с производным `oi_notional_usdt`. Каждая точка имеет `valuation_status`, отражающий качество конверсии.

### FR-3 · Расчёт окон
Для каждого `(exchange, canonical_symbol)` поддерживаются окна 5/15/30 минут с Δ в %.

### FR-4 · Алерты
При пересечении заданного порога Δ — отправка алерта в Telegram. Поддерживаются 5 типов сигналов (см. §3.1).

### FR-5 · Антиспам
Smart cooldown 15 минут default; повторный fire только при усилении Δ ≥ 1.5×.

### FR-6 · Веб-интерфейс
Dashboard, Symbol page, Alerts log, Settings — см. `10_DELIVERY_LAYER.md`.

### FR-7 · История
Хранение 90 дней нормализованных точек с возможностью графиков по любому окну (1h / 6h / 24h / 7d / 30d / 90d).

### FR-8 · Настройки
Из UI редактируются: пороги (per window × up/down), cooldown, whitelist бирж, blacklist символов, Telegram chat_id.

### FR-9 · Reprocessing
При смене формул нормализации возможен replay по `raw_exchange_events`. Backfill / replay не создают retro-fire алертов.

### FR-10 · Изоляция ошибок
Падение одной биржи не блокирует остальные. Per-exchange circuit breaker.

---

## 6. Non-functional requirements

### NFR-1 · Latency
End-to-end (тик биржи → отправка в TG):
- **P95 ≤ 90 секунд.**
- **P99 ≤ 180 секунд.**

### NFR-2 · Coverage
- `symbol_coverage_ratio ≥ 95%` per exchange (доля активных символов, получивших точку в текущем цикле).
- Падение ниже 95% триггерит exchange-health алерт.

### NFR-3 · Availability
- Best-effort (single-node, без HA).
- Перезапуск через systemd при падении процесса.
- При падении базы — graceful degradation: коннекторы пишут в local buffer (если применимо), TG-sender ставит сообщения в очередь.
- Бэкапы в v1 не настраиваются (см. `00_DECISIONS_LOG.md` Q7).

### NFR-4 · Storage
- ~5–7 М записей в день в `oi_samples`.
- ~150 GB raw за 90 дней до compression, ~10–15 GB после.
- Compression после 7 дней (10–20× сжатие).

### NFR-5 · Observability
- Prometheus метрики per component.
- Grafana дашборды: System, Exchanges, Alerts, Latency.
- Loki для structured logs.
- Подробности — в `12_OBSERVABILITY_SLO.md`.

### NFR-6 · Security
- TG bot token — env-only, не в БД.
- Web UI — без аутентификации, доступ ограничивается на сетевом уровне (VPN / firewall / iptables).
- БД — `pg_hba.conf` ограничивает доступ только с localhost.
- Никаких приватных API биржей не используется (только публичные endpoint'ы).

### NFR-7 · Maintainability
- Каждый коннектор — отдельный модуль с единым интерфейсом (см. `11_EXCHANGE_ADAPTERS/_BASE_CONNECTOR.md`).
- Добавление 13-й биржи = новый файл в `app/exchanges/`, реализующий `BaseConnector`.
- Все нормализационные правила версионируются (`normalization_version`).

### NFR-8 · Testability
- Coverage targets: ≥ 80% для core модулей (`normalizer`, `alert_engine`, `time_series_storage`).
- Contract tests per exchange с replay реальных payload-фикстур.
- E2E тесты на синтетических данных.

---

## 7. Success criteria / KPIs

| Метрика | Целевое значение | Источник |
|---|---|---|
| End-to-end latency P95 | ≤ 90s | Prometheus histogram `oi_alert_e2e_latency_seconds` |
| End-to-end latency P99 | ≤ 180s | то же |
| Symbol coverage ratio | ≥ 95% per exchange | `oi_collector_symbol_coverage_ratio` |
| Connector uptime | ≥ 99% per exchange (monthly) | `oi_collector_up` |
| Алерт false-positive rate | < 5% | self-reported, опросы пользователя |
| Алерт missed rate | 0 крупных движений (>10% за 5 мин на BTC/ETH) | self-reported |
| TG delivery success rate | ≥ 99% | `oi_tg_delivery_success_total` / `oi_tg_delivery_total` |
| UI load time (dashboard) | ≤ 1s @ 5000 rows | manual measure |

---

## 8. Constraints

### C-1 · Single user
Архитектура упрощена под single-user: нет аутентификации, user_id не существует. Расширение на multi-user — отдельный проект.

### C-2 · Bare-metal deployment
Без Docker. Систему запускают как 4 systemd-юнита на одном Linux-сервере. Это ограничивает горизонтальное масштабирование.

### C-3 · No backups (v1)
Потеря данных при сбое диска — допустима. Восстановление = свежий запуск, история обнуляется. Это сознательный trade-off; будет пересмотрено когда система проработает достаточно долго, чтобы накопленная история стала ценной.

### C-4 · Public exchange APIs only
Не используем приватные API (требуют ключей, лимиты, риски бана). Только публичные market endpoints. Если биржа ограничит публичный OI endpoint — придётся искать обходной путь.

### C-5 · USDT-M only
COIN-M, USDC-M, опционы, споты — out of scope. Игнорируются на уровне фильтрации в `sync_instruments()`.

### C-6 · Latency floor
P95 ≤ 90s — это floor, диктуемый 60s polling cadence + 30s обработки. Опускаться ниже без изменения cadence невозможно.

---

## 9. Risks

| Риск | Вероятность | Влияние | Митигация |
|---|---|---|---|
| Биржа меняет формат API | Средняя | Высокое | `raw_exchange_events` хранятся всегда → replay при обновлении парсера; `normalization_version` |
| Биржа вводит rate limit | Высокая | Среднее | Token bucket per-exchange; tiered polling в крайнем случае |
| Биржа закрывает публичный OI endpoint | Низкая | Очень высокое | Нет полной митигации; биржа удаляется из whitelist до восстановления |
| Disk full | Средняя | Высокое | Retention policy 90 дней + compression; алерт `disk_usage > 80%` |
| TG bot token leaked | Низкая | Среднее | Token в env-файле с правами `0600`; ротация при подозрении |
| Time skew (наш сервер vs биржа) | Высокая | Низкое | Хранение `ts_exchange` отдельно от `ts_ingested`; алерт при дрейфе > 30s |
| Single-node failure | Средняя | Среднее | Systemd auto-restart; ручная диагностика |
| Спам алертов на flash-crash | Средняя | Низкое | Smart cooldown + min_oi_notional + valuation_status filter |

---

## 10. Future work (явно отложено)

Следующие задачи зафиксированы как future work, но НЕ в текущей фазе:

- Бэкапы (pg_dump nightly + WAL-G в S3).
- Расширение на USDC-M / COIN-M.
- Добавление новых бирж.
- Funding rate, liquidations, OI-by-account-type.
- Веб-настройка типов сигналов из UI (сейчас правила Q2 редактируются через DB / config).
- Mobile push notifications.
- Multi-user mode.
- Дашборд алертов в Telegram с inline-кнопками (требует переход с sender-only на full bot).

---

## 11. Versioning

Этот документ — `1.0`.

При значимых изменениях scope (in/out, новые SLO, новые НФТ) — bump major version, фиксация в git history.

При корректировках формулировок — patch version, обновление даты в шапке.

Связанные файлы (`02_ARCHITECTURE.md`, `00_DECISIONS_LOG.md`) синхронизируются при изменении этого документа.
