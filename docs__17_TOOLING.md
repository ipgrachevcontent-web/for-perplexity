# 17 · Tooling

> Dev-инструменты, package management, pre-commit hooks, CI minimal. Скетчи `pyproject.toml` / `package.json` / `.pre-commit-config.yaml` — чтобы не выдумывать на ходу при старте M1.
> Связано: `02_ARCHITECTURE.md §4`, `14_TEST_STRATEGY.md`, `13_OPERATIONS.md §7` (deploy workflow).

---

## 1. Backend: Python 3.12

### 1.1 Package manager

**`uv`** (Astral) — выбран как primary.

**Почему:**
- ~10–100× быстрее `pip` на холодных установках.
- Lock-файл (`uv.lock`) детерминирован.
- Один бинарь, без bootstrap-овой Python-окружения.
- Совместим с `pyproject.toml` PEP 621.

**Fallback:** `pip + venv` совместимы с тем же `pyproject.toml` — если по любой причине `uv` недоступен, проект запустится через `python -m venv .venv && .venv/bin/pip install -e .`.

### 1.2 `pyproject.toml` skeleton

```toml
[project]
name = "oi-tracker"
version = "0.1.0"
description = "Open Interest tracker for 12 USDT-M perpetual exchanges"
readme = "README.md"
requires-python = ">=3.12,<3.13"
license = { text = "Proprietary" }

dependencies = [
    # HTTP
    "httpx[http2]>=0.27,<0.28",

    # Web framework
    "fastapi>=0.115,<0.116",
    "uvicorn[standard]>=0.30,<0.31",
    "sse-starlette>=2.1,<3.0",

    # Validation
    "pydantic>=2.7,<3.0",
    "pydantic-settings>=2.4,<3.0",

    # Database
    "sqlalchemy[asyncio]>=2.0,<3.0",
    "asyncpg>=0.29,<0.30",
    "alembic>=1.13,<2.0",

    # Telegram
    "aiogram>=3.10,<4.0",

    # Logging
    "structlog>=24.4,<25.0",
    "orjson>=3.10,<4.0",

    # Metrics
    "prometheus-client>=0.20,<1.0",

    # Time / dates
    "python-dateutil>=2.9,<3.0",

    # Decimal precision helpers
    # (Decimal в stdlib; добавь ниже если нужны хелперы)
]

[project.optional-dependencies]
dev = [
    # Test
    "pytest>=8.3,<9.0",
    "pytest-asyncio>=0.23,<1.0",
    "pytest-cov>=5.0,<6.0",
    "pytest-mock>=3.14,<4.0",
    "freezegun>=1.5,<2.0",
    "hypothesis>=6.111,<7.0",
    "respx>=0.21,<1.0",            # httpx mocking
    "asgi-lifespan>=2.1,<3.0",     # FastAPI lifespan in tests

    # Quality
    "ruff>=0.6,<1.0",
    "mypy>=1.11,<2.0",
    "types-python-dateutil",

    # Pre-commit
    "pre-commit>=3.8,<4.0",
]

[build-system]
requires = ["hatchling>=1.25"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/oi_tracker"]

# ---------------------------------------------------------------------------
# Tooling configuration
# ---------------------------------------------------------------------------

[tool.ruff]
line-length = 100
target-version = "py312"
src = ["src", "tests"]

[tool.ruff.lint]
select = [
    "E", "F", "W",       # pycodestyle / pyflakes
    "I",                 # isort
    "N",                 # pep8-naming
    "UP",                # pyupgrade
    "B",                 # bugbear
    "C4",                # comprehensions
    "SIM",               # simplify
    "TCH",               # type-checking imports
    "PIE", "PT", "RET",
    "RUF",
    "ASYNC",             # async best practices
    "S",                 # bandit (security)
]
ignore = [
    "S101",              # assert in tests is fine
    "S104",              # 0.0.0.0 — мы вообще на UDS, but justify per-case
]

[tool.ruff.lint.per-file-ignores]
"tests/**" = ["S", "PT011"]
"alembic/versions/**" = ["E501"]

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
line-ending = "lf"

[tool.mypy]
python_version = "3.12"
strict = true
plugins = ["pydantic.mypy"]
mypy_path = "src"
namespace_packages = true
explicit_package_bases = true
disallow_untyped_decorators = true
warn_unreachable = true
warn_redundant_casts = true
show_error_codes = true

[[tool.mypy.overrides]]
module = ["alembic.*"]
ignore_missing_imports = true

[tool.pytest.ini_options]
minversion = "8.0"
testpaths = ["tests"]
asyncio_mode = "auto"
addopts = [
    "--strict-markers",
    "--strict-config",
    "-ra",
]
markers = [
    "contract: contract tests (real fixture replay)",
    "integration: integration tests (real DB)",
    "slow: slow tests excluded from default run",
]
filterwarnings = [
    "error",
    "ignore::DeprecationWarning:dateutil.*",
]

[tool.coverage.run]
source = ["src/oi_tracker"]
branch = true
parallel = true

[tool.coverage.report]
fail_under = 80
exclude_lines = [
    "pragma: no cover",
    "if TYPE_CHECKING:",
    "raise NotImplementedError",
    "if __name__ == .__main__.:",
]
```

**Версии зафиксированы как ranges (`>=X,<Y`):** не pin-to-exact-version в `pyproject.toml` — это работа `uv.lock`. В `pyproject.toml` фиксируем major-границы.

### 1.3 Структура директорий

```
backend/
├── pyproject.toml
├── uv.lock
├── README.md
├── alembic.ini
├── alembic/
│   ├── env.py
│   ├── script.py.mako
│   └── versions/
├── src/
│   └── oi_tracker/
│       ├── __init__.py
│       ├── config.py            # pydantic-settings
│       ├── db/                  # SQLAlchemy models, repos
│       ├── exchanges/           # 12 connectors + base
│       │   ├── _base.py
│       │   ├── binance.py
│       │   ├── bybit.py
│       │   └── ...
│       ├── normalizer/
│       ├── alert_engine/
│       ├── tg_sender/
│       ├── api/                 # FastAPI app
│       │   ├── __init__.py
│       │   ├── main.py
│       │   ├── routers/
│       │   └── sse/
│       ├── scheduler/
│       ├── observability/       # metrics, logging
│       └── tools/               # CLI: replay, backfill, seed
├── tests/
│   ├── conftest.py
│   ├── unit/
│   ├── integration/
│   ├── contract/
│   └── fixtures/
│       └── exchanges/<exchange>/*.json
└── log_config.json
```

См. также `02_ARCHITECTURE.md §4` (он остаётся главным источником по структуре, этот файл его не переопределяет).

### 1.4 Основные команды

```bash
# Создать venv + установить
uv sync --extra dev

# Запустить всё локально (требует PG)
uv run alembic upgrade head
uv run python -m oi_tracker.scheduler_main &
uv run uvicorn oi_tracker.api.main:app --uds /tmp/oi-tracker.sock

# Тесты
uv run pytest                              # all
uv run pytest -m "not slow"                # default
uv run pytest --cov                        # с coverage
uv run pytest tests/contract/binance       # только Binance contract

# Линт + типы
uv run ruff check src tests
uv run ruff format --check src tests
uv run mypy src tests

# Алиас всё-сразу
uv run pre-commit run --all-files
```

---

## 2. Frontend: React + Vite + TypeScript

### 2.1 Package manager

**`pnpm`** — primary. Быстрее npm, дисковая дедупликация, детерминированный lock (`pnpm-lock.yaml`).

**Fallback:** `npm` совместим с тем же `package.json`.

### 2.2 `package.json` skeleton

```jsonc
{
  "name": "oi-tracker-frontend",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview",
    "lint": "eslint . --max-warnings 0",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "test:ui": "vitest"
  },
  "dependencies": {
    "react": "^18.3.0",
    "react-dom": "^18.3.0",
    "react-router-dom": "^6.26.0",
    "@tanstack/react-table": "^8.20.0",
    "@tanstack/react-virtual": "^3.10.0",
    "recharts": "^2.12.0",
    "decimal.js": "^10.4.0"
  },
  "devDependencies": {
    "@types/react": "^18.3.0",
    "@types/react-dom": "^18.3.0",
    "@vitejs/plugin-react": "^4.3.0",
    "vite": "^5.4.0",
    "typescript": "^5.5.0",
    "eslint": "^9.10.0",
    "@typescript-eslint/eslint-plugin": "^8.4.0",
    "@typescript-eslint/parser": "^8.4.0",
    "eslint-plugin-react": "^7.35.0",
    "eslint-plugin-react-hooks": "^4.6.0",
    "prettier": "^3.3.0",
    "vitest": "^2.0.0",
    "@testing-library/react": "^16.0.0",
    "@testing-library/jest-dom": "^6.5.0",
    "jsdom": "^25.0.0",
    "tailwindcss": "^3.4.0",
    "autoprefixer": "^10.4.0",
    "postcss": "^8.4.0"
  }
}
```

### 2.3 TS strict

`tsconfig.json` минимум:

```jsonc
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "jsx": "react-jsx",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "exactOptionalPropertyTypes": true,
    "noFallthroughCasesInSwitch": true,
    "skipLibCheck": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true,
    "resolveJsonModule": true,
    "esModuleInterop": true
  },
  "include": ["src", "vite.config.ts"]
}
```

### 2.4 Структура директорий

См. `10_DELIVERY_LAYER.md §6.2` — это главный источник, дублировать не буду.

### 2.5 Build target

```bash
pnpm install
pnpm build
# dist/ скопировать в /var/www/oi-tracker/frontend-dist
```

`vite.config.ts` должен содержать:
```ts
export default defineConfig({
  plugins: [react()],
  base: "/",
  build: {
    outDir: "dist",
    emptyOutDir: true,
    sourcemap: true,
  },
  server: {
    proxy: {
      "/api":  { target: "http://127.0.0.1:8010", changeOrigin: true },
      "/sse":  { target: "http://127.0.0.1:8010", changeOrigin: true, ws: false },
    },
  },
});
```

(При локальной разработке uvicorn запускается на TCP `127.0.0.1:8010` — fallback-порт по `00_DECISIONS_LOG F14`. Порт 8000 занят соседом `detector_api` на shared host. UDS используется только в production через nginx.)

---

## 3. Pre-commit hooks

`.pre-commit-config.yaml` в корне репозитория:

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: check-toml
      - id: check-added-large-files
        args: [--maxkb=500]
      - id: check-merge-conflict
      - id: detect-private-key
      - id: mixed-line-ending
        args: [--fix=lf]

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.6.4
    hooks:
      - id: ruff
        args: [--fix, --exit-non-zero-on-fix]
        files: ^backend/
      - id: ruff-format
        files: ^backend/

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.11.2
    hooks:
      - id: mypy
        files: ^backend/src/
        additional_dependencies:
          - pydantic>=2.7
          - sqlalchemy>=2.0

  - repo: local
    hooks:
      - id: pytest-quick
        name: pytest (quick subset)
        entry: bash -lc 'cd backend && uv run pytest -x -q -m "not slow and not integration and not contract"'
        language: system
        pass_filenames: false
        files: ^backend/(src|tests)/

      - id: frontend-typecheck
        name: tsc --noEmit
        entry: bash -lc 'cd frontend && pnpm typecheck'
        language: system
        pass_filenames: false
        files: ^frontend/(src|.*\.tsx?$)

      - id: frontend-lint
        name: eslint
        entry: bash -lc 'cd frontend && pnpm lint'
        language: system
        pass_filenames: false
        files: ^frontend/(src|.*\.tsx?$)

      - id: gitleaks
        name: gitleaks (secrets)
        entry: gitleaks detect --no-banner --redact --source .
        language: system
        pass_filenames: false
```

### 3.1 Установка

```bash
pre-commit install                        # один раз в repo
pre-commit install --hook-type commit-msg # для conventional commits
pre-commit run --all-files                # вручную
```

### 3.2 Conventional commits hook

`commitlint` или `gitlint` опционально. Базово CLAUDE.md §5 уже регламентирует формат, но для machine-проверки можно:

```yaml
  - repo: https://github.com/jorisroovers/gitlint
    rev: v0.19.1
    hooks:
      - id: gitlint
        stages: [commit-msg]
```

`.gitlint`:
```ini
[general]
contrib=contrib-title-conventional-commits
[contrib-title-conventional-commits]
types=feat,fix,refactor,docs,test,chore,perf,ci
```

---

## 4. CI minimal (GitHub Actions)

Single-user проект, но CI всё равно ловит дребезг от human-mistakes. Минимальный набор jobs:

```yaml
# .github/workflows/ci.yml
name: ci
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  backend:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: timescale/timescaledb:latest-pg16
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: oi_tracker_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 10
        ports: ['5432:5432']
    env:
      DATABASE_URL: postgresql+asyncpg://postgres:test@localhost:5432/oi_tracker_test
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
        with: { version: 'latest' }
      - run: cd backend && uv sync --extra dev
      - run: cd backend && uv run ruff check .
      - run: cd backend && uv run ruff format --check .
      - run: cd backend && uv run mypy src tests
      - run: cd backend && uv run alembic upgrade head
      - run: cd backend && uv run pytest --cov --cov-fail-under=80 -m "not slow"

  frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: 'pnpm', cache-dependency-path: frontend/pnpm-lock.yaml }
      - run: cd frontend && pnpm install --frozen-lockfile
      - run: cd frontend && pnpm typecheck
      - run: cd frontend && pnpm lint
      - run: cd frontend && pnpm test
      - run: cd frontend && pnpm build

  secrets:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: gitleaks/gitleaks-action@v2
```

CI **не** деплоит — деплой ручной (см. `13_OPERATIONS.md §7.1`). CI только validates.

### 4.1 Что CI НЕ делает

- Не пускает интеграционные/contract-тесты на реальные биржи (rate limits, flaky).
- Не пушит cover отчёты во внешние сервисы (single-user).
- Не запускает E2E браузерные тесты (M5 future work).

---

## 5. Editor / IDE

Не часть infra, но рекомендации для согласованности:

### 5.1 VS Code (`.vscode/settings.json`)

```jsonc
{
  "python.defaultInterpreterPath": "backend/.venv/bin/python",
  "python.analysis.typeCheckingMode": "strict",
  "[python]": {
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": { "source.fixAll": "explicit" },
    "editor.defaultFormatter": "charliermarsh.ruff"
  },
  "[typescript]": {
    "editor.formatOnSave": true,
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[typescriptreact]": { "editor.formatOnSave": true, "editor.defaultFormatter": "esbenp.prettier-vscode" },
  "files.eol": "\n",
  "files.insertFinalNewline": true,
  "files.trimTrailingWhitespace": true
}
```

`.vscode/extensions.json`:
```jsonc
{
  "recommendations": [
    "charliermarsh.ruff",
    "ms-python.python",
    "ms-python.mypy-type-checker",
    "esbenp.prettier-vscode",
    "dbaeumer.vscode-eslint",
    "tamasfe.even-better-toml",
    "redhat.vscode-yaml"
  ]
}
```

### 5.2 EditorConfig

`.editorconfig` (репо-уровень):

```ini
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true
indent_style = space
indent_size = 4

[*.{ts,tsx,js,jsx,json,jsonc,yml,yaml}]
indent_size = 2

[Makefile]
indent_style = tab
```

---

## 6. Cross-references

- Структура проекта (детали) → `02_ARCHITECTURE.md §4`.
- Test-стратегия (что и где тестируем) → `14_TEST_STRATEGY.md`.
- Deploy commands (production) → `13_OPERATIONS.md §2, §7`.
- Frontend компоненты → `10_DELIVERY_LAYER.md §6`.
- Decisions, на которые опирается выбор инструментов → `00_DECISIONS_LOG.md` (например `Q6` — без Docker).
