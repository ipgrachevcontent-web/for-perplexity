# 18 · Design System

> Single-user trader tool. UI — второй по частоте канал после TG. Рассчитан на long sessions, низкочастотные взгляды и информационную плотность. Не маркетинговый продукт.
> Связано: `01_PRODUCT_SPEC.md §3.3`, `10_DELIVERY_LAYER.md §6`, `16_ROADMAP.md M4`.

---

## 1. Принципы (декларация — всё остальное следует отсюда)

| # | Принцип | Что это значит на практике |
|---|---|---|
| **P1** | **Density over whitespace.** | Это терминал, а не лендинг. Trader’у важна плотность сигнала. Padding малый, шрифт компактный, ряды таблицы — 32–36px, не 56px. |
| **P2** | **Signal first, chrome last.** | Цвет несёт смысл (Δ ↑/↓, freshness, валюация). Хром — нейтрально-серый, не борется со значениями. |
| **P3** | **Tabular numbers everywhere.** | Числа выровнены по разрядам. Mono-шрифт + `font-variant-numeric: tabular-nums`. Никогда не «прыгают» при SSE-апдейте. |
| **P4** | **Stable layout under live updates.** | Никаких layout shift'ов от SSE. Высота строк, ширины колонок, grid — детерминированы. Новые данные → diff содержимого, не reflow. |
| **P5** | **Motion is informational, not decorative.** | Анимации — только состояние (fade-in алерта, мерцание freshness, micro-flash изменившейся ячейки). Никаких параллаксов, hero-анимаций, реверсивных bouncy-easing. |
| **P6** | **Dark-first.** | Основная тема — тёмная (часы у экрана, ночные сессии). Светлая — секондари (опционально). Контраст ≥ 4.5:1 для текста, ≥ 3:1 для UI-элементов. |
| **P7** | **Keyboard-first power user.** | Все списки и формы — клавиатурно-навигабельные. Глобальные шорткаты: `g d`, `g a`, `g s`, `/` (поиск), `?` (help). |
| **P8** | **Truth in state.** | Каждый компонент явно показывает: connection (SSE), freshness (last update age), valuation_status. Если данные несвежие — это видно немедленно, не зарыто. |

Иерархия конфликтов следует `CLAUDE.md §0`: корректность сигнала > плотность > эстетика > краткость кода.

---

## 2. Цветовая система

### 2.1 Подход

- Цвет хранится в **CSS variables** в OKLCH (для предсказуемых перцептуальных шкал) с fallback в HSL.
- Tailwind конфиг читает variables, не hardcode'ит hex.
- Семантические токены (`--color-fg-primary`, `--color-bull`) — единственное, что использует UI. Raw-шкалы (`--slate-50…950`) — только для генерации семантических.

### 2.2 Raw scales (генерируются один раз)

| Scale | Назначение |
|---|---|
| `slate` 50–950 | нейтральный chrome (фоны, бордеры, текст) |
| `cyan` 50–950 | brand accent (линки, focus ring, primary action) |
| `emerald` 50–950 | bull / positive Δ |
| `rose` 50–950 | bear / negative Δ |
| `amber` 50–950 | warning / degraded freshness |
| `violet` 50–950 | информационные акценты (divergence type) |

Шкалы в OKLCH со ступенчатым lightness `[0.985, 0.97, 0.93, 0.87, 0.78, 0.66, 0.55, 0.44, 0.32, 0.22, 0.13]`.

### 2.3 Семантические токены — Dark theme (default)

```css
:root[data-theme="dark"] {
  /* Surfaces (от глубокого к приподнятому) */
  --surface-base:        oklch(0.16 0.01 250);   /* фон body */
  --surface-1:           oklch(0.20 0.01 250);   /* карточки, sidebar */
  --surface-2:           oklch(0.24 0.01 250);   /* hover, отдельные строки */
  --surface-3:           oklch(0.28 0.01 250);   /* меню, popover, dropdown */
  --surface-overlay:     oklch(0.10 0.01 250 / 0.72); /* modal backdrop */

  /* Borders */
  --border-subtle:       oklch(0.30 0.005 250);
  --border-default:      oklch(0.36 0.005 250);
  --border-strong:       oklch(0.50 0.005 250);

  /* Foreground */
  --fg-primary:          oklch(0.96 0 0);        /* основной текст */
  --fg-secondary:        oklch(0.78 0 0);        /* labels, secondary */
  --fg-muted:            oklch(0.58 0 0);        /* meta, timestamps */
  --fg-disabled:         oklch(0.42 0 0);

  /* Brand / accent */
  --accent:              oklch(0.78 0.14 220);   /* cyan-400ish */
  --accent-hover:        oklch(0.84 0.14 220);
  --accent-fg:           oklch(0.18 0 0);        /* текст поверх accent */

  /* Semantic — signals */
  --bull:                oklch(0.78 0.17 150);   /* +Δ, fresh */
  --bull-bg:             oklch(0.78 0.17 150 / 0.12);
  --bear:                oklch(0.70 0.20 25);    /* -Δ */
  --bear-bg:             oklch(0.70 0.20 25 / 0.12);
  --warn:                oklch(0.82 0.16 75);    /* degraded */
  --warn-bg:             oklch(0.82 0.16 75 / 0.12);
  --info:                oklch(0.78 0.14 220);   /* SSE connected */
  --neutral:             oklch(0.62 0.005 250);  /* null, n/a */

  /* Valuation status */
  --val-authoritative:   var(--bull);
  --val-good-estimate:   oklch(0.78 0.10 180);
  --val-low-confidence:  var(--warn);

  /* Focus ring */
  --focus-ring:          oklch(0.78 0.14 220 / 0.55);
}
```

### 2.4 Δ — цветовое кодирование (критично для дашборда)

Не binary green/red. Δ-ячейки используют **intensity-aware градации**, чтобы 0.3% и 12% не выглядели одинаково:

| \|Δ_pct\| диапазон | text color | background |
|---|---|---|
| `< 0.5%` | `--fg-muted` | none |
| `0.5%..2%` | `--bull / --bear` (alpha 0.85) | none |
| `2%..5%` | `--bull / --bear` (alpha 1.0) | `--bull-bg / --bear-bg` (alpha 0.06) |
| `≥ 5%` | `--bull / --bear` (alpha 1.0) | `--bull-bg / --bear-bg` (alpha 0.14) + `font-weight: 600` |

Это позволяет глазу скан-ридить таблицу и сразу видеть «горячие» строки без сортировки.

### 2.5 Идентичность бирж (12 цветовых меток)

Per-exchange chip color на dashboard. Не используем брендовые цвета бирж (Binance-yellow и Bybit-yellow трудноразличимы) — назначаем **наш** набор перцептуально-различимых hue'ов:

```ts
export const EXCHANGE_HUE: Record<Exchange, number> = {
  binance: 48, bybit: 28, okx: 0, bitget: 320, gate: 280, mexc: 220,
  kucoin: 180, htx: 145, hyperliquid: 95, aster: 60, bitmart: 200, xt: 250,
};
// рендерим через oklch(0.74 0.15 var(--hue))
```

Chip — `text-fg-primary` на `oklch(0.30 0.10 var(--hue))` фоне с `oklch(0.55 0.15 var(--hue))` 1px бордером.

### 2.6 Light theme

Та же семантика, но `--surface-*` инвертированы (от almost-white к light slate-100). Bull/bear hue'и — чуть глубже (`lightness 0.55-0.62`), чтобы не выгорали на белом. **Light theme ≠ default**, переключатель в Settings.

---

## 3. Типографика

### 3.1 Шрифты

| Семейство | Назначение | Источник |
|---|---|---|
| **Inter** (var) | UI, заголовки, тексты | self-host woff2 (`/fonts/Inter.var.woff2`) |
| **JetBrains Mono** (var) | числа, code, key labels | self-host woff2 |

`font-display: swap`. Самохост — не CDN (single-user, bare-metal, никаких внешних трекеров — гармонирует с `noindex` headers `F13`).

`font-feature-settings`:
- Inter: `"cv11", "ss01", "ss02"` (читабельные `g`, `l`, `1`).
- JetBrains Mono: `"cv02", "cv14"` (без italic ligatures).

### 3.2 Шкала размеров

Modular scale 1.125 (major second), компактная под density-первый принцип:

| Token | px | line-height | letter-spacing | Use |
|---|---|---|---|---|
| `text-2xs` | 10 | 14 | +0.04em | badges, micro-meta |
| `text-xs` | 11 | 16 | +0.02em | table headers, chips |
| `text-sm` | 12 | 18 | 0 | **default body, table cells** |
| `text-base` | 13 | 20 | 0 | inputs, controls |
| `text-md` | 15 | 22 | 0 | section labels |
| `text-lg` | 17 | 24 | -0.005em | subheaders |
| `text-xl` | 20 | 28 | -0.01em | page titles |
| `text-2xl` | 24 | 32 | -0.015em | hero numbers (rare) |

> Заметка: дефолт `text-sm = 12px`, не 14/16. Это сознательное отклонение от веб-стандарта в пользу плотности (как в Bloomberg/TradingView). Контраст и шрифтовая обработка делают это читабельным.

### 3.3 Веса

`400` (regular), `500` (medium — выделение в таблице), `600` (semibold — заголовки, hot Δ), `700` (используем редко, hero numbers).

### 3.4 Числа — отдельный класс

```css
.tabular {
  font-family: 'JetBrains Mono', ui-monospace, monospace;
  font-variant-numeric: tabular-nums slashed-zero;
  font-feature-settings: "tnum", "zero";
}
```

Все колонки `OI ($)`, `Δ`, `price`, `last_sample_age` — `.tabular`. Это единственный способ, чтобы 12 строк цены `67,420.50` и `1,240.05` не плыли по горизонтали при SSE update.

### 3.5 Форматтеры (UX-контракт)

| Что | Формат | Пример |
|---|---|---|
| `oi_notional_usdt` | k/M/B humanized, 2 знака | `$1.24B`, `$5.30M`, `$12.5K` |
| `oi_coins` | `Intl.NumberFormat`, max 4 frac | `12,345.6789` |
| `delta_pct` | sign + 1 frac + `%`, no spaces | `+6.2%`, `−0.4%` (минус — `−`, не дефис) |
| `price` | `Intl.NumberFormat`, `maximumFractionDigits` зависит от величины | `$67,420.50`, `$0.00012345` |
| `ts_exchange` | `HH:MM:ss` локальное в таблице, `YYYY-MM-DD HH:MM:ss UTC` в tooltip | |
| `last_sample_age_sec` | `5s`, `2m 14s`, `1h 02m` (см §6.2 freshness indicator) | |

Все форматтеры — в `frontend/src/format/` как чистые функции, тестируются `vitest`.

---

## 4. Spacing, layout, grid

### 4.1 Шкала отступов

Базовая единица **4px**, шкала Tailwind-совместимая. На UI используются **только** эти ступени:

`0, 1 (4), 2 (8), 3 (12), 4 (16), 5 (20), 6 (24), 8 (32), 10 (40), 12 (48), 16 (64)`.

Дробные / произвольные значения — запрещены lint-правилом (`no-arbitrary-spacing`).

### 4.2 Density mode

Settings → Density: `comfortable` | `compact` (default `compact`). Меняет `--row-height`, `--cell-padding`:

| token | comfortable | compact |
|---|---|---|
| `--row-height` | 36px | 28px |
| `--cell-padding-y` | 8px | 4px |
| `--cell-padding-x` | 12px | 8px |
| `--font-size-table` | 13px | 12px |

Хранится в localStorage + `settings_audit_log` через PUT `/settings/ui_density` (новый key категории A).

### 4.3 Layout grid

```
┌─ AppShell (100dvh) ─────────────────────────────────────────┐
│  TopBar  56px                                               │
├──────────────────────────────────────────────────────────────┤
│ Sidebar │ Page                                              │
│  72px   │  ┌─ Page header — 48px ─────────────────────────┐ │
│  (icon  │  │ title • filters • status • actions           │ │
│   only) │  └────────────────────────────────────────────────┘
│         │  ┌─ Main content (fluid)────────────────────────┐ │
│         │  │                                              │ │
│         │  └────────────────────────────────────────────────┘
└──────────────────────────────────────────────────────────────┘
```

- **AppShell** — fixed height `100dvh`, без скролла body.
- **Sidebar** — иконочный, 72px (rail). Раскрытие — hover/click → 240px overlay (не push). Содержит навигацию + connection-status badge внизу.
- **Page** — internal scroll containers. Таблица на dashboard скроллится только в своей области, header и фильтры sticky.
- **Min viewport:** 1280×720. Меньше — показываем баннер «оптимизировано для desktop». Mobile — out of scope (`01_PRODUCT_SPEC.md §4`).
- **Max content width:** unlimited на dashboard (трейдер на 21:9 хочет всё видеть). Symbol page — `max-width: 1600px`, центрирована.

### 4.4 Sticky behavior

- Page header — `position: sticky; top: 0` внутри scroll container.
- Table head — sticky. `z-index: 10`.
- Filters bar — sticky под page header при скролле > 200px (с лёгкой тенью при detached state).

---

## 5. Радиусы, тени, elevation

### 5.1 Радиусы

| Token | px | Use |
|---|---|---|
| `radius-none` | 0 | таблицы, ряды |
| `radius-sm` | 4 | chips, badges, кнопки внутри строк |
| `radius-md` | 6 | inputs, primary buttons, table-wrapper |
| `radius-lg` | 10 | карточки, popover, modal |
| `radius-full` | 9999 | pill (status, count badges) |

Никаких 16/20/24px. Большие радиусы делают финансовый UI «игрушечным».

### 5.2 Тени (используются скупо)

В dark-first elevation выражается **в первую очередь поверхностью** (`--surface-1/2/3`), а не shadow. Тени включаются только для overlay'ев:

```css
--shadow-popover:  0 1px 2px oklch(0 0 0 / 0.3),
                   0 8px 24px oklch(0 0 0 / 0.35);
--shadow-modal:    0 2px 4px oklch(0 0 0 / 0.4),
                   0 24px 48px oklch(0 0 0 / 0.55);
--shadow-toast:    0 1px 2px oklch(0 0 0 / 0.3),
                   0 12px 32px oklch(0 0 0 / 0.45);

/* glow-эффекты только для focus и live-status */
--glow-focus:      0 0 0 2px var(--focus-ring);
--glow-live:       0 0 0 0 var(--bull / 0.5);  /* основа для pulse */
```

Карточки, sidebar, header — **без shadow**, только border `1px var(--border-subtle)` либо без border (контраст surface).

### 5.3 Elevation rules

| Level | Surface | Shadow | Когда |
|---|---|---|---|
| 0 | `--surface-base` | — | body |
| 1 | `--surface-1` | — | sidebar, page content cards |
| 2 | `--surface-2` | — | row hover, selected |
| 3 | `--surface-3` | `--shadow-popover` | dropdown, tooltip, command menu |
| 4 | `--surface-3` | `--shadow-modal` | dialog, drawer |
| 5 | `--surface-3` | `--shadow-toast` | toast |

---

## 6. Анимация

### 6.1 Бюджет и принципы

- **Не анимируем layout** (`width`, `height`, `top` и т.д.). Только `opacity`, `transform`, `background-color`, `color`, `box-shadow`.
- **Никакого spring bounce.** Всегда `ease-out` или `ease-in-out`. Финансовый UI = предсказуемый.
- **Reduced motion обязательно:** все нетривиальные анимации обёрнуты `@media (prefers-reduced-motion: reduce) { ... }` с переходом в мгновенный flash.
- **SSE-throttle:** не более 1 анимации flash'а на ячейку в 200ms (см. `10_DELIVERY_LAYER.md §6.5`).

### 6.2 Длительности и easing

| Token | duration | easing | Где |
|---|---|---|---|
| `motion-instant` | 80ms | linear | hover state, button press |
| `motion-fast` | 140ms | `cubic-bezier(.2,.8,.2,1)` | toggles, popover open |
| `motion-base` | 220ms | `cubic-bezier(.2,.8,.2,1)` | modal/drawer enter |
| `motion-slow` | 360ms | `cubic-bezier(.2,.8,.2,1)` | page transitions (если решим использовать) |
| `motion-flash` | 600ms | ease-out | cell flash on update |
| `motion-pulse` | 2000ms infinite | ease-in-out | live indicator dot |

### 6.3 Каноничные анимации

**`flash-update`** — ячейка-таблицы получила новое значение:
```css
@keyframes flash-bull { 0% { background: var(--bull-bg); } 100% { background: transparent; } }
@keyframes flash-bear { 0% { background: var(--bear-bg); } 100% { background: transparent; } }
.cell--just-updated-up   { animation: flash-bull 600ms ease-out; }
.cell--just-updated-down { animation: flash-bear 600ms ease-out; }
```

**`live-pulse`** — точка SSE-status / freshness:
```css
@keyframes live-pulse {
  0%, 100% { box-shadow: 0 0 0 0 var(--bull); opacity: 1; }
  50%      { box-shadow: 0 0 0 4px transparent; opacity: 0.7; }
}
```

**`alert-row-enter`** — новая строка в `/alerts` через SSE:
- `opacity 0 → 1`, `translateY(-4px → 0)`, 220ms ease-out.
- Подсветка строки `background --warn-bg` 1.5s, затем fade.

**`skeleton-shimmer`** — load state (исп. только если данные первые):
- 1.6s `linear infinite`, `linear-gradient` поверх `--surface-2`.

**Что мы НЕ делаем:** parallax, scroll-driven animation, page hero, cursor trails, sparkles, confetti, stagger > 50ms, любые «delight» эффекты. Это финансовая утилита.

---

## 7. Компоненты — паттерны и спецификации

### 7.1 LiveTable (центральный компонент дашборда)

- **Виртуализация** через TanStack Table + `@tanstack/react-virtual`. `estimateSize: () => --row-height`.
- **Header sticky.** Сортировка по любой колонке. Active sort — иконка + `text-accent`.
- **Row height фиксирован.** Никогда не меняется от контента (truncate с tooltip).
- **Δ-ячейка** — компонент `DeltaCell`:
  ```tsx
  <DeltaCell value={delta_5m_pct} window="5m" />
  // рендерит знак, цвет по §2.4, tabular-nums, при update — flash через ref
  ```
- **Cell update strategy:** `useRef` хранит prev value, при изменении в `useEffect` навешивает класс `cell--just-updated-up/down`, снимает после `motion-flash`. Это избегает re-render всей строки.
- **Empty / loading state:** skeleton из 20 строк placeholder'ов.
- **No-data per row** (null Δ, ещё нет истории): `—` в `--fg-muted`, не пустой cell.

### 7.2 DeltaBar (опционально, в symbol page)

Горизонтальная мини-bar для Δ за 5m, 15m, 30m. Ширина пропорциональна |Δ|, цвет по §2.4. Дополняет числовое значение, не заменяет.

### 7.3 ExchangeChip

```tsx
<ExchangeChip exchange="binance" status="ok" />
```
- 22×height padded, `radius-sm`, `text-2xs`, `font-weight: 500`.
- Цвет — из `EXCHANGE_HUE` (§2.5).
- Status dot (точка 6px) слева: `--bull` ok, `--warn` degraded, `--bear` down. Pulse если `live`.

### 7.4 FreshnessIndicator

- Показывает `last_sample_age_sec` для пары `(exchange, symbol)`.
- Fresh (< `2 × poll_interval`): `--bull`, без декора.
- Aging (`2..3 × poll_interval`): `--warn`, текст «aging».
- Stale (`> 3 × poll_interval`): `--bear`, текст «stale», иконка ⚠️.
- В compact mode — только цветная точка + tooltip с числом.

### 7.5 ValuationBadge

| status | цвет | tooltip |
|---|---|---|
| `authoritative` | `--val-authoritative` | «Прямой OI с биржи (oi_coins)» |
| `good_estimate` | `--val-good-estimate` | «Расчёт через mark price ± 0.1%» |
| `low_confidence` | `--val-low-confidence` | «Оценка на основе stale price; не для алертов» |

Размер — 8px dot в плотном виде, `text-2xs` chip в просторном.

### 7.6 ConnectionStatusBadge

В нижней части sidebar + дублируется в header'e dashboard.

```
●  live          — SSE connected, последний event < 5s назад. Pulse animation.
●  reconnecting  — backoff (см. 10_DELIVERY_LAYER.md §5.6). text-warn.
●  offline       — нет SSE > 30s. text-bear, не pulse, иконка ⚠.
```

### 7.7 Charts (Recharts)

- Theme через `<defs>` + ResponsiveContainer.
- Линии: 12 hue из §2.5, толщина 1.5px, `dot={false}`.
- Tooltip: `--surface-3`, `--shadow-popover`, числа `.tabular`.
- Grid: `stroke=var(--border-subtle)`, dashed `4 4`, без вертикалей в density-mode.
- X-axis: `axisLine=false`, `tickLine=false`, `tick fill=var(--fg-muted)`.
- Brush для long periods (≥ 7d).
- **Crosshair sync** между чартами на одной странице.
- Legend — chips с toggle (см. `10_DELIVERY_LAYER.md §6.1` Symbol page).

### 7.8 Forms / Settings

- Input `height: 32px` (compact) / 36px (comfortable).
- Label сверху, `text-xs`, `--fg-secondary`.
- Help text под input'ом, `text-xs`, `--fg-muted`.
- Slider: rail `--surface-2`, fill `--accent`, handle 14px circle.
- Toggle: 32×18, dot 14px, smooth `transform` 140ms.
- **Errors inline:** красный border + `text-xs` `--bear` под полем. Toast только для server errors / 429.
- **Audit-log в Settings** (см. `10_DELIVERY_LAYER.md §4.8`): свернутый список последних правок, click → expand diff.

### 7.9 Toasts

- Top-right, max 3 stacked. Auto-dismiss 4s (success), 8s (error), sticky (critical).
- Цвета по семантике (`bull/bear/warn/info`).
- Иконка слева 16px. Action link справа (`Undo`, `Retry`).

### 7.10 Empty states

| Где | Что показываем |
|---|---|
| Dashboard (cold start) | большой `--fg-muted` блок: «Сбор данных. Первые точки появятся через ~1 минуту.» + skeleton. |
| Symbol page (нет истории) | «Недостаточно истории для построения графика. Минимум `min_history_minutes_floor` мин.» |
| Alerts (нет за период) | «Нет алертов за выбранный период. Попробуйте увеличить диапазон.» |

### 7.11 Modals / Drawers

- **Modal**: 480 / 640 / 800px width, по центру, backdrop `--surface-overlay`. Esc / click backdrop / X закрывают.
- **Drawer right** для symbol-detail из dashboard (alt: full page). 480px, `transform: translateX` enter.
- Focus-trap обязателен. Возврат focus при закрытии.

### 7.12 Command menu (`Cmd/Ctrl-K`)

Power-user feature. Ищет: символы, биржи, страницы, действия. Surface-3, popover-shadow, монотипографичный.

---

## 8. Per-page UX-требования

### 8.1 Dashboard `/`

**Информационная иерархия (сверху вниз):**

1. **Status strip** (32px): SSE state • last cycle ts • data freshness summary (`12 / 12 exchanges fresh`) • density toggle.
2. **Filters bar** (40px): Exchange multi-select chips • Min OI slider (compact) • Symbol search input (debounced 250ms) • Sort dropdown.
3. **Live-таблица** — занимает остаток.

**Колонки (compact mode):**

| Col | Width | Align | Notes |
|---|---|---|---|
| Exchange | 96 | left | ExchangeChip |
| Symbol | 100 | left | mono, link to `/symbol/:c` |
| OI ($) | 110 | right | tabular, humanized |
| Δ5m | 78 | right | DeltaCell, см. §2.4 |
| Δ15m | 78 | right | DeltaCell |
| Δ30m | 78 | right | DeltaCell |
| Price | 110 | right | tabular |
| Val. | 28 | center | dot |
| Fresh | 56 | right | FreshnessIndicator |

`Σ ≈ 734px`. На 1280 viewport остаётся запас на sidebar + scrollbar.

**Поведение:**
- Клик строки → row selected (`--surface-2`), enter / dblclick → переход на `/symbol/:c`.
- Right-click row → context menu: «Copy symbol», «Add to blacklist», «Open chart», «View alerts for…».
- Sort persistence в localStorage.
- При SSE event — flash updated cells (§6.3), без перерасчёта sort'а в течение 1s (батчинг).

### 8.2 Symbol page `/symbol/:canonical`

**Layout:**
```
┌── Header: BTC • $1.24B total OI • +3.2% 5m avg • [period selector] ──┐
│                                                                       │
│  ┌── Main chart (12 lines) — height 480 ─────────────────────────┐    │
│                                                                       │
│  ┌── Per-exchange table (compact) ────────────────────────────────┐   │
│  │  ExchangeChip │ OI │ Δ5m │ Δ15m │ Δ30m │ Price │ Last update    │
│                                                                       │
│  ┌── Recent alerts for this symbol (last 24h) ────────────────────┐   │
└───────────────────────────────────────────────────────────────────────┘
```

- Period chips: `1h / 6h / 24h / 7d / 30d / 90d` — segmented control, не dropdown.
- Y-axis ticker с humanized USD.
- Hover crosshair → tooltip со значениями всех 12 линий, синхронизирован между chart и таблицей (highlight row).
- Toggle линий → click ExchangeChip в таблице.

### 8.3 Alerts log `/alerts`

- Таблица + sticky filters bar.
- **Live mode toggle** (по умолчанию on): SSE подписка, новые ряды animate-enter (§6.3).
- Колонки: `time | exchange | symbol | window | rule | Δ | decision | delivery`.
- Decision badge: `fire` (`--bear/--bull`-rim), `resolve` (`--neutral`), `cooldown_skip` (`--fg-muted`).
- Delivery status: `sent` (`--bull`), `pending` (`--warn`, pulse), `failed` (`--bear` + tooltip с причиной).
- Click row → side drawer с full alert payload (JSON viewer collapsible).

### 8.4 Settings `/settings`

- 5 tabs (см. `10_DELIVERY_LAYER.md §6.1`). Tabs sticky под header.
- Каждое поле: label + control + help-text + «default: X» link (восстановить дефолт).
- **Save behaviour:** автосохранение через `useDebounce(300ms)` для slider/numeric, immediate для toggle. Toast `saved` или error.
- **Diff peek:** при наведении на поле, изменённое в текущей сессии — подсказка с предыдущим значением.
- Tab «History» — read-only audit log (§7.8).

---

## 9. Доступность (a11y)

- Контраст: text ≥ 4.5:1 на surface-base/1/2 — проверяется автоматически в Storybook addon.
- Цвет — никогда не единственный носитель информации (Δ всегда имеет знак `+/−`, freshness — иконку, valuation — текстовый tooltip).
- Все interactive — keyboard reachable, `:focus-visible` ring (`--glow-focus`).
- ARIA roles: table `role="grid"`, charts получают `aria-label` с summary (max/min/last).
- `prefers-reduced-motion` — обязательно поддерживается (§6.1).
- Темы (dark/light) и density сохраняются в localStorage **и** settings (для cross-device consistency single-user'а).
- `lang="ru"` или `lang="en"` на html в зависимости от выбора.

---

## 10. Performance budgets (UI side)

| Метрика | Бюджет | Источник |
|---|---|---|
| TTI на dashboard | ≤ 1s @ 3000 rows | `01_PRODUCT_SPEC.md §7` |
| Bundle size (gz) | ≤ 250 KB initial | webpack-bundle-analyzer |
| Re-render on SSE event | ≤ 16ms (60fps) | React Profiler |
| Cell flash latency | ≤ 50ms от события до DOM | manual measure |
| SSE redraw throttle | 1× / 200ms | `10_DELIVERY_LAYER.md §6.5` |
| Chart 1k×12 — pan/zoom | ≤ 16ms frame | dev measure |

Реализация:
- Code splitting: Settings и Symbol page — `React.lazy`.
- `React.memo` для `DeltaCell`, `ExchangeChip`, `Row` — обязательно.
- TanStack virtualization, `overscan: 10`.
- Recharts: `isAnimationActive={false}` для real-time графиков.

---

## 11. Иконография

- **Lucide** (open-source, tree-shake-friendly) — единственный icon set.
- Размеры: 14 / 16 / 20 / 24px. `stroke-width: 1.5` (стандарт под текст в density-режиме).
- Никаких эмодзи в UI (TG-сообщения — да, см. `10_DELIVERY_LAYER.md §3`, но не в web). Исключение: ⚠ в alerts (заменяем на lucide `triangle-alert`).

---

## 12. Tokens — implementation

### 12.1 Структура

```
frontend/src/styles/
├── tokens/
│   ├── colors.css       # все --color-* в OKLCH
│   ├── typography.css   # font-faces, шкала
│   ├── spacing.css      # --space-*, --row-height
│   ├── motion.css       # --motion-*
│   └── elevation.css    # shadows, radii
├── themes/
│   ├── dark.css         # data-theme="dark"
│   └── light.css        # data-theme="light"
└── globals.css          # imports, reset, base
```

### 12.2 Tailwind config (extract)

```ts
// tailwind.config.ts
export default {
  theme: {
    extend: {
      colors: {
        // proxy в CSS variables
        surface: { base: 'var(--surface-base)', 1: 'var(--surface-1)', 2: 'var(--surface-2)', 3: 'var(--surface-3)' },
        fg: { primary: 'var(--fg-primary)', secondary: 'var(--fg-secondary)', muted: 'var(--fg-muted)' },
        bull: 'var(--bull)', bear: 'var(--bear)', warn: 'var(--warn)', info: 'var(--info)',
      },
      fontFamily: {
        sans: ['Inter', 'system-ui', 'sans-serif'],
        mono: ['JetBrains Mono', 'ui-monospace', 'monospace'],
      },
      fontSize: { '2xs': ['10px', '14px'], xs: ['11px', '16px'], sm: ['12px', '18px'] /* ... */ },
      transitionTimingFunction: { brand: 'cubic-bezier(.2,.8,.2,1)' },
      transitionDuration: { instant: '80ms', fast: '140ms', base: '220ms', slow: '360ms' },
      borderRadius: { sm: '4px', md: '6px', lg: '10px' },
      boxShadow: { popover: 'var(--shadow-popover)', modal: 'var(--shadow-modal)', toast: 'var(--shadow-toast)' },
    },
  },
  plugins: [require('@tailwindcss/forms'), require('tailwindcss-animate')],
}
```

### 12.3 Storybook

- Каждый компонент → story с состояниями (default / hover / loading / empty / error).
- Tokens browsable как Design Tokens addon.
- Visual regression через Chromatic или `@storybook/test-runner` + Playwright (опционально, M4+).

---

## 13. UX-требования к качеству (acceptance for M4)

В дополнение к DoD из roadmap:

- [ ] **No layout shift** на SSE update — измерено `web-vitals` CLS = 0.
- [ ] **Tabular alignment** — числовые колонки выровнены по разрядам во всех состояниях, включая `null`-значения.
- [ ] **Connection status** видим из любой страницы.
- [ ] **Freshness** видим для каждой пары на dashboard.
- [ ] **Density toggle** работает, сохраняется в localStorage + settings.
- [ ] **Theme toggle** работает; light theme проходит контраст-аудит.
- [ ] **Reduced motion** уважается во всех анимациях.
- [ ] **Keyboard nav:** dashboard полностью пройден без мыши (sort, filter, row select, navigate to symbol page).
- [ ] **Empty / loading / error** state у каждой страницы реализован, не пустой `<div>`.
- [ ] **Bundle ≤ 250 KB initial gz**.

---

## 14. Что мы сознательно НЕ делаем

- Никакого «design language» с brand mascot, illustrations, gradient hero. Это утилита.
- Никакого dark→light auto switch по времени суток. Single-user сам решает.
- Никаких toast-celebration на «saved» (тонкий fade — да, anim flourish — нет).
- Никаких animated charts on initial render для real-time данных (отвлекает).
- Никакого custom scrollbar styling за пределами цвета thumb (нативный UX > эстетика).
- Никаких карточек с borders + shadow одновременно (выберите одно — у нас border).
- Никакого blur/glassmorphism (читабельность чисел > вау-эффект).

---

## 15. Senior-рекомендации (открытые точки)

Три места, где принятые здесь дефолты — сознательный выбор, а не «единственно правильный»:

1. **Density default = `compact`** (12px tabular) — нестандартно для веба, но соответствует P1 (density-first для опытного трейдера). Альтернатива — `comfortable` + onboarding-tip про переключение. Зафиксировано: `compact`.
2. **OKLCH без HSL fallback** — требует Chrome 111+ / Safari 16.4+ / Firefox 113+. Для single-user'а на personal-use Chrome — норм. Зафиксировано: OKLCH only.
3. **Self-host fonts** — +работа, +bundle, но ZERO внешних коннектов (соответствует `noindex` + private VPN сетап `F12/F13`). Зафиксировано: self-host.

При появлении новых сигналов (например, второй пользователь) — пересмотреть в `00_DECISIONS_LOG.md` через `Fxx`.

---

## 16. Versioning

Этот документ — `1.0`.

Изменения design tokens (цвета, шкалы, motion) — patch version.
Изменения принципов §1 или per-page UX §8 — minor version.
Полная переработка темы / density-модели / типографики — major version.

Любое отклонение реализации от этого документа — фиксируется как `Fxx` в `00_DECISIONS_LOG.md` (по правилу `CLAUDE.md §1.2`).
