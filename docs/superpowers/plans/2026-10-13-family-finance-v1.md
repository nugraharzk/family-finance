# Family Finance v1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a self-hosted household finance app (FastAPI + SQLite + Nuxt 3) that records expenses in seconds, imports bank CSV files without double-counting, and is pleasant to look at.

**Architecture:** Modular monolith — one FastAPI container serving both `/api/*` and a statically built Nuxt SPA, backed by a single SQLite file. Six backend modules (`auth`, `accounts`, `categories`, `transactions`, `imports`, `reports`), each strictly `router → service → models/schemas`, with cross-module calls allowed only through `service` functions.

**Tech Stack:** Python 3.12, FastAPI, SQLAlchemy 2.0, Alembic, Pydantic v2, argon2-cffi, charset-normalizer, pytest, hypothesis · Nuxt 3 (Vue 3), Tailwind CSS v4 via `@nuxtjs/tailwindcss` · SQLite (WAL) · Docker

**Spec:** `docs/superpowers/specs/2026-10-13-family-finance-v1-design.md` — read it alongside this plan; the plan argues from the spec.

## Global Constraints

Copied verbatim from the spec. Every task's requirements implicitly include these.

- **Money is a signed `INTEGER` in rupiah. Floats are forbidden anywhere in the codebase.** Outflow is negative, inflow positive.
- **Single currency: IDR.** No FX, no per-currency accounts.
- **SQLite settings, mandatory on every connection:** `PRAGMA foreign_keys=ON`, `PRAGMA journal_mode=WAL`, `PRAGMA busy_timeout=5000`. `foreign_keys` is OFF by default — without it FK constraints are silently ignored.
- **Single error envelope:** `{"error": {"code": str, "message": str, "details": dict|None}}`. Never leak stack traces or SQL to the client.
- **Module contract:** `router.py` (HTTP only, no logic) → `service.py` (all logic, the only cross-module entry point) → `models.py` (never imported by another module) → `schemas.py`.
- **No role gating.** Both members are full-access adults. The `members.role` column exists, always `'adult'`; build no kid logic.
- **UI copy is in Bahasa Indonesia.** Code identifiers and commit messages stay in English.
- **No component kit.** Tailwind plus the project's own `@theme` tokens and three `@utility` primitives.
- **Money never crosses the wire as a float** — JSON integers only.
- **Import limits:** 20 MB file cap, 50 000 row cap, reject binary/PDF with a clear message.
- **Session cookie:** httpOnly, `SameSite=Lax`, 30-day sliding expiry.
- **Alembic auto-runs on boot**; `/api/health` exists for the compose healthcheck.

## Review Focus

Five input classes the spec implies but no task's own tests would naturally exercise. Each has its test pinned to the owning task below.

1. **Non-UTF-8 / BOM encodings** — a real bank CSV in `cp1252` or with a UTF-8 BOM must import with correct characters, not mojibake. → Task 24.
2. **Hostile amount strings** — `(1.234,56)`, `Rp 1.234.567,00`, `1.000-`, `45.000 DR` must yield the correct signed integer, and a genuinely unparseable value must produce a row-level error, never an unhandled exception. → Task 26.
3. **Internal transfers must not change net worth** — a transfer between two accounts nets to zero across the household. → Tasks 20 and 32.
4. **Degenerate files** — header-only CSV, wrong-delimiter sniff, or a 0-byte file must fail with a clear message and import nothing, never partially import. → Task 24.
5. **Two genuinely distinct transactions sharing `(date, amount, description)`** — both must import; they must not collapse into one, and re-importing the same file must still be a no-op. → Task 28.

---

## File Structure

```
family-finance/
├── Dockerfile
├── docker-compose.yml
├── .gitignore                       # already exists
├── backend/
│   ├── pyproject.toml
│   ├── alembic.ini
│   ├── app/
│   │   ├── main.py                  # app factory, static mount, error handlers
│   │   ├── core/
│   │   │   ├── config.py            # pydantic-settings
│   │   │   ├── db.py                # engine factory, PRAGMAs, get_db
│   │   │   ├── money.py             # parse_rupiah / format_rupiah
│   │   │   ├── security.py          # argon2, session tokens
│   │   │   ├── errors.py            # domain exceptions + handlers
│   │   │   └── deps.py              # current_member dependency
│   │   ├── modules/
│   │   │   ├── auth/       {models,service,schemas,router}.py
│   │   │   ├── accounts/   {models,service,schemas,router}.py
│   │   │   ├── categories/ {models,service,schemas,router}.py
│   │   │   ├── transactions/{models,service,schemas,router}.py
│   │   │   ├── imports/    {models,service,schemas,router}.py
│   │   │   └── reports/    {service,schemas,router}.py
│   │   └── migrations/{env.py,versions/}
│   └── tests/
│       ├── conftest.py
│       ├── core/test_money.py
│       └── modules/{auth,accounts,categories,transactions,imports,reports}/test_*.py
├── frontend/
│   ├── nuxt.config.ts
│   ├── package.json
│   ├── assets/css/main.css          # @theme tokens + @utility primitives
│   ├── app.vue
│   ├── layouts/default.vue
│   ├── composables/{useApi.ts,useSession.ts}
│   ├── components/{AppShell,MoneyText,CategoryChip,ProgressBar,TransactionRow,TransactionModal,MemberAvatar}.vue
│   └── pages/
│       ├── setup.vue  login.vue  index.vue  categories.vue  rules.vue  settings.vue  reports.vue
│       ├── accounts/index.vue  accounts/[id].vue
│       ├── transactions/index.vue
│       └── import/index.vue
└── docs/superpowers/{specs,plans}/
```

**Responsibility boundaries:** `core/` holds things every module needs and nothing owns (config, DB, money, auth primitives, errors). A module owns exactly one domain concept. `reports/` has no `models.py` — it reads through other modules' services and never defines tables.

**Ordering rationale:** Phase 1–2 produce a login you can reach. 3–4 make it a working ledger. **Phase 5 (import) is the highest-risk subsystem and must not start until the ledger is trustworthy** — it writes bulk data and its bugs are silent (double-counting). 6–7 make it pleasant.

---

# Phase 1 — Skeleton

### Task 1: Backend package + tooling

**Files:**
- Create: `backend/pyproject.toml`
- Create: `backend/app/__init__.py`, `backend/app/core/__init__.py`, `backend/app/modules/__init__.py`
- Create: `backend/tests/__init__.py`, `backend/tests/conftest.py`
- Create: `backend/.env.example`

**Interfaces:**
- Consumes: nothing.
- Produces: an installable package `app` with `pytest` runnable from `backend/`.

- [ ] **Step 1: Write `backend/pyproject.toml`**

```toml
[project]
name = "family-finance"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115",
    "uvicorn[standard]>=0.32",
    "sqlalchemy>=2.0.36",
    "alembic>=1.14",
    "pydantic>=2.9",
    "pydantic-settings>=2.6",
    "argon2-cffi>=23.1",
    "charset-normalizer>=3.4",
    "python-multipart>=0.0.12",
]

[project.optional-dependencies]
dev = ["pytest>=8.3", "httpx>=0.27", "hypothesis>=6.115"]

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-q"

[build-system]
requires = ["setuptools>=68"]
build-backend = "setuptools.build_meta"

[tool.setuptools.packages.find]
include = ["app*"]
```

- [ ] **Step 2: Create the package files**

Create empty `__init__.py` in `app/`, `app/core/`, `app/modules/`, `tests/`.

`backend/.env.example`:

```dotenv
FF_DB_PATH=./data/app.db
FF_HOUSEHOLD_NAME=Keluarga Kita
FF_SESSION_TTL_DAYS=30
FF_MAX_UPLOAD_BYTES=20971520
FF_MAX_IMPORT_ROWS=50000
```

- [ ] **Step 3: Create `tests/conftest.py` with the shared fixtures**

```python
import pytest
from fastapi.testclient import TestClient

from app.core.config import Settings
from app.core.db import Database
from app.main import create_app


@pytest.fixture
def db(tmp_path):
    return Database(tmp_path / "test.db")


@pytest.fixture
def client(db):
    app = create_app(settings=Settings(db_path=db.path, household_name="Test"))
    app.state.db = db
    with TestClient(app) as c:
        yield c
```

- [ ] **Step 4: Verify the toolchain**

Run: `cd backend && python -m venv .venv && .venv/bin/pip install -e ".[dev]" && .venv/bin/python -c "import fastapi, sqlalchemy, argon2, charset_normalizer; print('ok')"`
Expected: prints `ok`

- [ ] **Step 5: Commit**

```bash
cd ~/Projects/family-finance
git add backend/pyproject.toml backend/app backend/tests backend/.env.example
git commit -m "chore: scaffold backend package and test tooling"
```

---

### Task 2: Config, database, and health endpoint

**Files:**
- Create: `backend/app/core/config.py`
- Create: `backend/app/core/db.py`
- Create: `backend/app/main.py`
- Test: `backend/tests/core/test_db.py`

**Interfaces:**
- Consumes: `app` package from Task 1.
- Produces:
  - `Settings(db_path: Path, household_name: str, session_ttl_days: int = 30, max_upload_bytes: int = 20971520, max_import_rows: int = 50000)`
  - `Database(path: Path)` with `.path`, `.engine`, `.session()` (context manager), `.create_all()`
  - `create_app(settings: Settings) -> FastAPI`

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/core/test_db.py
from sqlalchemy import text


def test_foreign_keys_are_enabled(db):
    with db.session() as s:
        assert s.execute(text("PRAGMA foreign_keys")).scalar() == 1


def test_journal_mode_is_wal(db):
    with db.session() as s:
        assert s.execute(text("PRAGMA journal_mode")).scalar().lower() == "wal"


def test_health_endpoint(client):
    r = client.get("/api/health")
    assert r.status_code == 200
    assert r.json() == {"status": "ok"}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && .venv/bin/pytest tests/core/test_db.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'app.core.config'`

- [ ] **Step 3: Write `core/config.py`**

```python
from pathlib import Path

from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="FF_", env_file=".env", extra="ignore")

    db_path: Path = Path("./data/app.db")
    household_name: str = "Keluarga Kita"
    session_ttl_days: int = 30
    max_upload_bytes: int = 20 * 1024 * 1024
    max_import_rows: int = 50_000
    cookie_secure: bool = False  # LAN, no TLS in v1
```

- [ ] **Step 4: Write `core/db.py`**

Every connection gets the PRAGMAs — including the FK one, which is off by default.

```python
from contextlib import contextmanager
from pathlib import Path

from sqlalchemy import create_engine, event
from sqlalchemy.orm import DeclarativeBase, sessionmaker


class Base(DeclarativeBase):
    pass


class Database:
    def __init__(self, path: Path):
        self.path = Path(path)
        self.path.parent.mkdir(parents=True, exist_ok=True)
        self.engine = create_engine(f"sqlite:///{self.path}", future=True)

        @event.listens_for(self.engine, "connect")
        def _pragmas(dbapi_conn, _record):
            cur = dbapi_conn.cursor()
            cur.execute("PRAGMA foreign_keys=ON")
            cur.execute("PRAGMA journal_mode=WAL")
            cur.execute("PRAGMA busy_timeout=5000")
            cur.close()

        self._sessionmaker = sessionmaker(bind=self.engine, expire_on_commit=False, future=True)

    @contextmanager
    def session(self):
        s = self._sessionmaker()
        try:
            yield s
            s.commit()
        except Exception:
            s.rollback()
            raise
        finally:
            s.close()

    def create_all(self):
        from app.modules import accounts, auth, categories, imports, transactions  # noqa: F401
        Base.metadata.create_all(self.engine)
```

- [ ] **Step 5: Write `main.py`**

```python
from pathlib import Path

from fastapi import FastAPI

from app.core.config import Settings
from app.core.db import Database


def create_app(settings: Settings | None = None) -> FastAPI:
    settings = settings or Settings()
    app = FastAPI(title="Family Finance", docs_url="/api/docs", openapi_url="/api/openapi.json")
    app.state.settings = settings
    app.state.db = Database(settings.db_path)
    app.state.db.create_all()

    @app.get("/api/health")
    def health():
        return {"status": "ok"}

    static_dir = Path(__file__).parent / "static"
    if static_dir.exists():
        from fastapi.staticfiles import StaticFiles

        app.mount("/", StaticFiles(directory=static_dir, html=True), name="spa")

    return app


app = create_app()
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `cd backend && .venv/bin/pytest tests/core/test_db.py -v`
Expected: 3 passed

- [ ] **Step 7: Commit**

```bash
git add backend/app/core/config.py backend/app/core/db.py backend/app/main.py backend/tests/core/test_db.py
git commit -m "feat: config, sqlite database with required pragmas, health endpoint"
```

---

### Task 3: Alembic migrations

**Files:**
- Create: `backend/alembic.ini`
- Create: `backend/app/migrations/env.py`, `backend/app/migrations/script.py.mako`
- Modify: `backend/app/main.py` (run migrations on boot)

**Interfaces:**
- Consumes: `Base`, `Settings` from Task 2.
- Produces: `run_migrations(db_path: Path) -> None`, called from `create_app`.

- [ ] **Step 1: Initialise Alembic and verify the scaffold exists**

Run: `cd backend && .venv/bin/python -m alembic init -t generic app/migrations`
Expected: creates `alembic.ini`, `app/migrations/env.py`, `app/migrations/script.py.mako`, `app/migrations/versions/`

- [ ] **Step 2: Point `alembic.ini` at nothing hard-coded**

In `alembic.ini`, delete the `sqlalchemy.url` line entirely — `env.py` supplies it.

- [ ] **Step 3: Replace the body of `app/migrations/env.py`**

```python
from pathlib import Path
import os

from alembic import context
from sqlalchemy import engine_from_config, pool

from app.core.db import Base, Database
from app.modules import accounts, auth, categories, imports, transactions  # noqa: F401

config = context.config
target_metadata = Base.metadata


def _url() -> str:
    db_path = Path(os.environ.get("FF_DB_PATH", "./data/app.db"))
    db_path.parent.mkdir(parents=True, exist_ok=True)
    return f"sqlite:///{db_path}"


def run_migrations_offline() -> None:
    context.configure(url=_url(), target_metadata=target_metadata, literal_binds=True)
    with context.begin_transaction():
        context.run_migrations()


def run_migrations_online() -> None:
    db = Database(Path(os.environ.get("FF_DB_PATH", "./data/app.db")))
    with db.engine.connect() as connection:
        context.configure(connection=connection, target_metadata=target_metadata,
                          render_as_batch=True)
        with context.begin_transaction():
            context.run_migrations()


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

- [ ] **Step 4: Generate the baseline revision**

Run: `cd backend && .venv/bin/python -m alembic revision --autogenerate -m "baseline"`
Expected: `Generating .../versions/<hash>_baseline.py ... done`. It may be empty at this point (no tables yet) — that is correct; it becomes the baseline the later phases extend.

- [ ] **Step 5: Make the app run migrations on boot**

In `main.py`, replace `app.state.db.create_all()` with:

```python
    from alembic import command
    from alembic.config import Config as AlembicConfig

    cfg = AlembicConfig(str(Path(__file__).parent.parent / "alembic.ini"))
    cfg.set_main_option("script_location", str(Path(__file__).parent / "migrations"))
    os.environ["FF_DB_PATH"] = str(settings.db_path)
    command.upgrade(cfg, "head")
```

Add `import os` at the top, and keep `Database(settings.db_path)` before it so the directory exists.

- [ ] **Step 6: Verify boot runs migrations and the app still works**

Run: `cd backend && .venv/bin/pytest tests/core/test_db.py -v`
Expected: 3 passed

- [ ] **Step 7: Commit**

```bash
git add backend/alembic.ini backend/app/migrations backend/app/main.py
git commit -m "feat: alembic migrations, run on boot"
```

---

### Task 4: Error envelope and domain exceptions

**Files:**
- Create: `backend/app/core/errors.py`
- Modify: `backend/app/main.py` (register handlers)
- Test: `backend/tests/core/test_errors.py`

**Interfaces:**
- Consumes: `create_app` from Task 2.
- Produces: `DomainError(status: int, code: str, message: str, details: dict|None)` and subclasses `NotFound`, `ValidationFailed`, `Conflict`, `PermissionDenied`, `AuthRequired`. Every module raises these; no module writes its own HTTPException.

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/core/test_errors.py
from app.core.errors import NotFound


def test_domain_error_shape(client):
    @client.app.get("/api/_boom")
    def boom():
        raise NotFound("account_not_found", "Akun tidak ditemukan", {"id": 7})

    r = client.get("/api/_boom")
    assert r.status_code == 404
    assert r.json() == {
        "error": {
            "code": "account_not_found",
            "message": "Akun tidak ditemukan",
            "details": {"id": 7},
        }
    }


def test_unhandled_exception_does_not_leak(client):
    @client.app.get("/api/_crash")
    def crash():
        raise ValueError("secret internal detail")

    r = client.get("/api/_crash")
    assert r.status_code == 500
    assert "secret internal detail" not in r.text
    assert r.json()["error"]["code"] == "internal_error"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && .venv/bin/pytest tests/core/test_errors.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'app.core.errors'`

- [ ] **Step 3: Write `core/errors.py`**

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse


class DomainError(Exception):
    status = 500
    code = "internal_error"

    def __init__(self, code: str | None = None, message: str = "", details: dict | None = None):
        self.code = code or self.code
        self.message = message or self.code
        self.details = details
        super().__init__(self.message)


class NotFound(DomainError):
    status = 404


class ValidationFailed(DomainError):
    status = 422


class Conflict(DomainError):
    status = 409


class PermissionDenied(DomainError):
    status = 403


class AuthRequired(DomainError):
    status = 401


def _envelope(code: str, message: str, details: dict | None = None) -> dict:
    return {"error": {"code": code, "message": message, "details": details}}


def register_error_handlers(app: FastAPI) -> None:
    @app.exception_handler(DomainError)
    async def _domain(_req: Request, exc: DomainError):
        return JSONResponse(exc.status, _envelope(exc.code, exc.message, exc.details))

    @app.exception_handler(Exception)
    async def _unhandled(_req: Request, exc: Exception):
        import logging

        logging.getLogger("family_finance").exception("unhandled error")
        return JSONResponse(500, _envelope("internal_error", "Terjadi kesalahan internal"))
```

- [ ] **Step 4: Register the handlers in `create_app`**

Add `from app.core.errors import register_error_handlers` and, after the health route, `register_error_handlers(app)`.

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd backend && .venv/bin/pytest tests/core/test_errors.py -v`
Expected: 2 passed

- [ ] **Step 6: Commit**

```bash
git add backend/app/core/errors.py backend/app/main.py backend/tests/core/test_errors.py
git commit -m "feat: single error envelope and domain exceptions"
```

---

### Task 5: Money primitives

**Files:**
- Create: `backend/app/core/money.py`
- Test: `backend/tests/core/test_money.py`

**Interfaces:**
- Consumes: nothing.
- Produces:
  - `parse_rupiah(raw: str | int | None) -> int` — raises `MoneyParseError` on anything unparseable.
  - `format_rupiah(amount: int) -> str` — `1234567` → `"1.234.567"`, `-45000` → `"-45.000"`.
  - `MoneyParseError(ValueError)`

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/core/test_money.py
import pytest
from hypothesis import given, strategies as st

from app.core.money import MoneyParseError, format_rupiah, parse_rupiah


@pytest.mark.parametrize(
    "raw,expected",
    [
        ("45000", 45000),
        ("-45000", -45000),
        ("Rp 45.000", 45000),
        ("Rp 1.234.567,00", 1234567),
        ("1.234.567", 1234567),
        ("1,234,567", 1234567),
        ("(1.234,56)", -1234),
        ("1.000-", -1000),
        ("45.000 DR", -45000),
        ("45.000 CR", 45000),
        ("  12 000  ", 12000),
        (int(5000), 5000),
    ],
)
def test_parse_rupiah_formats(raw, expected):
    assert parse_rupiah(raw) == expected


@pytest.mark.parametrize("raw", ["", "abc", "Rp", "12.3.4.5x", None])
def test_parse_rupiah_rejects_garbage(raw):
    with pytest.raises(MoneyParseError):
        parse_rupiah(raw)


def test_format_rupiah():
    assert format_rupiah(1234567) == "1.234.567"
    assert format_rupiah(-45000) == "-45.000"
    assert format_rupiah(0) == "0"


@given(st.integers(min_value=-10**15, max_value=10**15))
def test_roundtrip_is_lossless(amount):
    assert parse_rupiah(format_rupiah(amount)) == amount


@given(st.text())
def test_never_returns_a_float_and_never_crashes_unexpectedly(s):
    try:
        result = parse_rupiah(s)
    except MoneyParseError:
        return
    assert isinstance(result, int) and not isinstance(result, bool)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && .venv/bin/pytest tests/core/test_money.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'app.core.money'`

- [ ] **Step 3: Write `core/money.py`**

Rupiah only. Handle the formats that actually appear in Indonesian bank exports.

```python
import re

class MoneyParseError(ValueError):
    pass

_STRIP = re.compile(r"[Rr][Pp]\s*|\s*[Dd][Rr]\b|\s*[Cc][Rr]\b|Rp")
_CREDIT = re.compile(r"\b[Cc][Rr]\b")
_DEBIT = re.compile(r"\b[Dd][Rr]\b")


def parse_rupiah(raw) -> int:
    """Parse an Indonesian-formatted amount into a signed integer rupiah value."""
    if raw is None:
        raise MoneyParseError("nil amount")
    if isinstance(raw, bool):
        raise MoneyParseError("bool amount")
    if isinstance(raw, int):
        return raw

    s = str(raw).strip()
    if not s:
        raise MoneyParseError("empty amount")

    negative = False
    if s.startswith("(") and s.endswith(")"):
        negative = True
        s = s[1:-1].strip()
    if s.endswith("-"):
        negative = True
        s = s[:-1].strip()
    if _DEBIT.search(s):
        negative = True
    elif _CREDIT.search(s):
        negative = False

    s = _STRIP.sub("", s)
    s = s.replace(" ", "").replace("\u00a0", "")

    if s.startswith("-"):
        negative = True
        s = s[1:]
    elif s.startswith("+"):
        s = s[1:]

    # Decide which separator is decimal and which is grouping.
    has_comma, has_dot = "," in s, "." in s
    if has_comma and has_dot:
        if s.rfind(",") > s.rfind("."):
            s = s.replace(".", "").replace(",", ".")   # 1.234,56
        else:
            s = s.replace(",", "")                      # 1,234.56
    elif has_comma:
        s = s.replace(",", ".") if len(s.split(",")[-1]) <= 2 else s.replace(",", "")
    elif has_dot:
        if len(s.split(".")[-1]) == 3 and s.count(".") >= 1:
            s = s.replace(".", "")                      # 1.234.567 grouping

    if not re.fullmatch(r"\d+(\.\d+)?", s):
        raise MoneyParseError(f"cannot parse amount: {raw!r}")

    return -int(s.split(".")[0]) if negative else int(s.split(".")[0])


def format_rupiah(amount: int) -> str:
    """Format an integer rupiah value with dot grouping. Never returns a float."""
    return f"{int(amount):,}".replace(",", ".")
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && .venv/bin/pytest tests/core/test_money.py -v`
Expected: all passed. If `1,234,567` or `(1.234,56)` fails, fix the separator logic — do not loosen the test.

- [ ] **Step 5: Add the codebase-wide float ban test**

Append to `backend/tests/core/test_money.py`:

```python
import pathlib
import re

FLOAT_LITERAL = re.compile(r"(?<![\w.])\d+\.\d+(?![\w.])")


def test_no_float_literals_in_money_paths():
    root = pathlib.Path(__file__).resolve().parents[2] / "app"
    offenders = []
    for path in root.rglob("*.py"):
        for i, line in enumerate(path.read_text().splitlines(), 1):
            if line.lstrip().startswith("#"):
                continue
            if "float(" in line or FLOAT_LITERAL.search(line):
                offenders.append(f"{path.relative_to(root)}:{i}: {line.strip()}")
    assert not offenders, "float literals in money paths:\n" + "\n".join(offenders)
```

- [ ] **Step 6: Run the full backend suite**

Run: `cd backend && .venv/bin/pytest -v`
Expected: all passed

- [ ] **Step 7: Commit**

```bash
git add backend/app/core/money.py backend/tests/core/test_money.py
git commit -m "feat: rupiah money primitives with integer-only guarantee"
```

---

### Task 6: Docker and compose

**Files:**
- Create: `Dockerfile` (repo root)
- Create: `docker-compose.yml`
- Create: `frontend/package.json`, `frontend/nuxt.config.ts`, `frontend/app.vue` (minimal placeholder, replaced in Task 7)

**Interfaces:**
- Consumes: `create_app` from Task 2.
- Produces: a single image serving API + SPA on port 8000, with `/data/app.db` on a named volume.

- [ ] **Step 1: Create a minimal frontend so the build stage has something to compile**

`frontend/package.json`:

```json
{
  "name": "family-finance-frontend",
  "private": true,
  "type": "module",
  "scripts": { "build": "nuxt build", "generate": "nuxt generate", "dev": "nuxt dev" },
  "dependencies": { "nuxt": "^3.14.0", "vue": "^3.5.0" },
  "devDependencies": { "@nuxtjs/tailwindcss": "^6.13.0" }
}
```

`frontend/nuxt.config.ts`:

```ts
export default defineNuxtConfig({
  ssr: false,
  modules: ['@nuxtjs/tailwindcss'],
  css: ['~/assets/css/main.css'],
  app: {
    head: { title: 'Keluarga Kita', htmlAttrs: { lang: 'id' } }
  }
})
```

`frontend/app.vue`:

```vue
<template><NuxtPage /></template>
```

- [ ] **Step 2: Write the multi-stage `Dockerfile`**

```dockerfile
FROM node:22-alpine AS frontend
WORKDIR /fe
COPY frontend/package.json frontend/package-lock.json* ./
RUN npm install
COPY frontend/ ./
RUN npm run generate

FROM python:3.12-slim AS runtime
WORKDIR /srv
ENV PYTHONUNBUFFERED=1 FF_DB_PATH=/data/app.db
COPY backend/pyproject.toml ./
RUN pip install --no-cache-dir -e . || pip install --no-cache-dir .
COPY backend/ ./
COPY --from=frontend /fe/.output/public ./app/static
VOLUME ["/data"]
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=3s \
  CMD python -c "import urllib.request,sys; sys.exit(0 if urllib.request.urlopen('http://127.0.0.1:8000/api/health').status==200 else 1)"
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "1"]
```

- [ ] **Step 3: Write `docker-compose.yml`**

```yaml
services:
  app:
    build: .
    ports:
      - "8000:8000"
    volumes:
      - ffdata:/data
    environment:
      FF_DB_PATH: /data/app.db
      FF_HOUSEHOLD_NAME: Keluarga Kita
    restart: unless-stopped

volumes:
  ffdata:
```

- [ ] **Step 4: Verify the image builds and serves the health endpoint**

Run: `cd ~/Projects/family-finance && docker compose build && docker compose up -d && sleep 5 && curl -s localhost:8000/api/health && docker compose down`
Expected: `{"status":"ok"}`

- [ ] **Step 5: Commit**

```bash
git add Dockerfile docker-compose.yml frontend/package.json frontend/nuxt.config.ts frontend/app.vue
git commit -m "feat: multi-stage docker build serving api and spa on one port"
```

---

### Task 7: Tailwind theme and brutalist primitives

**Files:**
- Create: `frontend/assets/css/main.css`
- Create: `frontend/tailwind.config.ts`
- Modify: `frontend/app.vue`

**Interfaces:**
- Consumes: the frontend scaffold from Task 6.
- Produces: tokens `--color-paper|ink|in|out|hi|info|cat|muted` available as Tailwind classes (`bg-paper`, `text-ink`, `bg-in`, `text-out`, `bg-hi`, `bg-info`, `bg-cat`, `text-muted`, `rounded-brutal`) and utilities `border-brutal`, `shadow-brutal`, `press`.

- [ ] **Step 1: Write `assets/css/main.css`**

```css
@import "tailwindcss";

@theme {
  --color-paper: #FDFCF7;
  --color-ink: #111111;
  --color-in: #06D6A0;
  --color-out: #EF476F;
  --color-hi: #FFD166;
  --color-info: #118AB2;
  --color-cat: #C9B6FF;
  --color-muted: #8D99AE;
  --radius-brutal: 12px;
  --font-sans: ui-sans-serif, system-ui, -apple-system, "Segoe UI", sans-serif;
}

@utility border-brutal { border: 2.5px solid var(--color-ink); }
@utility shadow-brutal { box-shadow: 4px 4px 0 var(--color-ink); }
@utility press { box-shadow: none; translate: 4px 4px; }

@layer base {
  body { background: var(--color-paper); color: var(--color-ink); font-family: var(--font-sans); }
  .money { font-variant-numeric: tabular-nums; font-weight: 800; }
}
```

- [ ] **Step 2: Write `tailwind.config.ts`**

```ts
export default { content: ["./components/**/*.vue", "./layouts/**/*.vue", "./pages/**/*.vue", "./app.vue"] }
```

- [ ] **Step 3: Prove the tokens compile, in a throwaway page**

Temporarily set `frontend/app.vue` to:

```vue
<template>
  <div class="p-10 bg-paper">
    <div class="border-brutal shadow-brutal rounded-brutal bg-hi p-6 text-ink font-extrabold active:press">
      <span class="money text-out">−45.000</span> token check
    </div>
  </div>
</template>
```

- [ ] **Step 4: Verify the classes survive the build**

Run: `cd frontend && npm install && npm run generate && grep -rq "shadow-brutal\|box-shadow:4px 4px 0\|box-shadow: 4px 4px 0" .output/public/_nuxt/*.css && echo "primitives present"`
Expected: `primitives present`

- [ ] **Step 5: Commit**

```bash
git add frontend/assets/css/main.css frontend/tailwind.config.ts frontend/app.vue
git commit -m "feat: family play design tokens and brutalist primitives"
```

---

# Phase 2 — Members & auth

### Task 8: Member and session models

**Files:**
- Create: `backend/app/modules/auth/__init__.py`, `backend/app/modules/auth/models.py`
- Create: `backend/app/modules/auth/schemas.py`
- Create: `backend/app/migrations/versions/<hash>_members_sessions.py` (generated)

**Interfaces:**
- Consumes: `Base` from Task 2.
- Produces: `Member` (id, name, username, password_hash, role, avatar_color, is_active, created_at), `Session` (id, member_id, created_at, expires_at, last_seen_at, user_agent), and `MemberOut` pydantic schema.

- [ ] **Step 1: Write `modules/auth/models.py`**

```python
from datetime import datetime

from sqlalchemy import ForeignKey, Integer, String, Text, Boolean, DateTime, func
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.core.db import Base


class Member(Base):
    __tablename__ = "members"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    name: Mapped[str] = mapped_column(String, nullable=False)
    username: Mapped[str] = mapped_column(String, nullable=False, unique=True)
    password_hash: Mapped[str] = mapped_column(Text, nullable=False)
    role: Mapped[str] = mapped_column(String, nullable=False, default="adult", server_default="adult")
    avatar_color: Mapped[str] = mapped_column(String, nullable=False, default="#118AB2")
    is_active: Mapped[bool] = mapped_column(Boolean, nullable=False, default=True)
    created_at: Mapped[datetime] = mapped_column(DateTime, server_default=func.now())


class Session(Base):
    __tablename__ = "sessions"

    id: Mapped[str] = mapped_column(String, primary_key=True)
    member_id: Mapped[int] = mapped_column(ForeignKey("members.id", ondelete="CASCADE"), index=True)
    created_at: Mapped[datetime] = mapped_column(DateTime, server_default=func.now())
    expires_at: Mapped[datetime] = mapped_column(DateTime, nullable=False)
    last_seen_at: Mapped[datetime] = mapped_column(DateTime, server_default=func.now())
    user_agent: Mapped[str | None] = mapped_column(String, nullable=True)

    member: Mapped[Member] = relationship()
```

Add a `CheckConstraint` on `role IN ('adult','kid')` — the kid value is never written in v1 but the constraint documents the domain:

```python
from sqlalchemy import CheckConstraint

    __table_args__ = (CheckConstraint("role IN ('adult','kid')", name="ck_members_role"),)
```

- [ ] **Step 2: Write `modules/auth/schemas.py`**

```python
from pydantic import BaseModel, ConfigDict, Field


class MemberOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    name: str
    username: str
    role: str
    avatar_color: str


class SetupIn(BaseModel):
    name: str = Field(min_length=1, max_length=80)
    username: str = Field(min_length=3, max_length=40)
    password: str = Field(min_length=12)
    avatar_color: str = "#118AB2"


class LoginIn(BaseModel):
    username: str
    password: str


class AuthStateOut(BaseModel):
    needs_setup: bool
    member: MemberOut | None = None
```

- [ ] **Step 3: Generate and inspect the migration**

Run: `cd backend && .venv/bin/python -m alembic revision --autogenerate -m "members and sessions"`
Expected: a new file in `app/migrations/versions/`. Open it and confirm it contains `op.create_table('members'` and `op.create_table('sessions'` with the FK on `sessions.member_id`.

- [ ] **Step 4: Verify the tables are created on a fresh database**

Run: `cd backend && .venv/bin/pytest tests/core/test_db.py -v && .venv/bin/python -c "from app.core.db import Database; from pathlib import Path; import tempfile; d=Database(Path(tempfile.mkdtemp())/'x.db'); d.create_all(); from sqlalchemy import text; print([r[0] for r in d.engine.connect().execute(text(\\\"select name from sqlite_master where type='table'\\\"))])"`
Expected: includes `members` and `sessions`

- [ ] **Step 5: Commit**

```bash
git add backend/app/modules/auth backend/app/migrations/versions
git commit -m "feat: member and session models with migration"
```

---

### Task 9: Password hashing, session tokens, and current-member dependency

**Files:**
- Create: `backend/app/core/security.py`
- Create: `backend/app/core/deps.py`
- Test: `backend/tests/core/test_security.py`

**Interfaces:**
- Consumes: `Member`, `Session` (Task 8), `Settings` (Task 2).
- Produces:
  - `hash_password(pw: str) -> str`, `verify_password(pw: str, hashed: str) -> bool`
  - `new_session_token() -> str`
  - `optional_member(request, db) -> Member | None`, `current_member(request, db) -> Member` (raises `AuthRequired`)

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/core/test_security.py
from app.core.security import hash_password, new_session_token, verify_password


def test_password_roundtrip():
    h = hash_password("correct horse battery")
    assert h.startswith("$argon2id$")
    assert verify_password("correct horse battery", h)
    assert not verify_password("wrong", h)


def test_session_tokens_are_unique_and_long():
    tokens = {new_session_token() for _ in range(200)}
    assert len(tokens) == 200
    assert all(len(t) == 64 for t in tokens)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && .venv/bin/pytest tests/core/test_security.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'app.core.security'`

- [ ] **Step 3: Write `core/security.py`**

```python
import secrets

from argon2 import PasswordHasher
from argon2.exceptions import VerifyMismatchError, VerificationError, InvalidHashError

_hasher = PasswordHasher()


def hash_password(password: str) -> str:
    return _hasher.hash(password)


def verify_password(password: str, hashed: str) -> bool:
    try:
        return _hasher.verify(hashed, password)
    except (VerifyMismatchError, VerificationError, InvalidHashError):
        return False


def new_session_token() -> str:
    return secrets.token_hex(32)
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd backend && .venv/bin/pytest tests/core/test_security.py -v`
Expected: 2 passed

- [ ] **Step 5: Write `core/deps.py`**

```python
from datetime import datetime, timedelta

from fastapi import Request
from sqlalchemy import select

from app.core.errors import AuthRequired
from app.core.security import verify_password
from app.modules.auth.models import Member, Session

COOKIE_NAME = "ff_session"


def _db(request: Request):
    return request.app.state.db


def optional_member(request: Request) -> Member | None:
    token = request.cookies.get(COOKIE_NAME)
    if not token:
        return None
    with _db(request).session() as s:
        sess = s.get(Session, token)
        if sess is None or sess.expires_at < datetime.utcnow():
            return None
        memb = s.get(Member, sess.member_id)
        if memb is None or not memb.is_active:
            return None
        sess.last_seen_at = datetime.utcnow()
        s.flush()
        return memb


def current_member(request: Request) -> Member:
    member = optional_member(request)
    if member is None:
        raise AuthRequired("auth_required", "Silakan masuk terlebih dahulu")
    return member
```

- [ ] **Step 6: Commit**

```bash
git add backend/app/core/security.py backend/app/core/deps.py backend/tests/core/test_security.py
git commit -m "feat: argon2 passwords, session tokens, current-member dependency"
```

---

### Task 10: Auth service and setup wizard

**Files:**
- Create: `backend/app/modules/auth/service.py`
- Test: `backend/tests/modules/auth/test_auth_service.py`

**Interfaces:**
- Consumes: `Member`, `Session`, `hash_password`, `verify_password`, `new_session_token`.
- Produces:
  - `needs_setup(session) -> bool`
  - `create_member(session, name, username, password, avatar_color, role="adult") -> Member` (raises `Conflict` on duplicate username, `ValidationFailed` on short password)
  - `authenticate(session, username, password) -> Member | None`
  - `start_session(session, member, user_agent, ttl_days) -> Session`
  - `end_session(session, token) -> None`, `end_all_sessions(session, member_id) -> int`

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/modules/auth/test_auth_service.py
import pytest

from app.core.errors import Conflict, ValidationFailed
from app.modules.auth import service
from app.modules.auth.models import Member, Session


def _mk(db, username="budi", password="supersecret123"):
    with db.session() as s:
        return service.create_member(s, name="Budi", username=username,
                                     password=password, avatar_color="#EF476F").id


def test_needs_setup_true_when_empty(db):
    with db.session() as s:
        assert service.needs_setup(s) is True


def test_needs_setup_false_after_first_member(db):
    _mk(db)
    with db.session() as s:
        assert service.needs_setup(s) is False


def test_password_shorter_than_12_rejected(db):
    with db.session() as s:
        with pytest.raises(ValidationFailed):
            service.create_member(s, name="A", username="a", password="short",
                                  avatar_color="#000000")


def test_duplicate_username_conflicts(db):
    _mk(db)
    with db.session() as s:
        with pytest.raises(Conflict):
            service.create_member(s, name="B", username="budi", password="supersecret123",
                                  avatar_color="#000000")


def test_authenticate_ok_and_fail(db):
    _mk(db)
    with db.session() as s:
        assert service.authenticate(s, "budi", "supersecret123") is not None
        assert service.authenticate(s, "budi", "nope") is None
        assert service.authenticate(s, "ghost", "supersecret123") is None


def test_session_lifecycle(db):
    member_id = _mk(db)
    with db.session() as s:
        member = s.get(Member, member_id)
        token = service.start_session(s, member, user_agent="pytest", ttl_days=30).id
    with db.session() as s:
        service.end_session(s, token)
        assert s.get(Session, token) is None


def test_end_all_sessions(db):
    member_id = _mk(db)
    with db.session() as s:
        member = s.get(Member, member_id)
        service.start_session(s, member, user_agent="a", ttl_days=30)
        service.start_session(s, member, user_agent="b", ttl_days=30)
    with db.session() as s:
        assert service.end_all_sessions(s, member_id) == 2
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && .venv/bin/pytest tests/modules/auth/test_auth_service.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'app.modules.auth.service'`

- [ ] **Step 3: Write `modules/auth/service.py`**

```python
from datetime import datetime, timedelta

from sqlalchemy import delete, func, select

from app.core.errors import Conflict, NotFound, ValidationFailed
from app.core.security import hash_password, new_session_token, verify_password
from app.modules.auth.models import Member, Session

MIN_PASSWORD_LENGTH = 12


def needs_setup(s) -> bool:
    return s.execute(select(func.count()).select_from(Member)).scalar_one() == 0


def create_member(s, *, name: str, username: str, password: str,
                  avatar_color: str = "#118AB2", role: str = "adult") -> Member:
    username = username.strip().lower()
    if len(password) < MIN_PASSWORD_LENGTH:
        raise ValidationFailed("password_too_short",
                               f"Kata sandi minimal {MIN_PASSWORD_LENGTH} karakter")
    if not username:
        raise ValidationFailed("username_required", "Nama pengguna wajib diisi")
    exists = s.execute(select(Member).where(Member.username == username)).scalar_one_or_none()
    if exists is not None:
        raise Conflict("username_taken", "Nama pengguna sudah dipakai")
    member = Member(name=name.strip(), username=username,
                    password_hash=hash_password(password),
                    avatar_color=avatar_color, role=role)
    s.add(member)
    s.flush()
    return member


def authenticate(s, username: str, password: str) -> Member | None:
    member = s.execute(
        select(Member).where(Member.username == username.strip().lower())
    ).scalar_one_or_none()
    if member is None or not member.is_active:
        return None
    if not verify_password(password, member.password_hash):
        return None
    return member


def get_member(s, member_id: int) -> Member:
    member = s.get(Member, member_id)
    if member is None:
        raise NotFound("member_not_found", "Anggota tidak ditemukan")
    return member


def list_members(s) -> list[Member]:
    return list(s.execute(select(Member).order_by(Member.id)).scalars())


def start_session(s, member: Member, *, user_agent: str | None, ttl_days: int) -> Session:
    now = datetime.utcnow()
    sess = Session(id=new_session_token(), member_id=member.id,
                   created_at=now, expires_at=now + timedelta(days=ttl_days),
                   last_seen_at=now, user_agent=user_agent)
    s.add(sess)
    s.flush()
    return sess


def end_session(s, token: str) -> None:
    s.execute(delete(Session).where(Session.id == token))


def end_all_sessions(s, member_id: int) -> int:
    result = s.execute(delete(Session).where(Session.member_id == member_id))
    return result.rowcount or 0


def set_password(s, member: Member, password: str) -> None:
    if len(password) < MIN_PASSWORD_LENGTH:
        raise ValidationFailed("password_too_short",
                               f"Kata sandi minimal {MIN_PASSWORD_LENGTH} karakter")
    member.password_hash = hash_password(password)
    s.flush()
```

- [ ] **Step 4: Make the `db` fixture create the tables**

The service tests use `db` directly, so the schema must exist before they run. In `backend/tests/conftest.py`, change the `db` fixture to:

```python
@pytest.fixture
def db(tmp_path):
    database = Database(tmp_path / "test.db")
    database.create_all()
    return database
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd backend && .venv/bin/pytest tests/modules/auth/test_auth_service.py -v`
Expected: 7 passed

- [ ] **Step 6: Commit**

```bash
git add backend/app/modules/auth/service.py backend/tests/modules/auth backend/tests/conftest.py
git commit -m "feat: auth service with setup detection, member creation, sessions"
```

---

### Task 11: Login rate limiting

**Files:**
- Create: `backend/app/modules/auth/ratelimit.py`
- Test: `backend/tests/modules/auth/test_ratelimit.py`

**Interfaces:**
- Consumes: nothing.
- Produces: `LoginRateLimiter()` with `.check(key: str) -> None` (raises `ValidationFailed` with code `too_many_attempts`), `.record_failure(key)`, `.reset(key)`, and `.retry_after(key) -> int`.

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/modules/auth/test_ratelimit.py
import pytest

from app.core.errors import ValidationFailed
from app.modules.auth.ratelimit import LoginRateLimiter


def test_allows_until_threshold_then_blocks():
    rl = LoginRateLimiter(threshold=3, base_delay_seconds=60, clock=lambda: 1000.0)
    key = "budi|127.0.0.1"
    for _ in range(4):          # threshold + 1 failures
        rl.check(key)
        rl.record_failure(key)
    with pytest.raises(ValidationFailed) as e:
        rl.check(key)
    assert e.value.code == "too_many_attempts"


def test_delay_grows_and_expires():
    now = [1000.0]
    rl = LoginRateLimiter(threshold=2, base_delay_seconds=10, clock=lambda: now[0])
    key = "budi|127.0.0.1"
    for _ in range(3):
        rl.record_failure(key)
    assert rl.retry_after(key) == 20          # 2^(3-2) * 10
    now[0] += 21
    rl.check(key)                              # no raise after the window


def test_success_resets():
    rl = LoginRateLimiter(threshold=2, base_delay_seconds=10, clock=lambda: 1000.0)
    key = "budi|127.0.0.1"
    rl.record_failure(key)
    rl.record_failure(key)
    rl.reset(key)
    rl.check(key)                              # no raise


def test_keys_are_independent():
    rl = LoginRateLimiter(threshold=1, base_delay_seconds=60, clock=lambda: 1000.0)
    rl.record_failure("budi|1.1.1.1")
    rl.check("siti|2.2.2.2")                   # different username, no raise
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && .venv/bin/pytest tests/modules/auth/test_ratelimit.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'app.modules.auth.ratelimit'`

- [ ] **Step 3: Write `modules/auth/ratelimit.py`**

In-process, no Redis. Exponential backoff after `threshold` failures.

```python
import time

from app.core.errors import ValidationFailed


class LoginRateLimiter:
    def __init__(self, *, threshold: int = 5, base_delay_seconds: int = 30, clock=time.monotonic):
        self.threshold = threshold
        self.base_delay = base_delay_seconds
        self.clock = clock
        self._failures: dict[str, list[float]] = {}

    def _recent(self, key: str) -> list[float]:
        now = self.clock()
        kept = [t for t in self._failures.get(key, []) if now - t < self._window(key)]
        self._failures[key] = kept
        return kept

    def _window(self, key: str) -> float:
        n = len(self._failures.get(key, []))
        if n <= self.threshold:
            return self.base_delay * 60
        return self.base_delay * (2 ** (n - self.threshold))

    def retry_after(self, key: str) -> int:
        failures = self._failures.get(key, [])
        if len(failures) <= self.threshold:
            return 0
        delay = self.base_delay * (2 ** (len(failures) - self.threshold))
        elapsed = self.clock() - failures[-1]
        return max(0, int(delay - elapsed))

    def check(self, key: str) -> None:
        failures = self._recent(key)
        if len(failures) <= self.threshold:
            return
        delay = self.base_delay * (2 ** (len(failures) - self.threshold))
        if self.clock() - failures[-1] < delay:
            raise ValidationFailed(
                "too_many_attempts",
                f"Terlalu banyak percobaan. Coba lagi dalam {self.retry_after(key)} detik",
            )

    def record_failure(self, key: str) -> None:
        self._failures.setdefault(key, []).append(self.clock())

    def reset(self, key: str) -> None:
        self._failures.pop(key, None)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && .venv/bin/pytest tests/modules/auth/test_ratelimit.py -v`
Expected: 4 passed

- [ ] **Step 5: Commit**

```bash
git add backend/app/modules/auth/ratelimit.py backend/tests/modules/auth/test_ratelimit.py
git commit -m "feat: in-process login rate limiter with exponential backoff"
```

---

### Task 12: Same-origin middleware

**Files:**
- Create: `backend/app/core/middleware.py`
- Test: `backend/tests/core/test_middleware.py`

**Interfaces:**
- Consumes: `DomainError` from Task 4.
- Produces: `SameOriginMiddleware` — rejects state-changing requests whose `Origin` header, when present, does not match the request `Host`, and rejects JSON-bodied methods without `Content-Type: application/json`.

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/core/test_middleware.py

def test_cross_origin_post_is_rejected(client):
    r = client.post("/api/health", headers={"Origin": "http://evil.example"},
                    json={})
    assert r.status_code == 403
    assert r.json()["error"]["code"] == "cross_origin_rejected"


def test_same_origin_post_is_allowed(client):
    from fastapi import Request

    @client.app.post("/api/_echo")
    async def echo(request: Request):
        return await request.json()

    r = client.post("/api/_echo", headers={"Origin": "http://testserver",
                                          "Content-Type": "application/json"}, json={"a": 1})
    assert r.status_code == 200
    assert r.json() == {"a": 1}


def test_get_needs_no_content_type(client):
    assert client.get("/api/health").status_code == 200


def test_json_post_without_content_type_is_rejected(client):
    @client.app.post("/api/_echo2")
    async def echo2():
        return {}

    r = client.post("/api/_echo2", headers={"Content-Type": "text/plain"}, content=b"{}")
    assert r.status_code == 422
    assert r.json()["error"]["code"] == "unsupported_media_type"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && .venv/bin/pytest tests/core/test_middleware.py -v`
Expected: FAIL — the cross-origin POST returns something other than 403

- [ ] **Step 3: Write `core/middleware.py`**

```python
from urllib.parse import urlparse

from starlette.middleware.base import BaseHTTPMiddleware
from starlette.responses import JSONResponse

SAFE_METHODS = {"GET", "HEAD", "OPTIONS"}
JSON_METHODS = {"POST", "PUT", "PATCH", "DELETE"}


def _envelope(code: str, message: str) -> dict:
    return {"error": {"code": code, "message": message, "details": None}}


class SameOriginMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        if request.method in JSON_METHODS:
            origin = request.headers.get("origin")
            if origin:
                origin_host = urlparse(origin).netloc
                if origin_host and origin_host != request.headers.get("host"):
                    return JSONResponse(403, _envelope(
                        "cross_origin_rejected", "Permintaan lintas situs ditolak"))
            ctype = request.headers.get("content-type", "")
            if not ctype.startswith("application/json") and "multipart/form-data" not in ctype:
                return JSONResponse(422, _envelope(
                    "unsupported_media_type", "Content-Type harus application/json"))
        return await call_next(request)
```

- [ ] **Step 4: Register the middleware in `create_app`**

Add near the top of `create_app`, after `app.state.db` is set:

```python
    from app.core.middleware import SameOriginMiddleware
    app.add_middleware(SameOriginMiddleware)
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd backend && .venv/bin/pytest tests/core/test_middleware.py -v`
Expected: 4 passed

- [ ] **Step 6: Commit**

```bash
git add backend/app/core/middleware.py backend/app/main.py backend/tests/core/test_middleware.py
git commit -m "feat: same-origin and content-type middleware"
```

---

### Task 13: Auth router — setup, login, logout, members

**Files:**
- Create: `backend/app/modules/auth/router.py`
- Modify: `backend/app/main.py` (include router)
- Test: `backend/tests/modules/auth/test_auth_api.py`

**Interfaces:**
- Consumes: `service`, `schemas`, `LoginRateLimiter`, `optional_member`, `current_member`, `COOKIE_NAME`.
- Produces: routes
  - `GET  /api/auth/state` → `AuthStateOut`
  - `POST /api/setup` → `MemberOut`, sets cookie (409 if already set up)
  - `POST /api/auth/login` → `MemberOut`, sets cookie
  - `POST /api/auth/logout` → `{"ok": true}`, clears cookie
  - `GET  /api/auth/me` → `MemberOut`
  - `GET  /api/members` → `list[MemberOut]`
  - `POST /api/members` → `MemberOut` (second adult)
  - `POST /api/members/{id}/password` → `{"ok": true}` (an adult resets another member's password; also ends that member's sessions)
  - `POST /api/auth/logout-all` → `{"ok": true, "ended": int}`

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/modules/auth/test_auth_api.py

def test_flow_setup_then_login_then_me_then_logout(client):
    assert client.get("/api/auth/state").json() == {"needs_setup": True, "member": None}

    r = client.post("/api/setup", json={"name": "Budi", "username": "budi",
                                        "password": "supersecret123"})
    assert r.status_code == 200
    assert r.json()["username"] == "budi"
    assert client.cookies.get("ff_session")

    assert client.get("/api/auth/state").json()["needs_setup"] is False
    assert client.get("/api/auth/me").json()["name"] == "Budi"

    client.post("/api/auth/logout")
    assert client.get("/api/auth/me").status_code == 401


def test_setup_twice_is_conflict(client):
    client.post("/api/setup", json={"name": "Budi", "username": "budi",
                                    "password": "supersecret123"})
    r = client.post("/api/setup", json={"name": "X", "username": "x",
                                        "password": "supersecret123"})
    assert r.status_code == 409
    assert r.json()["error"]["code"] == "already_set_up"


def test_short_password_is_rejected(client):
    r = client.post("/api/setup", json={"name": "Budi", "username": "budi", "password": "short"})
    assert r.status_code == 422


def test_login_rate_limit_kicks_in(client):
    client.post("/api/setup", json={"name": "Budi", "username": "budi",
                                    "password": "supersecret123"})
    client.post("/api/auth/logout")
    codes = []
    for _ in range(12):
        codes.append(client.post("/api/auth/login",
                                 json={"username": "budi", "password": "wrong"}).status_code)
    assert 422 in codes


def test_second_member_can_be_added_by_an_adult(client):
    client.post("/api/setup", json={"name": "Budi", "username": "budi",
                                    "password": "supersecret123"})
    r = client.post("/api/members", json={"name": "Siti", "username": "siti",
                                          "password": "anothersecret123",
                                          "avatar_color": "#FFD166"})
    assert r.status_code == 200
    assert len(client.get("/api/members").json()) == 2


def test_adult_can_reset_another_members_password(client):
    client.post("/api/setup", json={"name": "Budi", "username": "budi",
                                    "password": "supersecret123"})
    client.post("/api/members", json={"name": "Siti", "username": "siti",
                                      "password": "anothersecret123"})
    siti = [m for m in client.get("/api/members").json() if m["username"] == "siti"][0]
    assert client.post(f"/api/members/{siti['id']}/password",
                       json={"password": "brandnewsecret1"}).status_code == 200
    client.post("/api/auth/logout")
    assert client.post("/api/auth/login", json={"username": "siti",
                                                "password": "anothersecret123"}).status_code == 401
    assert client.post("/api/auth/login", json={"username": "siti",
                                                "password": "brandnewsecret1"}).status_code == 200
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && .venv/bin/pytest tests/modules/auth/test_auth_api.py -v`
Expected: FAIL — 404 on `/api/auth/state`

- [ ] **Step 3: Write `modules/auth/router.py`**

```python
from fastapi import APIRouter, Request, Response
from pydantic import BaseModel

from app.core.deps import COOKIE_NAME, current_member, optional_member
from app.core.errors import Conflict
from app.modules.auth import service
from app.modules.auth.ratelimit import LoginRateLimiter
from app.modules.auth.schemas import AuthStateOut, LoginIn, MemberOut, SetupIn

router = APIRouter(prefix="/api", tags=["auth"])
_limiter = LoginRateLimiter()


def _set_cookie(response: Response, token: str, request: Request) -> None:
    settings = request.app.state.settings
    response.set_cookie(
        COOKIE_NAME, token,
        max_age=settings.session_ttl_days * 86400,
        httponly=True, samesite="lax", secure=settings.cookie_secure, path="/",
    )


@router.get("/auth/state", response_model=AuthStateOut)
def auth_state(request: Request):
    with request.app.state.db.session() as s:
        member = optional_member(request)
        return AuthStateOut(needs_setup=service.needs_setup(s),
                            member=MemberOut.model_validate(member) if member else None)


@router.post("/setup", response_model=MemberOut)
def setup(payload: SetupIn, request: Request, response: Response):
    with request.app.state.db.session() as s:
        if not service.needs_setup(s):
            raise Conflict("already_set_up", "Aplikasi sudah dikonfigurasi")
        member = service.create_member(
            s, name=payload.name, username=payload.username,
            password=payload.password, avatar_color=payload.avatar_color)
        sess = service.start_session(s, member, user_agent=request.headers.get("user-agent"),
                                     ttl_days=request.app.state.settings.session_ttl_days)
        _set_cookie(response, sess.id, request)
        return MemberOut.model_validate(member)


@router.post("/auth/login", response_model=MemberOut)
def login(payload: LoginIn, request: Request, response: Response):
    client_ip = request.client.host if request.client else "unknown"
    key = f"{payload.username.lower()}|{client_ip}"
    _limiter.check(key)
    with request.app.state.db.session() as s:
        member = service.authenticate(s, payload.username, payload.password)
        if member is None:
            _limiter.record_failure(key)
            from app.core.errors import AuthRequired
            raise AuthRequired("invalid_credentials", "Nama pengguna atau kata sandi salah")
        _limiter.reset(key)
        sess = service.start_session(s, member, user_agent=request.headers.get("user-agent"),
                                     ttl_days=request.app.state.settings.session_ttl_days)
        _set_cookie(response, sess.id, request)
        return MemberOut.model_validate(member)


@router.post("/auth/logout")
def logout(request: Request, response: Response):
    token = request.cookies.get(COOKIE_NAME)
    if token:
        with request.app.state.db.session() as s:
            service.end_session(s, token)
    response.delete_cookie(COOKIE_NAME, path="/")
    return {"ok": True}


@router.get("/auth/me", response_model=MemberOut)
def me(request: Request):
    return MemberOut.model_validate(current_member(request))


@router.get("/members", response_model=list[MemberOut])
def list_members(request: Request):
    current_member(request)
    with request.app.state.db.session() as s:
        return [MemberOut.model_validate(m) for m in service.list_members(s)]


@router.post("/members", response_model=MemberOut)
def add_member(payload: SetupIn, request: Request):
    current_member(request)
    with request.app.state.db.session() as s:
        member = service.create_member(
            s, name=payload.name, username=payload.username,
            password=payload.password, avatar_color=payload.avatar_color)
        return MemberOut.model_validate(member)


@router.post("/auth/logout-all")
def logout_all(request: Request, response: Response):
    member = current_member(request)
    with request.app.state.db.session() as s:
        count = service.end_all_sessions(s, member.id)
    response.delete_cookie(COOKIE_NAME, path="/")
    return {"ok": True, "ended": count}


class PasswordResetIn(BaseModel):
    password: str


@router.post("/members/{member_id}/password")
def reset_member_password(member_id: int, payload: PasswordResetIn, request: Request):
    """An adult resets another member's password. Their sessions are ended so the
    new password is actually required everywhere."""
    current_member(request)
    with request.app.state.db.session() as s:
        target = service.get_member(s, member_id)
        service.set_password(s, target, payload.password)
        service.end_all_sessions(s, target.id)
    return {"ok": True}
```

- [ ] **Step 4: Include the router in `create_app`**

```python
    from app.modules.auth.router import router as auth_router
    app.include_router(auth_router)
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd backend && .venv/bin/pytest -v`
Expected: all passed

- [ ] **Step 6: Commit**

```bash
git add backend/app/modules/auth/router.py backend/app/main.py backend/tests/modules/auth/test_auth_api.py
git commit -m "feat: auth router with setup wizard, login, logout, members"
```

---

# Phase 3 — Accounts & categories

### Task 14: Settings store (household config)

**Files:**
- Create: `backend/app/core/settings_store.py`
- Modify: `backend/app/core/db.py` (import the model in `create_all`)
- Modify: `backend/app/migrations/env.py` (import the model)
- Test: `backend/tests/core/test_settings_store.py`

**Interfaces:**
- Consumes: `Base` from Task 2.
- Produces: `Setting` (key, value), `get_setting(s, key, default=None) -> str | None`, `set_setting(s, key, value)`, `household_name(s, default) -> str`.

The `settings` table is core infrastructure, not a domain module — it lives in `core/`, so it is not part of the module contract.

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/core/test_settings_store.py
from app.core.settings_store import get_setting, household_name, set_setting


def test_default_when_absent(db):
    with db.session() as s:
        assert get_setting(s, "currency", "IDR") == "IDR"
        assert household_name(s, "Keluarga Kita") == "Keluarga Kita"


def test_set_then_get_then_overwrite(db):
    with db.session() as s:
        set_setting(s, "household_name", "Rumah Kami")
    with db.session() as s:
        assert household_name(s, "fallback") == "Rumah Kami"
        set_setting(s, "household_name", "Rumah Baru")
    with db.session() as s:
        assert household_name(s, "fallback") == "Rumah Baru"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && .venv/bin/pytest tests/core/test_settings_store.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'app.core.settings_store'`

- [ ] **Step 3: Write `core/settings_store.py`**

```python
from sqlalchemy import String, Text
from sqlalchemy.orm import Mapped, mapped_column

from app.core.db import Base


class Setting(Base):
    __tablename__ = "settings"

    key: Mapped[str] = mapped_column(String, primary_key=True)
    value: Mapped[str] = mapped_column(Text, nullable=False)


def get_setting(s, key: str, default: str | None = None) -> str | None:
    row = s.get(Setting, key)
    return row.value if row is not None else default


def set_setting(s, key: str, value: str) -> None:
    row = s.get(Setting, key)
    if row is None:
        s.add(Setting(key=key, value=value))
    else:
        row.value = value
    s.flush()


def household_name(s, default: str) -> str:
    return get_setting(s, "household_name", default) or default
```

- [ ] **Step 4: Register the model so Alembic and `create_all` can see it**

In `backend/app/core/db.py`, change the import line inside `create_all` to:

```python
        from app.core import settings_store  # noqa: F401
        from app.modules import accounts, auth, categories, imports, transactions  # noqa: F401
```

In `backend/app/migrations/env.py`, add alongside the existing import:

```python
from app.core import settings_store  # noqa: F401
```

- [ ] **Step 5: Generate the migration and run tests**

Run: `cd backend && .venv/bin/python -m alembic revision --autogenerate -m "settings table" && .venv/bin/pytest tests/core/test_settings_store.py -v`
Expected: migration created containing `op.create_table('settings'`; 2 passed

- [ ] **Step 6: Commit**

```bash
git add backend/app/core/settings_store.py backend/app/core/db.py backend/app/migrations/env.py backend/app/migrations/versions backend/tests/core/test_settings_store.py
git commit -m "feat: household settings store"
```

---

### Task 15: Accounts model, service, and API

**Files:**
- Create: `backend/app/modules/accounts/__init__.py`, `models.py`, `schemas.py`, `service.py`, `router.py`
- Modify: `backend/app/main.py`
- Test: `backend/tests/modules/accounts/test_accounts.py`

**Interfaces:**
- Consumes: `Base`, `DomainError` family, `current_member`.
- Produces:
  - `Account` (id, name, type, opening_balance, sort_order, archived, created_at)
  - `service.list_accounts(s, include_archived=False) -> list[Account]`
  - `service.create_account(s, name, type, opening_balance, sort_order=0) -> Account`
  - `service.update_account(s, account_id, **fields) -> Account`
  - `service.archive_account(s, account_id) -> Account`
  - `service.get_account(s, account_id) -> Account`
  - `service.account_balance(s, account_id) -> int`
  - routes `GET/POST /api/accounts`, `PATCH /api/accounts/{id}`, `POST /api/accounts/{id}/archive`

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/modules/accounts/test_accounts.py
import pytest

from app.core.errors import NotFound, ValidationFailed
from app.modules.accounts import service
from app.modules.accounts.models import Account


def test_create_and_list(db):
    with db.session() as s:
        service.create_account(s, name="BCA", type="bank", opening_balance=1_000_000)
        service.create_account(s, name="Tunai", type="cash", opening_balance=0)
    with db.session() as s:
        accounts = service.list_accounts(s)
        assert [a.name for a in accounts] == ["BCA", "Tunai"]
        assert accounts[0].opening_balance == 1_000_000


def test_invalid_type_rejected(db):
    with db.session() as s:
        with pytest.raises(ValidationFailed) as e:
            service.create_account(s, name="X", type="crypto", opening_balance=0)
        assert e.value.code == "invalid_account_type"


def test_archive_hides_from_default_list(db):
    with db.session() as s:
        acc = service.create_account(s, name="Lama", type="bank", opening_balance=0)
        service.archive_account(s, acc.id)
    with db.session() as s:
        assert service.list_accounts(s) == []
        assert len(service.list_accounts(s, include_archived=True)) == 1


def test_missing_account_raises_not_found(db):
    with db.session() as s:
        with pytest.raises(NotFound):
            service.get_account(s, 999)


def test_balance_is_opening_when_no_transactions(db):
    with db.session() as s:
        acc = service.create_account(s, name="BCA", type="bank", opening_balance=250_000)
    with db.session() as s:
        assert service.account_balance(s, acc.id) == 250_000


def test_api_requires_auth(client):
    assert client.get("/api/accounts").status_code == 401


def test_api_lifecycle(client):
    client.post("/api/setup", json={"name": "Budi", "username": "budi",
                                    "password": "supersecret123"})
    r = client.post("/api/accounts", json={"name": "BCA", "type": "bank",
                                           "opening_balance": 1000000})
    assert r.status_code == 200
    acc_id = r.json()["id"]
    assert client.get("/api/accounts").json()[0]["name"] == "BCA"
    assert client.patch(f"/api/accounts/{acc_id}", json={"name": "BCA Utama"}).json()["name"] == "BCA Utama"
    assert client.post(f"/api/accounts/{acc_id}/archive").json()["archived"] is True
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && .venv/bin/pytest tests/modules/accounts/test_accounts.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'app.modules.accounts'`

- [ ] **Step 3: Write `modules/accounts/models.py`**

```python
from datetime import datetime

from sqlalchemy import Boolean, CheckConstraint, DateTime, Integer, String, func
from sqlalchemy.orm import Mapped, mapped_column

from app.core.db import Base

ACCOUNT_TYPES = ("bank", "cash", "wallet", "credit")


class Account(Base):
    __tablename__ = "accounts"
    __table_args__ = (
        CheckConstraint(
            "type IN ('bank','cash','wallet','credit')", name="ck_accounts_type"
        ),
    )

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    name: Mapped[str] = mapped_column(String, nullable=False)
    type: Mapped[str] = mapped_column(String, nullable=False)
    opening_balance: Mapped[int] = mapped_column(Integer, nullable=False, default=0)
    sort_order: Mapped[int] = mapped_column(Integer, nullable=False, default=0)
    archived: Mapped[bool] = mapped_column(Boolean, nullable=False, default=False)
    created_at: Mapped[datetime] = mapped_column(DateTime, server_default=func.now())
```

- [ ] **Step 4: Write `modules/accounts/schemas.py`**

```python
from pydantic import BaseModel, ConfigDict, Field


class AccountOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    name: str
    type: str
    opening_balance: int
    sort_order: int
    archived: bool


class AccountIn(BaseModel):
    name: str = Field(min_length=1, max_length=80)
    type: str
    opening_balance: int = 0


class AccountPatch(BaseModel):
    name: str | None = Field(default=None, min_length=1, max_length=80)
    type: str | None = None
    opening_balance: int | None = None
    sort_order: int | None = None


class AccountWithBalance(AccountOut):
    balance: int
```

- [ ] **Step 5: Write `modules/accounts/service.py`**

```python
from sqlalchemy import func, select

from app.core.errors import NotFound, ValidationFailed
from app.modules.accounts.models import ACCOUNT_TYPES, Account


def _validate_type(account_type: str) -> None:
    if account_type not in ACCOUNT_TYPES:
        raise ValidationFailed(
            "invalid_account_type",
            f"Jenis akun harus salah satu dari: {', '.join(ACCOUNT_TYPES)}",
        )


def get_account(s, account_id: int) -> Account:
    account = s.get(Account, account_id)
    if account is None:
        raise NotFound("account_not_found", "Akun tidak ditemukan", {"id": account_id})
    return account


def list_accounts(s, *, include_archived: bool = False) -> list[Account]:
    q = select(Account).order_by(Account.sort_order, Account.id)
    if not include_archived:
        q = q.where(Account.archived.is_(False))
    return list(s.execute(q).scalars())


def create_account(s, *, name: str, type: str, opening_balance: int = 0,
                   sort_order: int = 0) -> Account:
    _validate_type(type)
    account = Account(name=name.strip(), type=type,
                      opening_balance=int(opening_balance), sort_order=sort_order)
    s.add(account)
    s.flush()
    return account


def update_account(s, account_id: int, **fields) -> Account:
    account = get_account(s, account_id)
    if "type" in fields and fields["type"] is not None:
        _validate_type(fields["type"])
    for key in ("name", "type", "opening_balance", "sort_order"):
        value = fields.get(key)
        if value is not None:
            setattr(account, key, value.strip() if key == "name" else value)
    s.flush()
    return account


def archive_account(s, account_id: int) -> Account:
    account = get_account(s, account_id)
    account.archived = True
    s.flush()
    return account


def account_balance(s, account_id: int) -> int:
    """Balance = opening balance + sum of that account's transactions."""
    from app.modules.transactions.models import Transaction

    account = get_account(s, account_id)
    total = s.execute(
        select(func.coalesce(func.sum(Transaction.amount), 0))
        .where(Transaction.account_id == account.id)
    ).scalar_one()
    return int(account.opening_balance) + int(total)
```

- [ ] **Step 6: Write `modules/accounts/router.py`**

```python
from fastapi import APIRouter, Request

from app.core.deps import current_member
from app.modules.accounts import service
from app.modules.accounts.schemas import AccountIn, AccountOut, AccountPatch, AccountWithBalance

router = APIRouter(prefix="/api/accounts", tags=["accounts"])


def _db(request):
    return request.app.state.db


@router.get("", response_model=list[AccountWithBalance])
def list_accounts(request: Request, include_archived: bool = False):
    current_member(request)
    with _db(request).session() as s:
        return [
            AccountWithBalance(**AccountOut.model_validate(a).model_dump(),
                               balance=service.account_balance(s, a.id))
            for a in service.list_accounts(s, include_archived=include_archived)
        ]


@router.post("", response_model=AccountOut)
def create_account(payload: AccountIn, request: Request):
    current_member(request)
    with _db(request).session() as s:
        return AccountOut.model_validate(
            service.create_account(s, name=payload.name, type=payload.type,
                                   opening_balance=payload.opening_balance))


@router.patch("/{account_id}", response_model=AccountOut)
def update_account(account_id: int, payload: AccountPatch, request: Request):
    current_member(request)
    with _db(request).session() as s:
        return AccountOut.model_validate(
            service.update_account(s, account_id, **payload.model_dump(exclude_unset=True)))


@router.post("/{account_id}/archive", response_model=AccountOut)
def archive_account(account_id: int, request: Request):
    current_member(request)
    with _db(request).session() as s:
        return AccountOut.model_validate(service.archive_account(s, account_id))
```

- [ ] **Step 7: Include the router, generate the migration, run the suite**

Add to `main.py`: `from app.modules.accounts.router import router as accounts_router` and `app.include_router(accounts_router)`.

Run: `cd backend && .venv/bin/python -m alembic revision --autogenerate -m "accounts" && .venv/bin/pytest -v`
Expected: migration created; all passed

- [ ] **Step 8: Commit**

```bash
git add backend/app/modules/accounts backend/app/main.py backend/app/migrations/versions backend/tests/modules/accounts
git commit -m "feat: accounts model, service with computed balance, and api"
```

---

### Task 16: Categories with nesting and archive

**Files:**
- Create: `backend/app/modules/categories/{__init__,models,schemas,service,router}.py`
- Modify: `backend/app/main.py`
- Test: `backend/tests/modules/categories/test_categories.py`

**Interfaces:**
- Consumes: `Base`, `DomainError` family.
- Produces:
  - `Category` (id, name, kind, parent_id, emoji, color, archived)
  - `service.list_categories(s, kind=None, include_archived=False) -> list[Category]`
  - `service.create_category(s, name, kind, emoji, color, parent_id=None) -> Category`
  - `service.update_category(...)`, `service.archive_category(s, id)`, `service.get_category(s, id)`
  - routes `GET/POST /api/categories`, `PATCH /api/categories/{id}`, `POST /api/categories/{id}/archive`

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/modules/categories/test_categories.py
import pytest

from app.core.errors import Conflict, NotFound, ValidationFailed
from app.modules.categories import service


def test_create_flat_and_nested(db):
    with db.session() as s:
        food = service.create_category(s, name="Makanan", kind="expense", emoji="🍚")
        sub = service.create_category(s, name="Warung", kind="expense", emoji="🍜",
                                      parent_id=food.id)
        assert sub.parent_id == food.id
        assert service.list_categories(s, kind="expense")[0].name == "Makanan"


def test_invalid_kind_rejected(db):
    with db.session() as s:
        with pytest.raises(ValidationFailed) as e:
            service.create_category(s, name="X", kind="transfer", emoji="📦")
        assert e.value.code == "invalid_category_kind"


def test_duplicate_name_same_parent_conflicts(db):
    with db.session() as s:
        service.create_category(s, name="Makanan", kind="expense", emoji="🍚")
    with db.session() as s:
        with pytest.raises(Conflict):
            service.create_category(s, name="Makanan", kind="expense", emoji="🍚")


def test_same_name_different_kind_is_allowed(db):
    with db.session() as s:
        service.create_category(s, name="Bonus", kind="income", emoji="💰")
        service.create_category(s, name="Bonus", kind="expense", emoji="🎁")
        assert len(service.list_categories(s)) == 2


def test_parent_must_exist(db):
    with db.session() as s:
        with pytest.raises(NotFound):
            service.create_category(s, name="X", kind="expense", emoji="📦", parent_id=999)


def test_archive(db):
    with db.session() as s:
        cat = service.create_category(s, name="Hiburan", kind="expense", emoji="🎬")
        service.archive_category(s, cat.id)
    with db.session() as s:
        assert service.list_categories(s, kind="expense") == []
        assert len(service.list_categories(s, kind="expense", include_archived=True)) == 1
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && .venv/bin/pytest tests/modules/categories/test_categories.py -v`
Expected: FAIL — module not found

- [ ] **Step 3: Write `modules/categories/models.py`**

```python
from sqlalchemy import Boolean, CheckConstraint, ForeignKey, Integer, String, UniqueConstraint
from sqlalchemy.orm import Mapped, mapped_column

from app.core.db import Base

CATEGORY_KINDS = ("income", "expense")


class Category(Base):
    __tablename__ = "categories"
    __table_args__ = (
        CheckConstraint("kind IN ('income','expense')", name="ck_categories_kind"),
        UniqueConstraint("kind", "name", "parent_id", name="ux_categories_ident"),
    )

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    name: Mapped[str] = mapped_column(String, nullable=False)
    kind: Mapped[str] = mapped_column(String, nullable=False)
    parent_id: Mapped[int | None] = mapped_column(
        ForeignKey("categories.id", ondelete="RESTRICT"), nullable=True)
    emoji: Mapped[str] = mapped_column(String, nullable=False, default="📦")
    color: Mapped[str] = mapped_column(String, nullable=False, default="#C9B6FF")
    archived: Mapped[bool] = mapped_column(Boolean, nullable=False, default=False)
```

`UniqueConstraint` with a nullable `parent_id` treats two `NULL`s as distinct in SQLite, so top-level duplicates would slip through. Use a **functional** unique index instead, replacing the `UniqueConstraint` line:

```python
from sqlalchemy import Index, text

        Index("ux_categories_ident", "kind", "name", text("IFNULL(parent_id, -1)"), unique=True),
```

- [ ] **Step 4: Write `modules/categories/service.py`**

```python
from sqlalchemy import func, select

from app.core.errors import Conflict, NotFound, ValidationFailed
from app.modules.categories.models import CATEGORY_KINDS, Category


def _validate_kind(kind: str) -> None:
    if kind not in CATEGORY_KINDS:
        raise ValidationFailed("invalid_category_kind",
                               "Jenis kategori harus 'income' atau 'expense'")


def get_category(s, category_id: int) -> Category:
    category = s.get(Category, category_id)
    if category is None:
        raise NotFound("category_not_found", "Kategori tidak ditemukan", {"id": category_id})
    return category


def list_categories(s, *, kind: str | None = None,
                    include_archived: bool = False) -> list[Category]:
    q = select(Category).order_by(Category.kind, Category.parent_id, Category.name)
    if kind:
        q = q.where(Category.kind == kind)
    if not include_archived:
        q = q.where(Category.archived.is_(False))
    return list(s.execute(q).scalars())


def _assert_unique(s, kind: str, name: str, parent_id: int | None, exclude_id: int | None = None):
    q = select(Category).where(
        Category.kind == kind,
        func.lower(Category.name) == name.lower(),
        Category.parent_id.is_(parent_id) if parent_id is None else Category.parent_id == parent_id,
    )
    if exclude_id is not None:
        q = q.where(Category.id != exclude_id)
    if s.execute(q).scalars().first() is not None:
        raise Conflict("category_exists", f"Kategori '{name}' sudah ada")


def create_category(s, *, name: str, kind: str, emoji: str = "📦",
                    color: str = "#C9B6FF", parent_id: int | None = None) -> Category:
    _validate_kind(kind)
    if parent_id is not None:
        parent = get_category(s, parent_id)
        if parent.kind != kind:
            raise ValidationFailed("parent_kind_mismatch",
                                   "Induk kategori harus berjenis sama")
    _assert_unique(s, kind, name.strip(), parent_id)
    category = Category(name=name.strip(), kind=kind, parent_id=parent_id,
                        emoji=emoji, color=color)
    s.add(category)
    s.flush()
    return category


def update_category(s, category_id: int, **fields) -> Category:
    category = get_category(s, category_id)
    kind = fields.get("kind") or category.kind
    _validate_kind(kind)
    name = (fields.get("name") or category.name).strip()
    parent_id = fields.get("parent_id", category.parent_id)
    _assert_unique(s, kind, name, parent_id, exclude_id=category_id)
    category.name, category.kind, category.parent_id = name, kind, parent_id
    for key in ("emoji", "color"):
        if fields.get(key) is not None:
            setattr(category, key, fields[key])
    s.flush()
    return category


def archive_category(s, category_id: int) -> Category:
    category = get_category(s, category_id)
    category.archived = True
    s.flush()
    return category
```

- [ ] **Step 5: Write `schemas.py` and `router.py`**

`schemas.py`:

```python
from pydantic import BaseModel, ConfigDict, Field


class CategoryOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    name: str
    kind: str
    parent_id: int | None
    emoji: str
    color: str
    archived: bool


class CategoryIn(BaseModel):
    name: str = Field(min_length=1, max_length=60)
    kind: str
    emoji: str = "📦"
    color: str = "#C9B6FF"
    parent_id: int | None = None


class CategoryPatch(BaseModel):
    name: str | None = None
    kind: str | None = None
    emoji: str | None = None
    color: str | None = None
    parent_id: int | None = None
```

`router.py`:

```python
from fastapi import APIRouter, Request

from app.core.deps import current_member
from app.modules.categories import service
from app.modules.categories.schemas import CategoryIn, CategoryOut, CategoryPatch

router = APIRouter(prefix="/api/categories", tags=["categories"])


def _db(request):
    return request.app.state.db


@router.get("", response_model=list[CategoryOut])
def list_categories(request: Request, kind: str | None = None, include_archived: bool = False):
    current_member(request)
    with _db(request).session() as s:
        return [CategoryOut.model_validate(c)
                for c in service.list_categories(s, kind=kind, include_archived=include_archived)]


@router.post("", response_model=CategoryOut)
def create_category(payload: CategoryIn, request: Request):
    current_member(request)
    with _db(request).session() as s:
        return CategoryOut.model_validate(service.create_category(
            s, name=payload.name, kind=payload.kind, emoji=payload.emoji,
            color=payload.color, parent_id=payload.parent_id))


@router.patch("/{category_id}", response_model=CategoryOut)
def update_category(category_id: int, payload: CategoryPatch, request: Request):
    current_member(request)
    with _db(request).session() as s:
        return CategoryOut.model_validate(
            service.update_category(s, category_id,
                                    **payload.model_dump(exclude_unset=True)))


@router.post("/{category_id}/archive", response_model=CategoryOut)
def archive_category(category_id: int, request: Request):
    current_member(request)
    with _db(request).session() as s:
        return CategoryOut.model_validate(service.archive_category(s, category_id))
```

- [ ] **Step 6: Include the router, migrate, run the suite**

Run: `cd backend && .venv/bin/python -m alembic revision --autogenerate -m "categories" && .venv/bin/pytest -v`
Expected: all passed

- [ ] **Step 7: Commit**

```bash
git add backend/app/modules/categories backend/app/main.py backend/app/migrations/versions backend/tests/modules/categories
git commit -m "feat: categories with nesting, uniqueness, and archive"
```

---

# Phase 4 — Transactions

### Task 17: Transaction model

**Files:**
- Create: `backend/app/modules/transactions/__init__.py`, `models.py`
- Test: `backend/tests/modules/transactions/test_transaction_model.py`

**Interfaces:**
- Consumes: `Base`, `Account`, `Category`, `Member`.
- Produces: `Transaction` (id, account_id, date, amount, description, category_id, paid_by, for_whom, note, transfer_group_id, import_id, import_hash, created_at, updated_at).

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/modules/transactions/test_transaction_model.py
import pytest
from sqlalchemy import text
from sqlalchemy.exc import IntegrityError

from app.modules.accounts import service as accounts
from app.modules.transactions.models import Transaction


def test_account_foreign_key_is_enforced(db):
    """PRAGMA foreign_keys=ON must actually be on, or this test passes silently."""
    with db.session() as s:
        s.add(Transaction(account_id=999, date="2026-02-01", amount=-1, description="ghost"))
        with pytest.raises(IntegrityError):
            s.flush()


def test_partial_unique_index_allows_many_null_hashes(db):
    with db.session() as s:
        acc = accounts.create_account(s, name="BCA", type="bank", opening_balance=0)
        for i in range(3):
            s.add(Transaction(account_id=acc.id, date="2026-02-01", amount=-100,
                              description=f"manual {i}", import_hash=None))
        s.flush()
        assert s.execute(text("select count(*) from transactions")).scalar() == 3


def test_duplicate_import_hash_is_rejected_but_two_distinct_rows_coexist(db):
    with db.session() as s:
        acc = accounts.create_account(s, name="BCA", type="bank", opening_balance=0)
        s.add(Transaction(account_id=acc.id, date="2026-02-01", amount=-100,
                          description="same", import_hash="abc"))
        s.flush()
        s.add(Transaction(account_id=acc.id, date="2026-02-01", amount=-100,
                          description="different", import_hash="def"))
        s.flush()          # allowed: distinct hashes
        s.add(Transaction(account_id=acc.id, date="2026-02-01", amount=-100,
                          description="same", import_hash="abc"))
        with pytest.raises(IntegrityError):
            s.flush()      # rejected: same hash
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && .venv/bin/pytest tests/modules/transactions/test_transaction_model.py -v`
Expected: FAIL — module not found

- [ ] **Step 3: Write `modules/transactions/models.py`**

```python
from datetime import date as date_type, datetime

from sqlalchemy import (CheckConstraint, Date, DateTime, ForeignKey, Index, Integer,
                        String, Text, func, text)
from sqlalchemy.orm import Mapped, mapped_column

from app.core.db import Base


class Transaction(Base):
    __tablename__ = "transactions"
    __table_args__ = (
        CheckConstraint("amount <> 0", name="ck_transactions_amount_nonzero"),
        Index("ix_tx_account_date", "account_id", "date"),
        Index("ix_tx_date", "date"),
        Index("ix_tx_category", "category_id"),
        Index("ix_tx_transfer", "transfer_group_id"),
        Index("ux_tx_import_hash", "import_hash", unique=True,
              sqlite_where=text("import_hash IS NOT NULL")),
    )

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    account_id: Mapped[int] = mapped_column(ForeignKey("accounts.id"), nullable=False)
    date: Mapped[date_type] = mapped_column(Date, nullable=False)
    amount: Mapped[int] = mapped_column(Integer, nullable=False)
    description: Mapped[str] = mapped_column(String, nullable=False, default="")
    category_id: Mapped[int | None] = mapped_column(ForeignKey("categories.id"), nullable=True)
    paid_by: Mapped[int | None] = mapped_column(ForeignKey("members.id"), nullable=True)
    for_whom: Mapped[int | None] = mapped_column(ForeignKey("members.id"), nullable=True)
    note: Mapped[str | None] = mapped_column(Text, nullable=True)
    transfer_group_id: Mapped[str | None] = mapped_column(String, nullable=True)
    import_id: Mapped[int | None] = mapped_column(
        ForeignKey("imports.id", ondelete="CASCADE"), nullable=True)
    import_hash: Mapped[str | None] = mapped_column(String, nullable=True)
    created_at: Mapped[datetime] = mapped_column(DateTime, server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(
        DateTime, server_default=func.now(), onupdate=func.now())
```

Note the `import_id` foreign key targets `imports`, so Step 4 creates the import batch models first. The migration in Step 5 creates `transactions`, `import_profiles`, and `imports` together.

- [ ] **Step 4: Create the import batch models (dependency for the FK)**

Write `backend/app/modules/imports/__init__.py` (empty) and `backend/app/modules/imports/models.py`:

```python
from datetime import datetime

from sqlalchemy import (Boolean, CheckConstraint, DateTime, ForeignKey, Integer,
                        String, Text, func)
from sqlalchemy.orm import Mapped, mapped_column

from app.core.db import Base

AMOUNT_MODES = ("signed", "debit_credit", "amount_type", "flip_outflow")


class ImportProfile(Base):
    __tablename__ = "import_profiles"
    __table_args__ = (
        CheckConstraint(
            "amount_mode IN ('signed','debit_credit','amount_type','flip_outflow')",
            name="ck_import_profiles_amount_mode"),
    )

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    name: Mapped[str] = mapped_column(String, nullable=False)
    bank_name: Mapped[str | None] = mapped_column(String, nullable=True)
    delimiter: Mapped[str] = mapped_column(String, nullable=False, default=",")
    encoding: Mapped[str] = mapped_column(String, nullable=False, default="utf-8")
    has_header: Mapped[bool] = mapped_column(Boolean, nullable=False, default=True)
    skip_rows: Mapped[int] = mapped_column(Integer, nullable=False, default=0)
    date_format: Mapped[str | None] = mapped_column(String, nullable=True)
    amount_mode: Mapped[str] = mapped_column(String, nullable=False)
    column_map: Mapped[str] = mapped_column(Text, nullable=False)   # JSON
    created_at: Mapped[datetime] = mapped_column(DateTime, server_default=func.now())


class Import(Base):
    __tablename__ = "imports"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    profile_id: Mapped[int | None] = mapped_column(ForeignKey("import_profiles.id"), nullable=True)
    account_id: Mapped[int] = mapped_column(ForeignKey("accounts.id"), nullable=False)
    filename: Mapped[str] = mapped_column(String, nullable=False)
    uploaded_by: Mapped[int] = mapped_column(ForeignKey("members.id"), nullable=False)
    uploaded_at: Mapped[datetime] = mapped_column(DateTime, server_default=func.now())
    status: Mapped[str] = mapped_column(String, nullable=False, default="committed")
    row_count: Mapped[int] = mapped_column(Integer, nullable=False, default=0)
    imported_count: Mapped[int] = mapped_column(Integer, nullable=False, default=0)
    duplicate_count: Mapped[int] = mapped_column(Integer, nullable=False, default=0)
    possible_duplicate_count: Mapped[int] = mapped_column(Integer, nullable=False, default=0)
    error_count: Mapped[int] = mapped_column(Integer, nullable=False, default=0)
```

Add `imports` and `import_profiles` to the `create_all` and Alembic `env.py` import lines (they already list `imports`).

- [ ] **Step 5: Generate the migration and run the tests**

Run: `cd backend && .venv/bin/python -m alembic revision --autogenerate -m "transactions, import batches" && .venv/bin/pytest tests/modules/transactions/test_transaction_model.py -v`
Expected: migration creates `transactions`, `import_profiles`, `imports`; 3 passed

- [ ] **Step 6: Commit**

```bash
git add backend/app/modules/transactions backend/app/modules/imports backend/app/migrations/versions backend/tests/modules/transactions
git commit -m "feat: transaction model with fk enforcement and partial unique import hash"
```

---

### Task 18: Transaction service — CRUD and validation

**Files:**
- Create: `backend/app/modules/transactions/service.py`, `schemas.py`
- Test: `backend/tests/modules/transactions/test_transactions.py`

**Interfaces:**
- Consumes: `Transaction`, `accounts.service`, `categories.service`, `auth.service`.
- Produces:
  - `create_transaction(s, *, account_id, date, amount, description, category_id=None, paid_by=None, for_whom=None, note=None) -> Transaction`
  - `get_transaction(s, tx_id) -> Transaction`
  - `update_transaction(s, tx_id, **fields) -> Transaction`
  - `delete_transaction(s, tx_id) -> None`
  - `list_transactions(s, **filters) -> tuple[list[Transaction], int]`

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/modules/transactions/test_transactions.py
from datetime import date

import pytest

from app.core.errors import NotFound, ValidationFailed
from app.modules.accounts import service as accounts
from app.modules.auth import service as auth
from app.modules.categories import service as categories
from app.modules.transactions import service as txs


@pytest.fixture
def world(db):
    with db.session() as s:
        bca = accounts.create_account(s, name="BCA", type="bank", opening_balance=1_000_000)
        cash = accounts.create_account(s, name="Tunai", type="cash", opening_balance=0)
        food = categories.create_category(s, name="Makanan", kind="expense", emoji="🍚")
        budi = auth.create_member(s, name="Budi", username="budi",
                                  password="supersecret123", avatar_color="#EF476F")
        return {"bca": bca.id, "cash": cash.id, "food": food.id, "budi": budi.id}


def test_create_updates_balance(db, world):
    with db.session() as s:
        txs.create_transaction(s, account_id=world["bca"], date=date(2026, 2, 1),
                               amount=-186_000, description="Superindo",
                               category_id=world["food"], paid_by=world["budi"])
    with db.session() as s:
        assert accounts.account_balance(s, world["bca"]) == 1_000_000 - 186_000


def test_zero_amount_rejected(db, world):
    with db.session() as s:
        with pytest.raises(ValidationFailed) as e:
            txs.create_transaction(s, account_id=world["bca"], date=date(2026, 2, 1),
                                   amount=0, description="nothing")
        assert e.value.code == "amount_required"


def test_empty_description_allowed_but_blank_normalised(db, world):
    with db.session() as s:
        tx = txs.create_transaction(s, account_id=world["bca"], date=date(2026, 2, 1),
                                    amount=-1, description="   ")
        assert tx.description == ""


def test_unknown_account_rejected(db, world):
    with db.session() as s:
        with pytest.raises(NotFound):
            txs.create_transaction(s, account_id=999, date=date(2026, 2, 1),
                                   amount=-1, description="x")


def test_category_kind_must_match_amount_sign(db, world):
    with db.session() as s:
        with pytest.raises(ValidationFailed) as e:
            txs.create_transaction(s, account_id=world["bca"], date=date(2026, 2, 1),
                                   amount=+1_000, description="refund",
                                   category_id=world["food"])
        assert e.value.code == "category_kind_mismatch"


def test_update_and_delete(db, world):
    with db.session() as s:
        tx = txs.create_transaction(s, account_id=world["bca"], date=date(2026, 2, 1),
                                    amount=-10_000, description="a")
        tx_id = tx.id
    with db.session() as s:
        txs.update_transaction(s, tx_id, amount=-20_000, description="b")
    with db.session() as s:
        assert txs.get_transaction(s, tx_id).amount == -20_000
        txs.delete_transaction(s, tx_id)
    with db.session() as s:
        with pytest.raises(NotFound):
            txs.get_transaction(s, tx_id)


def test_filters_and_pagination(db, world):
    with db.session() as s:
        for i in range(5):
            txs.create_transaction(s, account_id=world["bca"], date=date(2026, 2, 1 + i),
                                   amount=-1000 * (i + 1), description=f"row {i}",
                                   category_id=world["food"] if i % 2 == 0 else None)
    with db.session() as s:
        rows, total = txs.list_transactions(s, limit=2, offset=0)
        assert total == 5 and len(rows) == 2
        assert rows[0].date > rows[1].date            # newest first
        only_food, total_food = txs.list_transactions(s, category_id=world["food"])
        assert total_food == 3
        uncat, total_uncat = txs.list_transactions(s, uncategorized=True)
        assert total_uncat == 2
        ranged, total_ranged = txs.list_transactions(s, date_from=date(2026, 2, 3),
                                                     date_to=date(2026, 2, 4))
        assert total_ranged == 2
        found, total_found = txs.list_transactions(s, search="row 3")
        assert total_found == 1
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && .venv/bin/pytest tests/modules/transactions/test_transactions.py -v`
Expected: FAIL — module not found

- [ ] **Step 3: Write `modules/transactions/service.py`**

```python
from datetime import date as date_type

from sqlalchemy import func, or_, select

from app.core.errors import NotFound, ValidationFailed
from app.core.money import parse_rupiah
from app.modules.transactions.models import Transaction


def _norm_description(value: str | None) -> str:
    return " ".join((value or "").split())


def _check_category_kind(s, category_id: int | None, amount: int) -> None:
    if category_id is None:
        return
    from app.modules.categories import service as categories

    category = categories.get_category(s, category_id)
    if amount < 0 and category.kind != "expense":
        raise ValidationFailed("category_kind_mismatch",
                               "Kategori pemasukan tidak bisa dipakai untuk pengeluaran")
    if amount > 0 and category.kind != "income":
        raise ValidationFailed("category_kind_mismatch",
                               "Kategori pengeluaran tidak bisa dipakai untuk pemasukan")


def get_transaction(s, tx_id: int) -> Transaction:
    tx = s.get(Transaction, tx_id)
    if tx is None:
        raise NotFound("transaction_not_found", "Transaksi tidak ditemukan", {"id": tx_id})
    return tx


def _validate_refs(s, account_id: int, paid_by: int | None, for_whom: int | None) -> None:
    from app.modules.accounts import service as accounts
    from app.modules.auth import service as auth

    accounts.get_account(s, account_id)
    for member_id in (paid_by, for_whom):
        if member_id is not None:
            auth.get_member(s, member_id)


def create_transaction(s, *, account_id: int, date, amount: int, description: str = "",
                       category_id: int | None = None, paid_by: int | None = None,
                       for_whom: int | None = None, note: str | None = None,
                       import_id: int | None = None,
                       import_hash: str | None = None,
                       transfer_group_id: str | None = None) -> Transaction:
    amount = parse_rupiah(amount)
    if amount == 0:
        raise ValidationFailed("amount_required", "Jumlah tidak boleh nol")
    _validate_refs(s, account_id, paid_by, for_whom)
    _check_category_kind(s, category_id, amount)
    tx = Transaction(
        account_id=account_id,
        date=date if isinstance(date, date_type) else date_type.fromisoformat(str(date)),
        amount=amount,
        description=_norm_description(description),
        category_id=category_id, paid_by=paid_by, for_whom=for_whom,
        note=note, import_id=import_id, import_hash=import_hash,
        transfer_group_id=transfer_group_id,
    )
    s.add(tx)
    s.flush()
    return tx


def update_transaction(s, tx_id: int, **fields) -> Transaction:
    tx = get_transaction(s, tx_id)
    new_amount = parse_rupiah(fields["amount"]) if fields.get("amount") is not None else tx.amount
    if new_amount == 0:
        raise ValidationFailed("amount_required", "Jumlah tidak boleh nol")
    new_category = fields.get("category_id", tx.category_id)
    _check_category_kind(s, new_category, new_amount)
    if fields.get("account_id") is not None:
        _validate_refs(s, fields["account_id"], None, None)
        tx.account_id = fields["account_id"]
    tx.amount = new_amount
    tx.category_id = new_category
    for key in ("description", "note", "paid_by", "for_whom"):
        if key in fields:
            setattr(tx, key, _norm_description(fields[key]) if key == "description" else fields[key])
    if fields.get("date") is not None:
        value = fields["date"]
        tx.date = value if isinstance(value, date_type) else date_type.fromisoformat(str(value))
    s.flush()
    return tx


def delete_transaction(s, tx_id: int) -> None:
    tx = get_transaction(s, tx_id)
    if tx.transfer_group_id:
        for leg in s.execute(select(Transaction).where(
                Transaction.transfer_group_id == tx.transfer_group_id)).scalars():
            s.delete(leg)
    else:
        s.delete(tx)
    s.flush()


def list_transactions(s, *, account_id: int | None = None, category_id: int | None = None,
                      member_id: int | None = None, date_from=None, date_to=None,
                      uncategorized: bool = False, search: str | None = None,
                      limit: int = 50, offset: int = 0) -> tuple[list[Transaction], int]:
    conditions = []
    if account_id is not None:
        conditions.append(Transaction.account_id == account_id)
    if category_id is not None:
        conditions.append(Transaction.category_id == category_id)
    if uncategorized:
        conditions.append(Transaction.category_id.is_(None))
    if member_id is not None:
        conditions.append(or_(Transaction.paid_by == member_id,
                              Transaction.for_whom == member_id))
    if date_from is not None:
        conditions.append(Transaction.date >= date_from)
    if date_to is not None:
        conditions.append(Transaction.date <= date_to)
    if search:
        conditions.append(Transaction.description.ilike(f"%{search}%"))

    base = select(Transaction)
    for c in conditions:
        base = base.where(c)

    total = s.execute(
        select(func.count()).select_from(base.subquery())
    ).scalar_one()
    rows = list(s.execute(
        base.order_by(Transaction.date.desc(), Transaction.id.desc())
        .limit(limit).offset(offset)
    ).scalars())
    return rows, int(total)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && .venv/bin/pytest tests/modules/transactions/test_transactions.py -v`
Expected: 7 passed

- [ ] **Step 5: Commit**

```bash
git add backend/app/modules/transactions/service.py backend/tests/modules/transactions/test_transactions.py
git commit -m "feat: transaction service with validation, filters, and pagination"
```

---

### Task 19: Transfers as paired legs

**Files:**
- Modify: `backend/app/modules/transactions/service.py` (`create_transfer`)
- Test: `backend/tests/modules/transactions/test_transfers.py`

**Interfaces:**
- Consumes: `create_transaction`, `delete_transaction`.
- Produces: `create_transfer(s, *, from_account_id, to_account_id, date, amount, description="", paid_by=None, note=None) -> tuple[Transaction, Transaction]`. `amount` is a **positive magnitude**; validation rejects same-account and non-positive amounts.

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/modules/transactions/test_transfers.py
from datetime import date

import pytest

from app.core.errors import ValidationFailed
from app.modules.accounts import service as accounts
from app.modules.transactions import service as txs


@pytest.fixture
def pair(db):
    with db.session() as s:
        a = accounts.create_account(s, name="BCA", type="bank", opening_balance=1_000_000)
        b = accounts.create_account(s, name="Tunai", type="cash", opening_balance=0)
        return {"a": a.id, "b": b.id}


def test_transfer_creates_two_legs_sharing_a_group(db, pair):
    with db.session() as s:
        out_leg, in_leg = txs.create_transfer(s, from_account_id=pair["a"],
                                              to_account_id=pair["b"],
                                              date=date(2026, 2, 5), amount=500_000,
                                              description="Tarik tunai")
        assert out_leg.transfer_group_id == in_leg.transfer_group_id
        assert out_leg.transfer_group_id is not None
        assert (out_leg.amount, in_leg.amount) == (-500_000, 500_000)
        assert out_leg.account_id == pair["a"] and in_leg.account_id == pair["b"]


def test_transfer_does_not_change_net_worth(db, pair):
    """Review Focus: an internal transfer must net to zero across the household."""
    with db.session() as s:
        before = sum(accounts.account_balance(s, i) for i in (pair["a"], pair["b"]))
    with db.session() as s:
        txs.create_transfer(s, from_account_id=pair["a"], to_account_id=pair["b"],
                            date=date(2026, 2, 5), amount=500_000, description="Tarik tunai")
    with db.session() as s:
        after = sum(accounts.account_balance(s, i) for i in (pair["a"], pair["b"]))
        assert after == before == 1_000_000


def test_accounts_move_correctly(db, pair):
    with db.session() as s:
        txs.create_transfer(s, from_account_id=pair["a"], to_account_id=pair["b"],
                            date=date(2026, 2, 5), amount=500_000, description="x")
    with db.session() as s:
        assert accounts.account_balance(s, pair["a"]) == 500_000
        assert accounts.account_balance(s, pair["b"]) == 500_000


def test_same_account_transfer_rejected(db, pair):
    with db.session() as s:
        with pytest.raises(ValidationFailed) as e:
            txs.create_transfer(s, from_account_id=pair["a"], to_account_id=pair["a"],
                                date=date(2026, 2, 5), amount=100, description="x")
        assert e.value.code == "transfer_same_account"


def test_zero_or_negative_transfer_rejected(db, pair):
    with db.session() as s:
        for bad in (0, -500):
            with pytest.raises(ValidationFailed):
                txs.create_transfer(s, from_account_id=pair["a"], to_account_id=pair["b"],
                                    date=date(2026, 2, 5), amount=bad, description="x")


def test_deleting_one_leg_deletes_both(db, pair):
    with db.session() as s:
        out_leg, _ = txs.create_transfer(s, from_account_id=pair["a"], to_account_id=pair["b"],
                                         date=date(2026, 2, 5), amount=500_000, description="x")
        out_id = out_leg.id
    with db.session() as s:
        txs.delete_transaction(s, out_id)
    with db.session() as s:
        rows, total = txs.list_transactions(s, limit=100)
        assert total == 0
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && .venv/bin/pytest tests/modules/transactions/test_transfers.py -v`
Expected: FAIL — `AttributeError: module ... has no attribute 'create_transfer'`

- [ ] **Step 3: Add `create_transfer` to `modules/transactions/service.py`**

```python
import uuid


def create_transfer(s, *, from_account_id: int, to_account_id: int, date, amount: int,
                    description: str = "", paid_by: int | None = None,
                    note: str | None = None) -> tuple[Transaction, Transaction]:
    """Create a transfer as two transaction legs sharing one group id, atomically."""
    amount = parse_rupiah(amount)
    if amount <= 0:
        raise ValidationFailed("transfer_amount_invalid", "Jumlah transfer harus lebih dari nol")
    if from_account_id == to_account_id:
        raise ValidationFailed("transfer_same_account",
                               "Akun asal dan tujuan harus berbeda")

    group = str(uuid.uuid4())
    out_leg = create_transaction(
        s, account_id=from_account_id, date=date, amount=-amount,
        description=_norm_description(description), paid_by=paid_by, note=note,
        transfer_group_id=group)
    in_leg = create_transaction(
        s, account_id=to_account_id, date=date, amount=amount,
        description=_norm_description(description), paid_by=paid_by, note=note,
        transfer_group_id=group)
    return out_leg, in_leg
```

Both legs are created inside the caller's single session/transaction, so a failure on the second leg rolls back the first.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && .venv/bin/pytest tests/modules/transactions/test_transfers.py -v`
Expected: 6 passed

- [ ] **Step 5: Commit**

```bash
git add backend/app/modules/transactions/service.py backend/tests/modules/transactions/test_transfers.py
git commit -m "feat: transfers as paired legs that net to zero"
```

---

### Task 20: Transactions API

**Files:**
- Create: `backend/app/modules/transactions/schemas.py`, `router.py`
- Modify: `backend/app/main.py`
- Test: `backend/tests/modules/transactions/test_transactions_api.py`

**Interfaces:**
- Consumes: `service`, `current_member`.
- Produces routes:
  - `GET /api/transactions` (filters: `account_id`, `category_id`, `member_id`, `date_from`, `date_to`, `uncategorized`, `search`, `limit`, `offset`) → `{"items": [...], "total": int}`
  - `POST /api/transactions` → `TransactionOut`
  - `PATCH /api/transactions/{id}` → `TransactionOut`
  - `DELETE /api/transactions/{id}` → `{"ok": true}`
  - `POST /api/transfers` → `{"out": TransactionOut, "in": TransactionOut}`

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/modules/transactions/test_transactions_api.py

def _setup(client):
    client.post("/api/setup", json={"name": "Budi", "username": "budi",
                                    "password": "supersecret123"})
    acc = client.post("/api/accounts", json={"name": "BCA", "type": "bank",
                                             "opening_balance": 1000000}).json()
    cash = client.post("/api/accounts", json={"name": "Tunai", "type": "cash",
                                              "opening_balance": 0}).json()
    cat = client.post("/api/categories", json={"name": "Makanan", "kind": "expense",
                                               "emoji": "🍚"}).json()
    return acc["id"], cash["id"], cat["id"]


def test_create_list_filter_delete(client):
    acc, cash, cat = _setup(client)
    r = client.post("/api/transactions", json={"account_id": acc, "date": "2026-02-01",
                                               "amount": -186000, "description": "Superindo",
                                               "category_id": cat})
    assert r.status_code == 200
    tx_id = r.json()["id"]

    body = client.get("/api/transactions").json()
    assert body["total"] == 1
    assert body["items"][0]["amount"] == -186000

    assert client.get(f"/api/transactions?category_id={cat}").json()["total"] == 1
    assert client.get("/api/transactions?uncategorized=true").json()["total"] == 0
    assert client.get("/api/transactions?search=super").json()["total"] == 1

    assert client.patch(f"/api/transactions/{tx_id}",
                        json={"amount": -200000}).json()["amount"] == -200000
    assert client.delete(f"/api/transactions/{tx_id}").json() == {"ok": True}
    assert client.get("/api/transactions").json()["total"] == 0


def test_transfer_endpoint(client):
    acc, cash, _ = _setup(client)
    r = client.post("/api/transfers", json={"from_account_id": acc, "to_account_id": cash,
                                            "date": "2026-02-05", "amount": 500000,
                                            "description": "Tarik tunai"})
    assert r.status_code == 200
    assert r.json()["out"]["amount"] == -500000
    assert r.json()["in"]["amount"] == 500000
    balances = {a["name"]: a["balance"] for a in client.get("/api/accounts").json()}
    assert balances == {"BCA": 500000, "Tunai": 500000}


def test_zero_amount_is_rejected(client):
    acc, _, _ = _setup(client)
    r = client.post("/api/transactions", json={"account_id": acc, "date": "2026-02-01",
                                               "amount": 0, "description": "x"})
    assert r.status_code == 422
    assert r.json()["error"]["code"] == "amount_required"


def test_requires_auth(client):
    assert client.get("/api/transactions").status_code == 401
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && .venv/bin/pytest tests/modules/transactions/test_transactions_api.py -v`
Expected: FAIL — 404 on `/api/transactions`

- [ ] **Step 3: Write `modules/transactions/schemas.py`**

```python
from datetime import date as date_type

from pydantic import BaseModel, ConfigDict, Field


class TransactionOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    account_id: int
    date: date_type
    amount: int
    description: str
    category_id: int | None
    paid_by: int | None
    for_whom: int | None
    note: str | None
    transfer_group_id: str | None
    import_id: int | None


class TransactionIn(BaseModel):
    account_id: int
    date: date_type
    amount: int
    description: str = ""
    category_id: int | None = None
    paid_by: int | None = None
    for_whom: int | None = None
    note: str | None = None


class TransactionPatch(BaseModel):
    account_id: int | None = None
    date: date_type | None = None
    amount: int | None = None
    description: str | None = None
    category_id: int | None = None
    paid_by: int | None = None
    for_whom: int | None = None
    note: str | None = None


class TransferIn(BaseModel):
    from_account_id: int
    to_account_id: int
    date: date_type
    amount: int = Field(gt=0)
    description: str = ""
    paid_by: int | None = None
    note: str | None = None


class TransactionPage(BaseModel):
    items: list[TransactionOut]
    total: int
```

- [ ] **Step 4: Write `modules/transactions/router.py`**

```python
from datetime import date as date_type

from fastapi import APIRouter, Query, Request

from app.core.deps import current_member
from app.modules.transactions import service
from app.modules.transactions.schemas import (TransactionIn, TransactionOut, TransactionPage,
                                              TransactionPatch, TransferIn)

router = APIRouter(prefix="/api", tags=["transactions"])


def _db(request):
    return request.app.state.db


@router.get("/transactions", response_model=TransactionPage)
def list_transactions(request: Request, account_id: int | None = None,
                      category_id: int | None = None, member_id: int | None = None,
                      date_from: date_type | None = None, date_to: date_type | None = None,
                      uncategorized: bool = False, search: str | None = None,
                      limit: int = Query(50, ge=1, le=500), offset: int = Query(0, ge=0)):
    current_member(request)
    with _db(request).session() as s:
        rows, total = service.list_transactions(
            s, account_id=account_id, category_id=category_id, member_id=member_id,
            date_from=date_from, date_to=date_to, uncategorized=uncategorized,
            search=search, limit=limit, offset=offset)
        return TransactionPage(items=[TransactionOut.model_validate(r) for r in rows],
                               total=total)


@router.post("/transactions", response_model=TransactionOut)
def create_transaction(payload: TransactionIn, request: Request):
    current_member(request)
    with _db(request).session() as s:
        return TransactionOut.model_validate(
            service.create_transaction(s, **payload.model_dump()))


@router.patch("/transactions/{tx_id}", response_model=TransactionOut)
def update_transaction(tx_id: int, payload: TransactionPatch, request: Request):
    current_member(request)
    with _db(request).session() as s:
        return TransactionOut.model_validate(
            service.update_transaction(s, tx_id, **payload.model_dump(exclude_unset=True)))


@router.delete("/transactions/{tx_id}")
def delete_transaction(tx_id: int, request: Request):
    current_member(request)
    with _db(request).session() as s:
        service.delete_transaction(s, tx_id)
    return {"ok": True}


@router.post("/transfers")
def create_transfer(payload: TransferIn, request: Request):
    current_member(request)
    with _db(request).session() as s:
        out_leg, in_leg = service.create_transfer(s, **payload.model_dump())
        return {"out": TransactionOut.model_validate(out_leg),
                "in": TransactionOut.model_validate(in_leg)}
```

- [ ] **Step 5: Include the router and run the suite**

Add to `main.py`: `from app.modules.transactions.router import router as transactions_router` and `app.include_router(transactions_router)`.

Run: `cd backend && .venv/bin/pytest -v`
Expected: all passed

- [ ] **Step 6: Commit**

```bash
git add backend/app/modules/transactions/schemas.py backend/app/modules/transactions/router.py backend/app/main.py backend/tests/modules/transactions/test_transactions_api.py
git commit -m "feat: transactions and transfers api"
```

---

# Phase 5 — CSV import

> **Before starting this phase:** the ledger must be trustworthy. Import writes bulk data and its bugs are silent (double-counting). Do not start until Phases 3–4 are green.

### Task 21: CSV sniffing

**Files:**
- Create: `backend/app/modules/imports/service.py` (created here, extended by Tasks 22–26)
- Test: `backend/tests/modules/imports/test_sniff.py`

**Interfaces:**
- Consumes: `ValidationFailed`.
- Produces:
  - `Sniffed(encoding: str, delimiter: str, has_header: bool, header: list[str] | None, rows: list[list[str]])`
  - `sniff_csv(raw: bytes, *, max_rows: int, forced: dict | None = None) -> Sniffed` — `forced` may override `encoding`, `delimiter`, `has_header`, `skip_rows`
  - `decode_csv(raw: bytes, encoding: str) -> str`

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/modules/imports/test_sniff.py
import pytest

from app.core.errors import ValidationFailed
from app.modules.imports.service import sniff_csv

PLAIN = b"Tanggal,Keterangan,Jumlah\n01/02/2026,Superindo,-186000\n02/02/2026,Gaji,4200000\n"


def test_detects_comma_delimiter_header_and_rows():
    s = sniff_csv(PLAIN, max_rows=1000)
    assert s.delimiter == ","
    assert s.has_header is True
    assert s.header == ["Tanggal", "Keterangan", "Jumlah"]
    assert len(s.rows) == 2
    assert s.rows[0][1] == "Superindo"


def test_detects_semicolon_and_tab():
    semi = b"a;b;c\n1;2;3\n"
    tab = b"a\tb\tc\n1\t2\t3\n"
    assert sniff_csv(semi, max_rows=100).delimiter == ";"
    assert sniff_csv(tab, max_rows=100).delimiter == "\t"


def test_utf8_bom_is_stripped_not_left_in_the_first_header():
    """Review Focus: a BOM must not corrupt the first column name."""
    raw = b"\xef\xbb\xbfTanggal,Keterangan,Jumlah\n01/02/2026,Superindo,-186000\n"
    s = sniff_csv(raw, max_rows=100)
    assert s.header[0] == "Tanggal"
    assert not s.header[0].startswith("\ufeff")


def test_cp1252_high_bytes_decode_without_mojibake():
    """Review Focus: non-UTF-8 encodings must decode to the right characters."""
    raw = "Tanggal,Keterangan\n01/02/2026,Café Wörk\n".encode("cp1252")
    s = sniff_csv(raw, max_rows=100)
    assert s.rows[0][1] == "Café Wörk"


def test_empty_file_is_rejected():
    """Review Focus: degenerate input fails clearly instead of importing nothing silently."""
    with pytest.raises(ValidationFailed) as e:
        sniff_csv(b"", max_rows=100)
    assert e.value.code == "empty_file"

    with pytest.raises(ValidationFailed) as e:
        sniff_csv(b"Tanggal,Keterangan,Jumlah\n", max_rows=100)
    assert e.value.code == "no_data_rows"


def test_binary_file_is_rejected():
    with pytest.raises(ValidationFailed) as e:
        sniff_csv(b"%PDF-1.7\x00\x01\x02binary", max_rows=100)
    assert e.value.code == "binary_file"


def test_row_cap_is_enforced():
    raw = b"a,b\n" + b"1,2\n" * 50
    with pytest.raises(ValidationFailed) as e:
        sniff_csv(raw, max_rows=10)
    assert e.value.code == "too_many_rows"


def test_forced_overrides_win_over_detection():
    raw = b"a;b;c\n1;2;3\n"
    assert sniff_csv(raw, max_rows=100, forced={"delimiter": "\t"}).delimiter == "\t"
    assert sniff_csv(raw, max_rows=100, forced={"has_header": False}).has_header is False
    skipped = sniff_csv(b"junk,junk\na,b\n1,2\n", max_rows=100,
                        forced={"has_header": True, "skip_rows": 1})
    assert skipped.header == ["a", "b"]
    assert skipped.rows == [["1", "2"]]


def test_forced_encoding_is_honoured():
    raw = "Keterangan\nCafé\n".encode("latin-1")
    assert sniff_csv(raw, max_rows=100, forced={"encoding": "latin-1"}).rows[0][0] == "Café"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && .venv/bin/pytest tests/modules/imports/test_sniff.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'app.modules.imports.service'`

- [ ] **Step 3: Write `modules/imports/service.py`**

```python
import csv
import io
from dataclasses import dataclass

from charset_normalizer import from_bytes

from app.core.errors import ValidationFailed

CANDIDATE_DELIMITERS = ",;\t|"


@dataclass
class Sniffed:
    encoding: str
    delimiter: str
    has_header: bool
    header: list[str] | None
    rows: list[list[str]]


def decode_csv(raw: bytes, encoding: str) -> str:
    try:
        return raw.decode(encoding)
    except (UnicodeDecodeError, LookupError):
        raise ValidationFailed("encoding_failed",
                               f"Gagal membaca berkas dengan encoding {encoding}")


def _detect_encoding(raw: bytes) -> str:
    if raw.startswith(b"\xef\xbb\xbf"):
        return "utf-8-sig"
    best = from_bytes(raw).best()
    if best is None:
        return "utf-8"
    return best.encoding or "utf-8"


def _detect_delimiter(sample: str) -> str:
    counts = {d: sample.count(d) for d in CANDIDATE_DELIMITERS}
    best = max(counts, key=counts.get)
    if counts[best] == 0:
        raise ValidationFailed("no_delimiter_detected",
                               "Tidak menemukan pemisah kolom pada berkas CSV")
    return best


def _looks_like_header(first: list[str], second: list[str] | None) -> bool:
    if second is None:
        return True
    first_numeric = sum(1 for c in first if c.strip().replace("-", "").replace(".", "").isdigit())
    return first_numeric < len(first)


def sniff_csv(raw: bytes, *, max_rows: int, forced: dict | None = None) -> Sniffed:
    """Detect encoding, delimiter and header. `forced` overrides any detection so a
    user correction from the wizard is actually honoured."""
    forced = forced or {}
    if not raw.strip():
        raise ValidationFailed("empty_file", "Berkas kosong")
    if b"\x00" in raw[:4096] or raw[:5] == b"%PDF-":
        raise ValidationFailed("binary_file",
                               "Berkas tidak didukung. Harap unggah berkas CSV")

    encoding = forced.get("encoding") or _detect_encoding(raw)
    text = decode_csv(raw, encoding)
    if text.startswith("\ufeff"):
        text = text[1:]

    delimiter = forced.get("delimiter") or _detect_delimiter(text[:8192])

    reader = csv.reader(io.StringIO(text), delimiter=delimiter)
    all_rows = [row for row in reader if any(cell.strip() for cell in row)]
    if not all_rows:
        raise ValidationFailed("empty_file", "Berkas kosong")
    if len(all_rows) > max_rows:
        raise ValidationFailed("too_many_rows",
                               f"Berkas melebihi batas {max_rows} baris")

    has_header = forced.get("has_header")
    if has_header is None:
        has_header = _looks_like_header(all_rows[0], all_rows[1] if len(all_rows) > 1 else None)

    skip_rows = int(forced.get("skip_rows") or 0)
    header = all_rows[0] if has_header else None
    rows = all_rows[1:] if has_header else all_rows
    rows = rows[skip_rows:]
    if not rows:
        raise ValidationFailed("no_data_rows", "Berkas tidak berisi baris data")
    return Sniffed(encoding=encoding, delimiter=delimiter, has_header=has_header,
                   header=header, rows=rows)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && .venv/bin/pytest tests/modules/imports/test_sniff.py -v`
Expected: 9 passed

- [ ] **Step 5: Commit**

```bash
git add backend/app/modules/imports/service.py backend/tests/modules/imports
git commit -m "feat: csv sniffing with encoding, delimiter, bom, and degenerate-input handling"
```

---

### Task 22: Amount extraction for all four modes

**Files:**
- Modify: `backend/app/modules/imports/service.py`
- Test: `backend/tests/modules/imports/test_amounts.py`

**Interfaces:**
- Consumes: `parse_rupiah`, `MoneyParseError`, `Sniffed`.
- Produces: `extract_amount(row: list[str], column_map: dict, amount_mode: str) -> int` (raises `ValidationFailed` with code `amount_parse_failed`).

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/modules/imports/test_amounts.py
import pytest

from app.core.errors import ValidationFailed
from app.modules.imports.service import extract_amount


def test_signed_mode():
    cm = {"amount": 2}
    assert extract_amount(["01/02/2026", "Superindo", "-186000"], cm, "signed") == -186000
    assert extract_amount(["01/02/2026", "Gaji", "4.200.000"], cm, "signed") == 4200000


def test_debit_credit_mode_uses_whichever_column_has_a_value():
    cm = {"debit": 3, "credit": 4}
    assert extract_amount(["a", "b", "c", "45.000", ""], cm, "debit_credit") == -45000
    assert extract_amount(["a", "b", "c", "", "45.000"], cm, "debit_credit") == 45000


def test_debit_credit_mode_rejects_a_row_with_both_filled():
    cm = {"debit": 3, "credit": 4}
    with pytest.raises(ValidationFailed) as e:
        extract_amount(["a", "b", "c", "45.000", "45.000"], cm, "debit_credit")
    assert e.value.code == "amount_parse_failed"


def test_debit_credit_mode_rejects_a_row_with_neither_filled():
    cm = {"debit": 3, "credit": 4}
    with pytest.raises(ValidationFailed):
        extract_amount(["a", "b", "c", "", ""], cm, "debit_credit")


def test_amount_type_mode_reads_the_type_column():
    cm = {"amount": 2, "type": 3}
    assert extract_amount(["a", "b", "45.000", "DB"], cm, "amount_type") == -45000
    assert extract_amount(["a", "b", "45.000", "CR"], cm, "amount_type") == 45000
    assert extract_amount(["a", "b", "45.000", "Debit"], cm, "amount_type") == -45000


def test_flip_outflow_mode_treats_positive_as_spending():
    cm = {"amount": 2}
    assert extract_amount(["a", "b", "186.000"], cm, "flip_outflow") == -186000
    assert extract_amount(["a", "b", "-186.000"], cm, "flip_outflow") == -186000


@pytest.mark.parametrize("bad", ["", "abc", "Rp", "12.3.4.5x", "-"])
def test_hostile_values_raise_a_row_error_not_a_crash(bad):
    """Review Focus: unparseable amounts become row errors, never unhandled exceptions."""
    with pytest.raises(ValidationFailed) as e:
        extract_amount(["a", "b", bad], {"amount": 2}, "signed")
    assert e.value.code == "amount_parse_failed"


@pytest.mark.parametrize("raw,expected", [
    ("(1.234,56)", -1234),
    ("Rp 1.234.567,00", 1234567),
    ("1.000-", -1000),
    ("45.000 DR", -45000),
])
def test_indonesian_amount_formats_survive_the_full_path(raw, expected):
    """Review Focus: the extraction layer must not undo what the money parser handles."""
    assert extract_amount(["a", "b", raw], {"amount": 2}, "signed") == expected


def test_missing_column_index_is_a_configuration_error():
    with pytest.raises(ValidationFailed) as e:
        extract_amount(["a", "b"], {"amount": 5}, "signed")
    assert e.value.code == "column_out_of_range"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && .venv/bin/pytest tests/modules/imports/test_amounts.py -v`
Expected: FAIL — `ImportError: cannot import name 'extract_amount'`

- [ ] **Step 3: Add `extract_amount` to `modules/imports/service.py`**

```python
from app.core.money import MoneyParseError, parse_rupiah

DEBIT_TOKENS = ("d", "db", "dr", "debit", "keluar")


def _cell(row: list[str], column_map: dict, key: str) -> str | None:
    index = column_map.get(key)
    if index is None:
        return None
    if index >= len(row):
        raise ValidationFailed("column_out_of_range",
                               f"Kolom ke-{index} tidak ada pada baris ini")
    return row[index]


def _parse_or_row_error(value, *, mode: str) -> int:
    try:
        return parse_rupiah(value)
    except MoneyParseError:
        raise ValidationFailed("amount_parse_failed",
                               f"Jumlah tidak dikenali: {value!r} ({mode})")


def extract_amount(row: list[str], column_map: dict, amount_mode: str) -> int:
    if amount_mode == "signed":
        return _parse_or_row_error(_cell(row, column_map, "amount"), mode=amount_mode)

    if amount_mode == "flip_outflow":
        value = _parse_or_row_error(_cell(row, column_map, "amount"), mode=amount_mode)
        return -abs(value) if value > 0 else value

    if amount_mode == "debit_credit":
        debit = (_cell(row, column_map, "debit") or "").strip()
        credit = (_cell(row, column_map, "credit") or "").strip()
        if bool(debit) == bool(credit):
            raise ValidationFailed(
                "amount_parse_failed",
                "Baris harus mengisi tepat satu dari kolom debit atau kredit")
        if debit:
            return -abs(_parse_or_row_error(debit, mode=amount_mode))
        return abs(_parse_or_row_error(credit, mode=amount_mode))

    if amount_mode == "amount_type":
        value = _parse_or_row_error(_cell(row, column_map, "amount"), mode=amount_mode)
        type_value = (_cell(row, column_map, "type") or "").strip().lower()
        is_debit = any(token not in ("debit",) and type_value == token
                       or type_value.startswith(token) for token in DEBIT_TOKENS)
        return -abs(value) if is_debit else abs(value)

    raise ValidationFailed("invalid_amount_mode", f"Mode jumlah tidak dikenal: {amount_mode}")
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && .venv/bin/pytest tests/modules/imports/test_amounts.py -v`
Expected: 15 passed

- [ ] **Step 5: Commit**

```bash
git add backend/app/modules/imports/service.py backend/tests/modules/imports/test_amounts.py
git commit -m "feat: amount extraction for signed, debit_credit, amount_type, flip_outflow"
```

---

### Task 23: Date parsing with an ambiguity gate

**Files:**
- Modify: `backend/app/modules/imports/service.py`
- Test: `backend/tests/modules/imports/test_dates.py`

**Interfaces:**
- Consumes: `ValidationFailed`.
- Produces:
  - `parse_import_date(raw: str, date_format: str | None) -> str` — returns `YYYY-MM-DD`.
  - `DateAmbiguous(ValidationFailed)` with code `date_ambiguous`.

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/modules/imports/test_dates.py
import pytest

from app.core.errors import ValidationFailed
from app.modules.imports.service import parse_import_date


def test_explicit_format_is_honoured():
    assert parse_import_date("03/04/2026", "%d/%m/%Y") == "2026-04-03"
    assert parse_import_date("03/04/2026", "%m/%d/%Y") == "2026-03-04"


def test_iso_and_dashed_formats():
    assert parse_import_date("2026-02-01", None) == "2026-02-01"
    assert parse_import_date("01-02-2026", "%d-%m-%Y") == "2026-02-01"


def test_two_digit_year_expands():
    assert parse_import_date("01/02/26", "%d/%m/%Y") == "2026-02-01"


def test_unambiguous_day_over_12_is_inferred():
    assert parse_import_date("25/12/2026", None) == "2026-12-25"
    assert parse_import_date("12/25/2026", None) == "2026-12-25"


def test_genuinely_ambiguous_date_blocks_instead_of_guessing():
    """Review Focus: never silently file a year of transactions in the wrong month."""
    with pytest.raises(ValidationFailed) as e:
        parse_import_date("03/04/2026", None)
    assert e.value.code == "date_ambiguous"


def test_malformed_date_is_a_row_error():
    with pytest.raises(ValidationFailed) as e:
        parse_import_date("not a date", None)
    assert e.value.code == "date_parse_failed"


def test_wrong_explicit_format_is_a_row_error_not_a_guess():
    with pytest.raises(ValidationFailed) as e:
        parse_import_date("2026-02-01", "%d/%m/%Y")
    assert e.value.code == "date_parse_failed"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && .venv/bin/pytest tests/modules/imports/test_dates.py -v`
Expected: FAIL — `ImportError: cannot import name 'parse_import_date'`

- [ ] **Step 3: Add date parsing to `modules/imports/service.py`**

```python
import re
from datetime import datetime

AMBIGUOUS_PATTERN = re.compile(r"^\d{1,2}[/\-.]\d{1,2}[/\-.]\d{2,4}$")
_ISO_PATTERN = re.compile(r"^\d{4}-\d{2}-\d{2}$")


def _normalise_format(fmt: str) -> str:
    return fmt.replace("%y", "%Y")


def _try(value: str, fmt: str) -> str | None:
    try:
        return datetime.strptime(value, fmt).date().isoformat()
    except ValueError:
        return None


def parse_import_date(raw: str, date_format: str | None) -> str:
    value = (raw or "").strip()
    if not value:
        raise ValidationFailed("date_parse_failed", "Tanggal kosong")

    if date_format:
        parsed = _try(value, _normalise_format(date_format))
        if parsed is None:
            raise ValidationFailed("date_parse_failed",
                                   f"Tanggal {value!r} tidak sesuai format {date_format}")
        return parsed

    if _ISO_PATTERN.match(value):
        return value

    if AMBIGUOUS_PATTERN.match(value):
        day_first = _try(value, "%d/%m/%Y") or _try(value, "%d-%m-%Y") or _try(value, "%d.%m.%Y")
        month_first = _try(value, "%m/%d/%Y") or _try(value, "%m-%d-%Y") or _try(value, "%m.%d.%Y")
        if day_first and month_first and day_first != month_first:
            raise ValidationFailed(
                "date_ambiguous",
                f"Tanggal {value!r} bisa dibaca sebagai hari/bulan atau bulan/hari. "
                "Pilih format tanggal pada profil impor.")
        if day_first:
            return day_first
        if month_first:
            return month_first
        raise ValidationFailed("date_parse_failed", f"Tanggal tidak dikenali: {value!r}")

    for fmt in ("%d %b %Y", "%d %B %Y", "%Y%m%d"):
        parsed = _try(value, fmt)
        if parsed:
            return parsed

    raise ValidationFailed("date_parse_failed", f"Tanggal tidak dikenali: {value!r}")
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && .venv/bin/pytest tests/modules/imports/test_dates.py -v`
Expected: 7 passed

- [ ] **Step 5: Commit**

```bash
git add backend/app/modules/imports/service.py backend/tests/modules/imports/test_dates.py
git commit -m "feat: import date parsing with ambiguity gate instead of silent guessing"
```

---

### Task 24: Import hash and dedupe classification

**Files:**
- Modify: `backend/app/modules/imports/service.py`
- Test: `backend/tests/modules/imports/test_dedupe.py`

**Interfaces:**
- Consumes: `Transaction`, `accounts.service`.
- Produces:
  - `import_hash_for(account_id: int, iso_date: str, amount: int, description: str) -> str`
  - `RowResult` dataclass: `index, date, amount, description, status, error, category_id, paid_by, matched_rule_id, final_hash`
  - `classify_rows(s, *, account_id: int, drafts: list[dict]) -> list[RowResult]` where each draft is `{"date": str, "amount": int, "description": str}` plus the row index; statuses are `new`, `possible_duplicate`, `error`.

Re-importing the same file must be a no-op (Task 26 asserts this at the commit level).

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/modules/imports/test_dedupe.py
from datetime import date

import pytest

from app.modules.accounts import service as accounts
from app.modules.imports.service import RowResult, classify_rows, import_hash_for
from app.modules.transactions import service as txs


def test_hash_is_stable_across_whitespace_and_case():
    """Review Focus: the same logical row must hash identically on re-import."""
    a = import_hash_for(1, "2026-02-01", -186000, "Superindo  Jakarta")
    b = import_hash_for(1, "2026-02-01", -186000, "  superindo jakarta ")
    assert a == b


def test_hash_changes_with_account_date_amount_or_description():
    base = import_hash_for(1, "2026-02-01", -186000, "Superindo")
    assert base != import_hash_for(2, "2026-02-01", -186000, "Superindo")
    assert base != import_hash_for(1, "2026-02-02", -186000, "Superindo")
    assert base != import_hash_for(1, "2026-02-01", -186001, "Superindo")
    assert base != import_hash_for(1, "2026-02-01", -186000, "Indomaret")


def test_new_rows_are_new(db):
    with db.session() as s:
        acc = accounts.create_account(s, name="BCA", type="bank", opening_balance=0)
        results = classify_rows(s, account_id=acc.id, drafts=[
            {"date": "2026-02-01", "amount": -186000, "description": "Superindo"},
            {"date": "2026-02-02", "amount": 4200000, "description": "Gaji"},
        ])
    assert [r.status for r in results] == ["new", "new"]


def test_existing_hash_becomes_duplicate(db):
    with db.session() as s:
        acc = accounts.create_account(s, name="BCA", type="bank", opening_balance=0)
        txs.create_transaction(s, account_id=acc.id, date=date(2026, 2, 1), amount=-186000,
                               description="Superindo",
                               import_hash=import_hash_for(acc.id, "2026-02-01", -186000,
                                                           "Superindo"))
    with db.session() as s:
        results = classify_rows(s, account_id=acc.id, drafts=[
            {"date": "2026-02-01", "amount": -186000, "description": "Superindo"},
        ])
    assert results[0].status == "duplicate"


def test_same_amount_different_description_is_possible_duplicate(db):
    with db.session() as s:
        acc = accounts.create_account(s, name="BCA", type="bank", opening_balance=0)
        txs.create_transaction(s, account_id=acc.id, date=date(2026, 2, 1), amount=-50000,
                               description="Toko A", import_hash="unrelated")
    with db.session() as s:
        results = classify_rows(s, account_id=acc.id, drafts=[
            {"date": "2026-02-01", "amount": -50000, "description": "Toko B REF9931"},
        ])
    assert results[0].status == "possible_duplicate"


def test_two_distinct_same_day_rows_both_import(db):
    """Review Focus: identical date+amount+description rows must not collapse into one."""
    with db.session() as s:
        acc = accounts.create_account(s, name="BCA", type="bank", opening_balance=0)
        results = classify_rows(s, account_id=acc.id, drafts=[
            {"date": "2026-02-01", "amount": -20000, "description": "Parkir"},
            {"date": "2026-02-01", "amount": -20000, "description": "Parkir"},
            {"date": "2026-02-01", "amount": -20000, "description": "Parkir"},
        ])
    assert [r.status for r in results] == ["new", "new", "new"]
    assert len({r.final_hash for r in results}) == 3
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && .venv/bin/pytest tests/modules/imports/test_dedupe.py -v`
Expected: FAIL — `ImportError: cannot import name 'import_hash_for'`

- [ ] **Step 3: Add hashing and classification to `modules/imports/service.py`**

```python
import hashlib
from dataclasses import dataclass, field


def _normalise_for_hash(description: str) -> str:
    return " ".join((description or "").split()).casefold()


def import_hash_for(account_id: int, iso_date: str, amount: int, description: str) -> str:
    payload = f"{account_id}|{iso_date}|{int(amount)}|{_normalise_for_hash(description)}"
    return hashlib.sha256(payload.encode("utf-8")).hexdigest()


@dataclass
class RowResult:
    index: int
    date: str | None
    amount: int | None
    description: str
    status: str                       # new | possible_duplicate | duplicate | error
    error: str | None = None
    category_id: int | None = None
    paid_by: int | None = None
    matched_rule_id: int | None = None
    final_hash: str | None = None
    include: bool = True


def _existing_keys(s, account_id: int) -> tuple[set[str], set[tuple[str, int]]]:
    from sqlalchemy import select

    from app.modules.transactions.models import Transaction

    hashes: set[str] = set()
    pairs: set[tuple[str, int]] = set()
    rows = s.execute(
        select(Transaction.import_hash, Transaction.date, Transaction.amount)
        .where(Transaction.account_id == account_id)
    ).all()
    for import_hash, tx_date, amount in rows:
        if import_hash:
            hashes.add(import_hash)
        pairs.add((tx_date.isoformat() if hasattr(tx_date, "isoformat") else str(tx_date),
                   int(amount)))
    return hashes, pairs


def classify_rows(s, *, account_id: int, drafts: list[dict]) -> list[RowResult]:
    """Classify draft rows against existing data and against each other.

    Repeated identical rows within one file get a distinct occurrence suffix, so
    two genuinely separate transactions both import while a re-import of the same
    file stays a no-op.
    """
    existing_hashes, existing_pairs = _existing_keys(s, account_id)
    occurrence: dict[str, int] = {}
    results: list[RowResult] = []

    for i, draft in enumerate(drafts):
        description = " ".join((draft.get("description") or "").split())
        iso_date = draft["date"]
        amount = int(draft["amount"])
        base = import_hash_for(account_id, iso_date, amount, description)

        seen = occurrence.get(base, 0)
        occurrence[base] = seen + 1
        final_hash = base if seen == 0 else f"{base}#{seen}"

        if final_hash in existing_hashes:
            status = "duplicate"
        elif base in existing_hashes or (iso_date, amount) in existing_pairs:
            status = "possible_duplicate"
        else:
            status = "new"

        existing_hashes.add(final_hash)
        results.append(RowResult(index=i, date=iso_date, amount=amount,
                                 description=description, status=status,
                                 final_hash=final_hash))

    return results
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && .venv/bin/pytest tests/modules/imports/test_dedupe.py -v`
Expected: 6 passed

- [ ] **Step 5: Commit**

```bash
git add backend/app/modules/imports/service.py backend/tests/modules/imports/test_dedupe.py
git commit -m "feat: import hashing and dedupe classification with occurrence suffix"
```

---

### Task 25: Rules engine

**Files:**
- Create: `backend/app/modules/imports/rules.py`
- Create: `backend/app/modules/imports/models.py` addition: `Rule`
- Test: `backend/tests/modules/imports/test_rules.py`

**Interfaces:**
- Consumes: `Base`, `Category`, `Member`.
- Produces:
  - `Rule` (id, priority, match_type, match_value, set_category_id, set_paid_by, is_active) with `MATCH_TYPES = ("contains","starts_with","equals","regex")`
  - `rule_matches(rule, description: str) -> bool`
  - `apply_rules(s, description: str) -> tuple[int | None, int | None, int | None]` → `(category_id, paid_by, matched_rule_id)`
  - `reapply_rules(s, tx_ids: list[int] | None = None) -> int`

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/modules/imports/test_rules.py
import pytest

from app.core.errors import ValidationFailed
from app.modules.categories import service as categories
from app.modules.imports import rules as rules_mod
from app.modules.imports.models import Rule
from app.modules.transactions import service as txs
from app.modules.accounts import service as accounts
from app.modules.auth import service as auth
from datetime import date


def _rule(s, **kw):
    r = Rule(**kw)
    s.add(r)
    s.flush()
    return r


def test_match_types():
    assert rules_mod.rule_matches(Rule(match_type="contains", match_value="super"), "Superindo Jakarta")
    assert not rules_mod.rule_matches(Rule(match_type="contains", match_value="super"), "Indomaret")
    assert rules_mod.rule_matches(Rule(match_type="starts_with", match_value="super"), "SUPERINDO")
    assert not rules_mod.rule_matches(Rule(match_type="starts_with", match_value="super"), "Toko Superindo")
    assert rules_mod.rule_matches(Rule(match_type="equals", match_value="indomaret"), "Indomaret")
    assert not rules_mod.rule_matches(Rule(match_type="equals", match_value="indo"), "Indomaret")
    assert rules_mod.rule_matches(Rule(match_type="regex", match_value=r"^TRF \d+$"), "TRF 8837192")


def test_invalid_regex_does_not_crash():
    assert rules_mod.rule_matches(Rule(match_type="regex", match_value="([unclosed"), "anything") is False


def test_empty_match_value_never_matches():
    assert rules_mod.rule_matches(Rule(match_type="contains", match_value=""), "anything") is False


def test_first_matching_rule_wins_by_priority(db):
    with db.session() as s:
        food = categories.create_category(s, name="Makanan", kind="expense", emoji="🍚")
        other = categories.create_category(s, name="Lainnya", kind="expense", emoji="📦")
        _rule(s, priority=200, match_type="contains", match_value="superindo", set_category_id=other.id)
        _rule(s, priority=10, match_type="contains", match_value="super", set_category_id=food.id)
    with db.session() as s:
        category_id, paid_by, rule_id = rules_mod.apply_rules(s, "Superindo Jakarta")
        assert category_id == food.id


def test_lowercase_contains_is_case_insensitive(db):
    with db.session() as s:
        food = categories.create_category(s, name="Makanan", kind="expense", emoji="🍚")
        _rule(s, match_type="contains", match_value="SUPERINDO", set_category_id=food.id)
    with db.session() as s:
        assert rules_mod.apply_rules(s, "superindo jakarta")[0] == food.id


def test_no_rule_match_returns_nothing(db):
    with db.session() as s:
        assert rules_mod.apply_rules(s, "unmatched text") == (None, None, None)


def test_inactive_rules_are_skipped(db):
    with db.session() as s:
        food = categories.create_category(s, name="Makanan", kind="expense", emoji="🍚")
        _rule(s, match_type="contains", match_value="super", set_category_id=food.id, is_active=False)
    with db.session() as s:
        assert rules_mod.apply_rules(s, "Superindo")[0] is None


def test_rule_can_set_paid_by(db):
    with db.session() as s:
        budi = auth.create_member(s, name="Budi", username="budi",
                                  password="supersecret123", avatar_color="#EF476F")
        _rule(s, match_type="contains", match_value="netflix", set_paid_by=budi.id)
    with db.session() as s:
        assert rules_mod.apply_rules(s, "NETFLIX.COM")[1] == budi.id


def test_reapply_updates_existing_transactions(db):
    with db.session() as s:
        acc = accounts.create_account(s, name="BCA", type="bank", opening_balance=0)
        tx = txs.create_transaction(s, account_id=acc.id, date=date(2026, 2, 1),
                                    amount=-186000, description="Superindo Jakarta")
        tx_id = tx.id
        food = categories.create_category(s, name="Makanan", kind="expense", emoji="🍚")
        _rule(s, match_type="contains", match_value="superindo", set_category_id=food.id)
    with db.session() as s:
        assert rules_mod.reapply_rules(s) == 1
    with db.session() as s:
        assert txs.get_transaction(s, tx_id).category_id == food.id


def test_unknown_match_type_is_rejected_by_the_database(db):
    from sqlalchemy.exc import IntegrityError

    with db.session() as s:
        s.add(Rule(match_type="nonsense", match_value="x"))
        with pytest.raises(IntegrityError):
            s.flush()

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && .venv/bin/pytest tests/modules/imports/test_rules.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'app.modules.imports.rules'`

- [ ] **Step 3: Add `Rule` to `modules/imports/models.py`**

```python
class Rule(Base):
    __tablename__ = "rules"
    __table_args__ = (
        CheckConstraint(
            "match_type IN ('contains','starts_with','equals','regex')",
            name="ck_rules_match_type"),
        Index("ix_rules_order", "is_active", "priority"),
    )

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    priority: Mapped[int] = mapped_column(Integer, nullable=False, default=100)
    match_type: Mapped[str] = mapped_column(String, nullable=False)
    match_value: Mapped[str] = mapped_column(String, nullable=False)
    set_category_id: Mapped[int | None] = mapped_column(ForeignKey("categories.id"), nullable=True)
    set_paid_by: Mapped[int | None] = mapped_column(ForeignKey("members.id"), nullable=True)
    is_active: Mapped[bool] = mapped_column(Boolean, nullable=False, default=True)
```

Add `Index` to the imports from `sqlalchemy` at the top of that file.

- [ ] **Step 4: Write `modules/imports/rules.py`**

```python
import re

from sqlalchemy import select

from app.modules.imports.models import Rule


def rule_matches(rule: Rule, description: str) -> bool:
    value = (rule.match_value or "").strip()
    if not value:
        return False
    text = (description or "").casefold()
    needle = value.casefold()
    if rule.match_type == "contains":
        return needle in text
    if rule.match_type == "starts_with":
        return text.startswith(needle)
    if rule.match_type == "equals":
        return text == needle
    if rule.match_type == "regex":
        try:
            return re.search(value, description or "", re.IGNORECASE) is not None
        except re.error:
            return False
    return False


def active_rules(s) -> list[Rule]:
    return list(s.execute(
        select(Rule).where(Rule.is_active.is_(True)).order_by(Rule.priority, Rule.id)
    ).scalars())


def apply_rules(s, description: str) -> tuple[int | None, int | None, int | None]:
    """First matching rule that sets each field wins. Fields resolve independently."""
    category_id: int | None = None
    paid_by: int | None = None
    matched_rule: int | None = None
    for rule in active_rules(s):
        if not rule_matches(rule, description):
            continue
        changed = False
        if category_id is None and rule.set_category_id is not None:
            category_id = rule.set_category_id
            changed = True
        if paid_by is None and rule.set_paid_by is not None:
            paid_by = rule.set_paid_by
            changed = True
        if changed and matched_rule is None:
            matched_rule = rule.id
        if category_id is not None and paid_by is not None:
            break
    return category_id, paid_by, matched_rule


def reapply_rules(s, tx_ids: list[int] | None = None) -> int:
    """Apply rules to existing transactions. Only fills fields that are still empty."""
    from sqlalchemy import select as _select

    from app.modules.transactions.models import Transaction

    query = _select(Transaction)
    if tx_ids is not None:
        query = query.where(Transaction.id.in_(tx_ids))
    updated = 0
    for tx in s.execute(query).scalars():
        category_id, paid_by, _ = apply_rules(s, tx.description)
        changed = False
        if tx.category_id is None and category_id is not None:
            tx.category_id = category_id
            changed = True
        if tx.paid_by is None and paid_by is not None:
            tx.paid_by = paid_by
            changed = True
        if changed:
            updated += 1
    s.flush()
    return updated
```

- [ ] **Step 5: Generate the migration and run the tests**

Run: `cd backend && .venv/bin/python -m alembic revision --autogenerate -m "rules" && .venv/bin/pytest tests/modules/imports/test_rules.py -v`
Expected: migration creates `rules`; 9 passed

- [ ] **Step 6: Commit**

```bash
git add backend/app/modules/imports/rules.py backend/app/modules/imports/models.py backend/app/migrations/versions backend/tests/modules/imports/test_rules.py
git commit -m "feat: rule engine with priority order and reapply to existing transactions"
```

---

### Task 26: Import analyze, commit, and rollback

**Files:**
- Modify: `backend/app/modules/imports/service.py`
- Create: `backend/app/modules/imports/schemas.py`
- Test: `backend/tests/modules/imports/test_import_flow.py`

**Interfaces:**
- Consumes: everything above.
- Produces:
  - `analyse_import(s, *, raw: bytes, filename: str, account_id: int, column_map: dict, amount_mode: str, date_format: str | None, max_rows: int, forced: dict | None = None) -> tuple[Sniffed, list[RowResult]]` — `forced` carries user overrides (`delimiter`, `encoding`, `has_header`, `skip_rows`) into sniffing
  - `commit_import(s, *, account_id: int, member_id: int, filename: str, profile_id: int | None, rows: list[dict]) -> Import`
  - `rollback_import(s, import_id: int) -> Import`

`rows` passed to `commit_import` are the accepted rows from the review step — `{"date", "amount", "description", "category_id", "paid_by"}`. The occurrence suffix and hashes are recomputed server-side from the full ordered list, so a client cannot forge a dedupe bypass.

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/modules/imports/test_import_flow.py
from datetime import date

from app.modules.accounts import service as accounts
from app.modules.imports import service as imp
from app.modules.transactions import service as txs

CSV = (b"Tanggal,Keterangan,Jumlah\n"
       b"01/02/2026,Superindo Jakarta,-186000\n"
       b"02/02/2026,Gaji Februari,4200000\n"
       b"03/02/2026,Netflix,-54000\n")

COLUMN_MAP = {"date": 0, "description": 1, "amount": 2}


def _setup(db):
    with db.session() as s:
        acc = accounts.create_account(s, name="BCA", type="bank", opening_balance=1_000_000)
        return acc.id


def test_analyse_returns_row_statuses(db):
    acc_id = _setup(db)
    with db.session() as s:
        sniffed, results = imp.analyse_import(
            s, raw=CSV, filename="bca.csv", account_id=acc_id,
            column_map=COLUMN_MAP, amount_mode="signed", date_format="%d/%m/%Y",
            max_rows=1000)
    assert sniffed.header == ["Tanggal", "Keterangan", "Jumlah"]
    assert [r.status for r in results] == ["new", "new", "new"]
    assert results[0].amount == -186000
    assert results[1].date == "2026-02-02"


def test_analyse_flags_ambiguous_dates_as_row_errors(db):
    acc_id = _setup(db)
    ambiguous = b"Tanggal,Keterangan,Jumlah\n03/04/2026,Superindo,-186000\n"
    with db.session() as s:
        _, results = imp.analyse_import(s, raw=ambiguous, filename="x.csv", account_id=acc_id,
                                        column_map=COLUMN_MAP, amount_mode="signed",
                                        date_format=None, max_rows=1000)
    assert results[0].status == "error"
    assert "bulan/hari" in (results[0].error or "")


def test_commit_inserts_and_reports_counts(db):
    acc_id = _setup(db)
    with db.session() as s:
        _, results = imp.analyse_import(s, raw=CSV, filename="bca.csv", account_id=acc_id,
                                        column_map=COLUMN_MAP, amount_mode="signed",
                                        date_format="%d/%m/%Y", max_rows=1000)
        rows = [{"date": r.date, "amount": r.amount, "description": r.description}
                for r in results]
    with db.session() as s:
        batch = imp.commit_import(s, account_id=acc_id, member_id=1, filename="bca.csv",
                                  profile_id=None, rows=rows)
        assert batch.imported_count == 3
        assert batch.duplicate_count == 0
        assert batch.row_count == 3
    with db.session() as s:
        assert accounts.account_balance(s, acc_id) == 1_000_000 - 186_000 + 4_200_000 - 54_000


def test_reimporting_the_same_rows_is_a_no_op(db):
    """Review Focus: re-import must not double-count."""
    acc_id = _setup(db)
    with db.session() as s:
        _, results = imp.analyse_import(s, raw=CSV, filename="bca.csv", account_id=acc_id,
                                        column_map=COLUMN_MAP, amount_mode="signed",
                                        date_format="%d/%m/%Y", max_rows=1000)
        rows = [{"date": r.date, "amount": r.amount, "description": r.description}
                for r in results]
    with db.session() as s:
        imp.commit_import(s, account_id=acc_id, member_id=1, filename="bca.csv",
                          profile_id=None, rows=rows)
    with db.session() as s:
        _, second = imp.analyse_import(s, raw=CSV, filename="bca.csv", account_id=acc_id,
                                       column_map=COLUMN_MAP, amount_mode="signed",
                                       date_format="%d/%m/%Y", max_rows=1000)
    assert all(r.status == "duplicate" for r in second)
    with db.session() as s:
        batch = imp.commit_import(s, account_id=acc_id, member_id=1, filename="bca.csv",
                                  profile_id=None, rows=[])
        assert batch.imported_count == 0
    with db.session() as s:
        _, total = txs.list_transactions(s, limit=100)
        assert total == 3


def test_rollback_removes_exactly_the_batch(db):
    acc_id = _setup(db)
    with db.session() as s:
        _, results = imp.analyse_import(s, raw=CSV, filename="bca.csv", account_id=acc_id,
                                        column_map=COLUMN_MAP, amount_mode="signed",
                                        date_format="%d/%m/%Y", max_rows=1000)
        rows = [{"date": r.date, "amount": r.amount, "description": r.description}
                for r in results]
    with db.session() as s:
        batch = imp.commit_import(s, account_id=acc_id, member_id=1, filename="bca.csv",
                                  profile_id=None, rows=rows)
        batch_id = batch.id
    with db.session() as s:
        txs.create_transaction(s, account_id=acc_id, date=date(2026, 2, 9), amount=-5000,
                               description="manual entry")
    with db.session() as s:
        imp.rollback_import(s, batch_id)
    with db.session() as s:
        rows_left, total = txs.list_transactions(s, limit=100)
        assert total == 1
        assert rows_left[0].description == "manual entry"
    with db.session() as s:
        assert accounts.account_balance(s, acc_id) == 1_000_000 - 5_000


def test_rules_are_applied_at_analyse_time(db):
    from app.modules.categories import service as categories
    from app.modules.imports.models import Rule

    acc_id = _setup(db)
    with db.session() as s:
        food = categories.create_category(s, name="Makanan", kind="expense", emoji="🍚")
        s.add(Rule(priority=10, match_type="contains", match_value="superindo",
                   set_category_id=food.id))
        s.flush()
    with db.session() as s:
        _, results = imp.analyse_import(s, raw=CSV, filename="bca.csv", account_id=acc_id,
                                        column_map=COLUMN_MAP, amount_mode="signed",
                                        date_format="%d/%m/%Y", max_rows=1000)
    assert results[0].category_id is not None
    assert results[0].matched_rule_id is not None
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && .venv/bin/pytest tests/modules/imports/test_import_flow.py -v`
Expected: FAIL — `AttributeError: module ... has no attribute 'analyse_import'`

- [ ] **Step 3: Add the flow to `modules/imports/service.py`**

```python
from app.modules.imports import rules as rules_mod


def _build_drafts(sniffed: Sniffed, column_map: dict, amount_mode: str,
                  date_format: str | None) -> tuple[list[dict], list[RowResult]]:
    """Parse every row. Failures become RowResults with status 'error'; they never abort."""
    drafts: list[dict] = []
    errors: list[RowResult] = []

    for i, row in enumerate(sniffed.rows):
        description = (_cell(row, column_map, "description") or "").strip()
        try:
            iso_date = parse_import_date(_cell(row, column_map, "date") or "", date_format)
            amount = extract_amount(row, column_map, amount_mode)
        except ValidationFailed as exc:
            errors.append(RowResult(index=i, date=None, amount=None, description=description,
                                    status="error", error=exc.message, include=False))
            continue
        drafts.append({"index": i, "date": iso_date, "amount": amount,
                       "description": description})

    return drafts, errors


def analyse_import(s, *, raw: bytes, filename: str, account_id: int, column_map: dict,
                   amount_mode: str, date_format: str | None, max_rows: int,
                   forced: dict | None = None) -> tuple[Sniffed, list[RowResult]]:
    from app.modules.accounts import service as accounts

    accounts.get_account(s, account_id)

    sniffed = sniff_csv(raw, max_rows=max_rows, forced=forced)
    drafts, errors = _build_drafts(sniffed, column_map, amount_mode, date_format)

    results = classify_rows(s, account_id=account_id, drafts=drafts)
    for r in results:
        category_id, paid_by, rule_id = rules_mod.apply_rules(s, r.description)
        r.category_id, r.paid_by, r.matched_rule_id = category_id, paid_by, rule_id

    merged = errors + results
    merged.sort(key=lambda r: r.index)
    return sniffed, merged


def commit_import(s, *, account_id: int, member_id: int, filename: str,
                  profile_id: int | None, rows: list[dict]) -> Import:
    """Insert the accepted rows in a single transaction. Hashes are recomputed here."""
    from app.modules.transactions import service as txs

    results = classify_rows(s, account_id=account_id, drafts=[
        {"date": r["date"], "amount": r["amount"], "description": r.get("description", "")}
        for r in rows
    ])

    batch = Import(profile_id=profile_id, account_id=account_id, filename=filename[:255],
                   uploaded_by=member_id, status="committed",
                   row_count=len(rows), imported_count=0, duplicate_count=0,
                   possible_duplicate_count=0, error_count=0)
    s.add(batch)
    s.flush()

    for row, result in zip(rows, results):
        if result.status == "duplicate":
            batch.duplicate_count += 1
            continue
        if result.status == "possible_duplicate":
            batch.possible_duplicate_count += 1
        txs.create_transaction(
            s, account_id=account_id, date=row["date"], amount=row["amount"],
            description=row.get("description", ""),
            category_id=row.get("category_id"), paid_by=row.get("paid_by"),
            import_id=batch.id, import_hash=result.final_hash)
        batch.imported_count += 1

    s.flush()
    return batch


def rollback_import(s, import_id: int) -> Import:
    from sqlalchemy import delete

    from app.modules.transactions.models import Transaction

    batch = s.get(Import, import_id)
    if batch is None:
        raise ValidationFailed("import_not_found", "Riwayat impor tidak ditemukan")
    s.execute(delete(Transaction).where(Transaction.import_id == batch.id))
    batch.status = "rolled_back"
    batch.imported_count = 0
    s.flush()
    return batch
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && .venv/bin/pytest tests/modules/imports/test_import_flow.py -v`
Expected: 7 passed

- [ ] **Step 5: Commit**

```bash
git add backend/app/modules/imports/service.py backend/tests/modules/imports/test_import_flow.py
git commit -m "feat: import analyze, commit, and rollback with server-side hash recomputation"
```

---

### Task 27: Import and profile API

**Files:**
- Create: `backend/app/modules/imports/schemas.py`
- Create: `backend/app/modules/imports/router.py`
- Modify: `backend/app/main.py`
- Test: `backend/tests/modules/imports/test_imports_api.py`

**Interfaces:**
- Consumes: `service`, `rules_mod`, `current_member`, `Settings`.
- Produces routes:
  - `POST /api/imports/analyze` (multipart: `file`, `account_id`, `profile_id?`, `delimiter?`, `encoding?`, `has_header?`, `skip_rows?`, `date_format?`, `amount_mode?`, `column_map` JSON) → `AnalyzeOut`
  - `POST /api/imports/commit` (JSON: `account_id`, `filename`, `profile_id?`, `save_profile?{name,bank_name}`, `rows[]`) → `ImportOut`
  - `GET /api/imports` → `list[ImportOut]`
  - `POST /api/imports/{id}/rollback` → `ImportOut`
  - `GET /api/import-profiles` → `list[ProfileOut]`
  - `POST /api/import-profiles` → `ProfileOut`
  - `DELETE /api/import-profiles/{id}` → `{"ok": true}`

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/modules/imports/test_imports_api.py
import json

CSV = (b"Tanggal,Keterangan,Jumlah\n"
       b"01/02/2026,Superindo Jakarta,-186000\n"
       b"02/02/2026,Gaji Februari,4200000\n")

COLUMN_MAP = {"date": 0, "description": 1, "amount": 2}


def _setup(client):
    client.post("/api/setup", json={"name": "Budi", "username": "budi",
                                    "password": "supersecret123"})
    return client.post("/api/accounts", json={"name": "BCA", "type": "bank",
                                              "opening_balance": 1000000}).json()["id"]


def test_analyze_then_commit_then_rollback(client):
    acc = _setup(client)
    r = client.post("/api/imports/analyze",
                    data={"account_id": str(acc), "amount_mode": "signed",
                          "date_format": "%d/%m/%Y", "column_map": json.dumps(COLUMN_MAP)},
                    files={"file": ("bca.csv", CSV, "text/csv")})
    assert r.status_code == 200, r.text
    body = r.json()
    assert body["header"] == ["Tanggal", "Keterangan", "Jumlah"]
    assert [row["status"] for row in body["rows"]] == ["new", "new"]

    rows = [{"date": x["date"], "amount": x["amount"], "description": x["description"]}
            for x in body["rows"]]
    r = client.post("/api/imports/commit",
                    json={"account_id": acc, "filename": "bca.csv", "rows": rows})
    assert r.status_code == 200
    batch = r.json()
    assert batch["imported_count"] == 2

    balances = {a["name"]: a["balance"] for a in client.get("/api/accounts").json()}
    assert balances["BCA"] == 1000000 - 186000 + 4200000

    assert client.get("/api/imports").json()[0]["filename"] == "bca.csv"
    assert client.post(f"/api/imports/{batch['id']}/rollback").json()["status"] == "rolled_back"
    assert client.get("/api/accounts").json()[0]["balance"] == 1000000


def test_oversized_upload_is_rejected(client):
    acc = _setup(client)
    big = b"Tanggal,Keterangan,Jumlah\n" + (b"01/02/2026,x,-1\n" * 10)
    r = client.post("/api/imports/analyze",
                    data={"account_id": str(acc), "amount_mode": "signed",
                          "column_map": json.dumps(COLUMN_MAP)},
                    files={"file": ("big.csv", big, "text/csv")})
    assert r.status_code in (200, 422)


def test_profile_crud_and_reuse(client):
    _setup(client)
    r = client.post("/api/import-profiles", json={
        "name": "BCA Februari", "bank_name": "BCA", "delimiter": ",",
        "encoding": "utf-8", "has_header": True, "skip_rows": 0, "date_format": "%d/%m/%Y",
        "amount_mode": "signed", "column_map": json.dumps(COLUMN_MAP)})
    assert r.status_code == 200
    profile = r.json()
    assert client.get("/api/import-profiles").json()[0]["name"] == "BCA Februari"
    assert client.delete(f"/api/import-profiles/{profile['id']}").json() == {"ok": True}
    assert client.get("/api/import-profiles").json() == []


def test_import_requires_auth(client):
    assert client.get("/api/imports").status_code == 401
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && .venv/bin/pytest tests/modules/imports/test_imports_api.py -v`
Expected: FAIL — 404 on `/api/imports/analyze`

- [ ] **Step 3: Write `modules/imports/schemas.py`**

```python
from datetime import datetime

from pydantic import BaseModel, ConfigDict, Field


class RowOut(BaseModel):
    index: int
    date: str | None
    amount: int | None
    description: str
    status: str
    error: str | None = None
    category_id: int | None = None
    paid_by: int | None = None
    matched_rule_id: int | None = None


class AnalyzeOut(BaseModel):
    encoding: str
    delimiter: str
    has_header: bool
    header: list[str] | None
    detected: dict
    counts: dict[str, int]
    rows: list[RowOut]


class RowIn(BaseModel):
    date: str
    amount: int
    description: str = ""
    category_id: int | None = None
    paid_by: int | None = None


class SaveProfileIn(BaseModel):
    name: str = Field(min_length=1, max_length=80)
    bank_name: str | None = None
    delimiter: str = ","
    encoding: str = "utf-8"
    has_header: bool = True
    skip_rows: int = 0
    date_format: str | None = None
    amount_mode: str = "signed"
    column_map: dict


class CommitIn(BaseModel):
    account_id: int
    filename: str
    profile_id: int | None = None
    save_profile: SaveProfileIn | None = None
    rows: list[RowIn]


class ImportOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    profile_id: int | None
    account_id: int
    filename: str
    uploaded_by: int
    uploaded_at: datetime
    status: str
    row_count: int
    imported_count: int
    duplicate_count: int
    possible_duplicate_count: int
    error_count: int


class ProfileOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    name: str
    bank_name: str | None
    delimiter: str
    encoding: str
    has_header: bool
    skip_rows: int
    date_format: str | None
    amount_mode: str
    column_map: str


class ProfileIn(BaseModel):
    name: str = Field(min_length=1, max_length=80)
    bank_name: str | None = None
    delimiter: str = ","
    encoding: str = "utf-8"
    has_header: bool = True
    skip_rows: int = 0
    date_format: str | None = None
    amount_mode: str = "signed"
    column_map: str
```

- [ ] **Step 4: Write `modules/imports/router.py`**

```python
import json

from fastapi import APIRouter, File, Form, Request, UploadFile
from sqlalchemy import select

from app.core.deps import current_member
from app.core.errors import ValidationFailed
from app.modules.imports import service
from app.modules.imports.models import ImportProfile
from app.modules.imports.schemas import (AnalyzeOut, CommitIn, ImportOut, ProfileIn,
                                         ProfileOut, RowOut)

router = APIRouter(prefix="/api", tags=["imports"])


def _db(request):
    return request.app.state.db


@router.post("/imports/analyze", response_model=AnalyzeOut)
async def analyze_import(request: Request, file: UploadFile = File(...),
                         account_id: int = Form(...), profile_id: int | None = Form(None),
                         delimiter: str | None = Form(None), encoding: str | None = Form(None),
                         has_header: bool | None = Form(None), skip_rows: int = Form(0),
                         date_format: str | None = Form(None),
                         amount_mode: str | None = Form(None),
                         column_map: str | None = Form(None)):
    current_member(request)
    settings = request.app.state.settings
    raw = await file.read()
    if len(raw) > settings.max_upload_bytes:
        raise ValidationFailed("file_too_large",
                               f"Ukuran berkas melebihi {settings.max_upload_bytes // (1024 * 1024)} MB")

    with _db(request).session() as s:
        profile = s.get(ImportProfile, profile_id) if profile_id else None
        effective_map = json.loads(column_map) if column_map else (
            json.loads(profile.column_map) if profile else None)
        effective_mode = amount_mode or (profile.amount_mode if profile else None)
        effective_date_format = date_format if date_format is not None else (
            profile.date_format if profile else None)
        if not effective_map or not effective_mode:
            raise ValidationFailed("mapping_required",
                                   "Pemetaan kolom dan mode jumlah wajib diisi")

        forced = {
            "delimiter": delimiter or (profile.delimiter if profile else None),
            "encoding": encoding or (profile.encoding if profile else None),
            "has_header": (has_header if has_header is not None
                           else (profile.has_header if profile else None)),
            "skip_rows": skip_rows or (profile.skip_rows if profile else 0),
        }
        forced = {k: v for k, v in forced.items() if v is not None}

        sniffed, results = service.analyse_import(
            s, raw=raw, filename=file.filename or "upload.csv", account_id=account_id,
            column_map=effective_map, amount_mode=effective_mode,
            date_format=effective_date_format, max_rows=settings.max_import_rows,
            forced=forced)

        counts: dict[str, int] = {}
        for r in results:
            counts[r.status] = counts.get(r.status, 0) + 1

        return AnalyzeOut(
            encoding=sniffed.encoding,
            delimiter=sniffed.delimiter,
            has_header=sniffed.has_header,
            header=sniffed.header,
            detected={"encoding": sniffed.encoding, "delimiter": sniffed.delimiter,
                      "has_header": sniffed.has_header, "skip_rows": skip_rows},
            counts=counts,
            rows=[RowOut(index=r.index, date=r.date, amount=r.amount,
                         description=r.description, status=r.status, error=r.error,
                         category_id=r.category_id, paid_by=r.paid_by,
                         matched_rule_id=r.matched_rule_id) for r in results])


@router.post("/imports/commit", response_model=ImportOut)
def commit_import(payload: CommitIn, request: Request):
    member = current_member(request)
    settings = request.app.state.settings
    if len(payload.rows) > settings.max_import_rows:
        raise ValidationFailed("too_many_rows", "Terlalu banyak baris")
    with _db(request).session() as s:
        profile_id = payload.profile_id
        if payload.save_profile is not None:
            saved = ImportProfile(
                name=payload.save_profile.name, bank_name=payload.save_profile.bank_name,
                delimiter=payload.save_profile.delimiter,
                encoding=payload.save_profile.encoding,
                has_header=payload.save_profile.has_header,
                skip_rows=payload.save_profile.skip_rows,
                date_format=payload.save_profile.date_format,
                amount_mode=payload.save_profile.amount_mode,
                column_map=json.dumps(payload.save_profile.column_map))
            s.add(saved)
            s.flush()
            profile_id = saved.id
        batch = service.commit_import(
            s, account_id=payload.account_id, member_id=member.id,
            filename=payload.filename, profile_id=profile_id,
            rows=[r.model_dump() for r in payload.rows])
        return ImportOut.model_validate(batch)


@router.get("/imports", response_model=list[ImportOut])
def list_imports(request: Request, account_id: int | None = None):
    current_member(request)
    from app.modules.imports.models import Import

    with _db(request).session() as s:
        query = select(Import).order_by(Import.uploaded_at.desc(), Import.id.desc())
        if account_id is not None:
            query = query.where(Import.account_id == account_id)
        return [ImportOut.model_validate(i) for i in s.execute(query).scalars()]


@router.post("/imports/{import_id}/rollback", response_model=ImportOut)
def rollback_import(import_id: int, request: Request):
    current_member(request)
    with _db(request).session() as s:
        return ImportOut.model_validate(service.rollback_import(s, import_id))


@router.get("/import-profiles", response_model=list[ProfileOut])
def list_profiles(request: Request):
    current_member(request)
    with _db(request).session() as s:
        return [ProfileOut.model_validate(p) for p in
                s.execute(select(ImportProfile).order_by(ImportProfile.name)).scalars()]


@router.post("/import-profiles", response_model=ProfileOut)
def create_profile(payload: ProfileIn, request: Request):
    current_member(request)
    with _db(request).session() as s:
        profile = ImportProfile(**payload.model_dump())
        s.add(profile)
        s.flush()
        return ProfileOut.model_validate(profile)


@router.delete("/import-profiles/{profile_id}")
def delete_profile(profile_id: int, request: Request):
    current_member(request)
    with _db(request).session() as s:
        profile = s.get(ImportProfile, profile_id)
        if profile is not None:
            s.delete(profile)
    return {"ok": True}
```

- [ ] **Step 5: Include the router and run the suite**

Add to `main.py`: `from app.modules.imports.router import router as imports_router` and `app.include_router(imports_router)`.

Run: `cd backend && .venv/bin/pytest -v`
Expected: all passed

- [ ] **Step 6: Commit**

```bash
git add backend/app/modules/imports/schemas.py backend/app/modules/imports/router.py backend/app/main.py backend/tests/modules/imports/test_imports_api.py
git commit -m "feat: import analyze/commit/rollback api and import profiles"
```

---

# Phase 6 — Reports

### Task 28: Reports service

**Files:**
- Create: `backend/app/modules/reports/__init__.py`, `service.py`
- Test: `backend/tests/modules/reports/test_reports.py`

**Interfaces:**
- Consumes: `accounts.service`, `transactions.service`, `Transaction`.
- Produces:
  - `net_worth(s) -> dict` → `{"total": int, "accounts": [{"id","name","type","balance"}]}`
  - `cash_flow(s, *, date_from, date_to) -> dict` → `{"income": int, "expense": int, "net": int}`
  - `spend_by_category(s, *, date_from, date_to, kind="expense") -> list[dict]` → `[{"category_id","name","emoji","color","total"}]`

**Transfers are excluded from income, expense, and category totals.** A transfer is money moving between your own accounts — counting it as both income and expense inflates every report.

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/modules/reports/test_reports.py
from datetime import date

import pytest

from app.modules.accounts import service as accounts
from app.modules.categories import service as categories
from app.modules.reports import service as reports
from app.modules.transactions import service as txs


@pytest.fixture
def world(db):
    with db.session() as s:
        bca = accounts.create_account(s, name="BCA", type="bank", opening_balance=1_000_000)
        cash = accounts.create_account(s, name="Tunai", type="cash", opening_balance=200_000)
        food = categories.create_category(s, name="Makanan", kind="expense", emoji="🍚")
        salary = categories.create_category(s, name="Gaji", kind="income", emoji="💼")
        return {"bca": bca.id, "cash": cash.id, "food": food.id, "salary": salary.id}


def test_net_worth_sums_account_balances(db, world):
    with db.session() as s:
        txs.create_transaction(s, account_id=world["bca"], date=date(2026, 2, 1),
                               amount=-186_000, description="Superindo",
                               category_id=world["food"])
    with db.session() as s:
        result = reports.net_worth(s)
    assert result["total"] == 1_000_000 + 200_000 - 186_000
    assert {a["name"] for a in result["accounts"]} == {"BCA", "Tunai"}


def test_transfer_does_not_change_net_worth(db, world):
    """Review Focus: an internal transfer must be invisible to net worth."""
    with db.session() as s:
        before = reports.net_worth(s)["total"]
    with db.session() as s:
        txs.create_transfer(s, from_account_id=world["bca"], to_account_id=world["cash"],
                            date=date(2026, 2, 5), amount=500_000, description="Tarik tunai")
    with db.session() as s:
        assert reports.net_worth(s)["total"] == before
        balances = {a["name"]: a["balance"] for a in reports.net_worth(s)["accounts"]}
        assert balances == {"BCA": 500_000, "Tunai": 700_000}


def test_transfer_is_excluded_from_cash_flow(db, world):
    """A transfer is not income and not expense — it must not inflate either."""
    with db.session() as s:
        txs.create_transfer(s, from_account_id=world["bca"], to_account_id=world["cash"],
                            date=date(2026, 2, 5), amount=500_000, description="Tarik tunai")
        txs.create_transaction(s, account_id=world["bca"], date=date(2026, 2, 6),
                               amount=-186_000, description="Superindo",
                               category_id=world["food"])
        txs.create_transaction(s, account_id=world["bca"], date=date(2026, 2, 7),
                               amount=4_200_000, description="Gaji",
                               category_id=world["salary"])
    with db.session() as s:
        cf = reports.cash_flow(s, date_from=date(2026, 2, 1), date_to=date(2026, 2, 28))
    assert cf == {"income": 4_200_000, "expense": 186_000, "net": 4_014_000}


def test_spend_by_category_excludes_transfers_and_other_kinds(db, world):
    with db.session() as s:
        txs.create_transfer(s, from_account_id=world["bca"], to_account_id=world["cash"],
                            date=date(2026, 2, 5), amount=500_000, description="Tarik tunai")
        txs.create_transaction(s, account_id=world["bca"], date=date(2026, 2, 6),
                               amount=-186_000, description="Superindo",
                               category_id=world["food"])
        txs.create_transaction(s, account_id=world["bca"], date=date(2026, 2, 7),
                               amount=-54_000, description="Netflix",
                               category_id=world["food"])
        txs.create_transaction(s, account_id=world["bca"], date=date(2026, 2, 8),
                               amount=-9_000, description="Belum dikategorikan")
    with db.session() as s:
        rows = reports.spend_by_category(s, date_from=date(2026, 2, 1),
                                         date_to=date(2026, 2, 28))
    assert len(rows) == 1
    assert rows[0]["name"] == "Makanan"
    assert rows[0]["total"] == 240_000


def test_uncategorised_spending_is_reported_separately(db, world):
    with db.session() as s:
        txs.create_transaction(s, account_id=world["bca"], date=date(2026, 2, 8),
                               amount=-9_000, description="TRF 8837192")
    with db.session() as s:
        rows = reports.spend_by_category(s, date_from=date(2026, 2, 1),
                                         date_to=date(2026, 2, 28))
    assert rows == [{"category_id": None, "name": "Belum dikategorikan", "emoji": "❓",
                     "color": "#8D99AE", "total": 9_000}]


def test_date_range_is_inclusive_on_both_ends(db, world):
    with db.session() as s:
        txs.create_transaction(s, account_id=world["bca"], date=date(2026, 2, 1),
                               amount=-1_000, description="first")
        txs.create_transaction(s, account_id=world["bca"], date=date(2026, 2, 28),
                               amount=-2_000, description="last")
        txs.create_transaction(s, account_id=world["bca"], date=date(2026, 3, 1),
                               amount=-4_000, description="outside")
    with db.session() as s:
        cf = reports.cash_flow(s, date_from=date(2026, 2, 1), date_to=date(2026, 2, 28))
    assert cf["expense"] == 3_000


def test_income_report_uses_the_income_kind(db, world):
    with db.session() as s:
        txs.create_transaction(s, account_id=world["bca"], date=date(2026, 2, 7),
                               amount=4_200_000, description="Gaji",
                               category_id=world["salary"])
    with db.session() as s:
        rows = reports.spend_by_category(s, date_from=date(2026, 2, 1),
                                         date_to=date(2026, 2, 28), kind="income")
    assert rows[0]["name"] == "Gaji"
    assert rows[0]["total"] == 4_200_000


def test_archived_accounts_are_excluded_from_net_worth(db, world):
    with db.session() as s:
        accounts.archive_account(s, world["cash"])
    with db.session() as s:
        result = reports.net_worth(s)
    assert {a["name"] for a in result["accounts"]} == {"BCA"}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && .venv/bin/pytest tests/modules/reports/test_reports.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'app.modules.reports'`

- [ ] **Step 3: Write `modules/reports/service.py`**

```python
from sqlalchemy import func, select

from app.modules.accounts import service as accounts
from app.modules.transactions.models import Transaction

UNCATEGORISED = {"category_id": None, "name": "Belum dikategorikan",
                 "emoji": "❓", "color": "#8D99AE"}


def net_worth(s) -> dict:
    rows = []
    total = 0
    for account in accounts.list_accounts(s):
        balance = accounts.account_balance(s, account.id)
        total += balance
        rows.append({"id": account.id, "name": account.name,
                     "type": account.type, "balance": balance})
    return {"total": total, "accounts": rows}


def cash_flow(s, *, date_from, date_to) -> dict:
    conditions = [
        Transaction.transfer_group_id.is_(None),
        Transaction.date >= date_from,
        Transaction.date <= date_to,
    ]
    income = s.execute(
        select(func.coalesce(func.sum(Transaction.amount), 0))
        .where(*conditions, Transaction.amount > 0)
    ).scalar_one()
    expense = s.execute(
        select(func.coalesce(func.sum(Transaction.amount), 0))
        .where(*conditions, Transaction.amount < 0)
    ).scalar_one()
    income, expense = int(income), int(expense)
    return {"income": income, "expense": expense, "net": income + expense}


def spend_by_category(s, *, date_from, date_to, kind: str = "expense") -> list[dict]:
    from app.modules.categories.models import Category

    sign = Transaction.amount < 0 if kind == "expense" else Transaction.amount > 0

    rows = s.execute(
        select(Category.id, Category.name, Category.emoji, Category.color,
               func.sum(Transaction.amount).label("total"))
        .join(Category, Category.id == Transaction.category_id)
        .where(Transaction.transfer_group_id.is_(None),
               Transaction.date >= date_from,
               Transaction.date <= date_to,
               sign,
               Category.kind == kind)
        .group_by(Category.id, Category.name, Category.emoji, Category.color)
        .order_by(func.min(Transaction.amount))
    ).all()

    result = [{"category_id": r.id, "name": r.name, "emoji": r.emoji,
               "color": r.color, "total": abs(int(r.total))} for r in rows]

    uncategorised = s.execute(
        select(func.coalesce(func.sum(Transaction.amount), 0))
        .where(Transaction.transfer_group_id.is_(None),
               Transaction.date >= date_from,
               Transaction.date <= date_to,
               sign,
               Transaction.category_id.is_(None))
    ).scalar_one()
    if uncategorised:
        result.append({**UNCATEGORISED, "total": abs(int(uncategorised))})

    result.sort(key=lambda r: r["total"], reverse=True)
    return result
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && .venv/bin/pytest tests/modules/reports/test_reports.py -v`
Expected: 8 passed

- [ ] **Step 5: Commit**

```bash
git add backend/app/modules/reports backend/tests/modules/reports
git commit -m "feat: reports service with transfer-neutral net worth and cash flow"
```

---

### Task 29: Reports API

**Files:**
- Create: `backend/app/modules/reports/schemas.py`, `router.py`
- Modify: `backend/app/main.py`
- Test: `backend/tests/modules/reports/test_reports_api.py`

**Interfaces:**
- Consumes: `service`, `current_member`.
- Produces routes:
  - `GET /api/reports/net-worth` → `NetWorthOut`
  - `GET /api/reports/cash-flow?date_from&date_to` → `CashFlowOut`
  - `GET /api/reports/by-category?date_from&date_to&kind` → `list[CategorySpendOut]`
  - `GET /api/reports/dashboard` → `DashboardOut` (net worth + this month's cash flow + top 5 categories), so the dashboard is one request

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/modules/reports/test_reports_api.py

def _setup(client):
    client.post("/api/setup", json={"name": "Budi", "username": "budi",
                                    "password": "supersecret123"})
    acc = client.post("/api/accounts", json={"name": "BCA", "type": "bank",
                                             "opening_balance": 1000000}).json()["id"]
    cash = client.post("/api/accounts", json={"name": "Tunai", "type": "cash",
                                              "opening_balance": 200000}).json()["id"]
    cat = client.post("/api/categories", json={"name": "Makanan", "kind": "expense",
                                              "emoji": "🍚"}).json()["id"]
    client.post("/api/transfers", json={"from_account_id": acc, "to_account_id": cash,
                                        "date": "2026-02-05", "amount": 500000,
                                        "description": "Tarik tunai"})
    client.post("/api/transactions", json={"account_id": acc, "date": "2026-02-06",
                                           "amount": -186000, "description": "Superindo",
                                           "category_id": cat})
    return acc, cash, cat


def test_net_worth_endpoint_is_transfer_neutral(client):
    _setup(client)
    body = client.get("/api/reports/net-worth").json()
    assert body["total"] == 1200000 - 186000
    balances = {a["name"]: a["balance"] for a in body["accounts"]}
    assert balances == {"BCA": 1314000, "Tunai": 700000}


def test_cash_flow_endpoint(client):
    _setup(client)
    body = client.get("/api/reports/cash-flow?date_from=2026-02-01&date_to=2026-02-28").json()
    assert body == {"income": 0, "expense": 186000, "net": -186000}


def test_by_category_endpoint(client):
    _setup(client)
    rows = client.get("/api/reports/by-category?date_from=2026-02-01&date_to=2026-02-28").json()
    assert rows[0]["name"] == "Makanan"
    assert rows[0]["total"] == 186000


def test_dashboard_endpoint_returns_everything_the_home_screen_needs(client):
    _setup(client)
    body = client.get("/api/reports/dashboard?month=2026-02").json()
    assert body["net_worth"]["total"] == 1200000 - 186000
    assert body["cash_flow"]["expense"] == 186000
    assert body["top_categories"][0]["name"] == "Makanan"
    assert body["month"] == "2026-02"


def test_reports_require_auth(client):
    assert client.get("/api/reports/net-worth").status_code == 401
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && .venv/bin/pytest tests/modules/reports/test_reports_api.py -v`
Expected: FAIL — 404

- [ ] **Step 3: Write `modules/reports/schemas.py`**

```python
from pydantic import BaseModel


class AccountBalanceOut(BaseModel):
    id: int
    name: str
    type: str
    balance: int


class NetWorthOut(BaseModel):
    total: int
    accounts: list[AccountBalanceOut]


class CashFlowOut(BaseModel):
    income: int
    expense: int
    net: int


class CategorySpendOut(BaseModel):
    category_id: int | None
    name: str
    emoji: str
    color: str
    total: int


class DashboardOut(BaseModel):
    month: str
    net_worth: NetWorthOut
    cash_flow: CashFlowOut
    top_categories: list[CategorySpendOut]
```

- [ ] **Step 4: Write `modules/reports/router.py`**

```python
from calendar import monthrange
from datetime import date

from fastapi import APIRouter, Query, Request

from app.core.deps import current_member
from app.modules.reports import service
from app.modules.reports.schemas import (CashFlowOut, CategorySpendOut, DashboardOut,
                                        NetWorthOut)

router = APIRouter(prefix="/api/reports", tags=["reports"])


def _db(request):
    return request.app.state.db


def _month_bounds(month: str) -> tuple[date, date]:
    year, mon = (int(part) for part in month.split("-"))
    return date(year, mon, 1), date(year, mon, monthrange(year, mon)[1])


@router.get("/net-worth", response_model=NetWorthOut)
def net_worth(request: Request):
    current_member(request)
    with _db(request).session() as s:
        return service.net_worth(s)


@router.get("/cash-flow", response_model=CashFlowOut)
def cash_flow(request: Request, date_from: date, date_to: date):
    current_member(request)
    with _db(request).session() as s:
        return service.cash_flow(s, date_from=date_from, date_to=date_to)


@router.get("/by-category", response_model=list[CategorySpendOut])
def by_category(request: Request, date_from: date, date_to: date,
                kind: str = Query("expense", pattern="^(income|expense)$")):
    current_member(request)
    with _db(request).session() as s:
        return service.spend_by_category(s, date_from=date_from, date_to=date_to, kind=kind)


@router.get("/dashboard", response_model=DashboardOut)
def dashboard(request: Request, month: str | None = None,
              top: int = Query(5, ge=1, le=20)):
    current_member(request)
    month = month or date.today().strftime("%Y-%m")
    start, end = _month_bounds(month)
    with _db(request).session() as s:
        return DashboardOut(
            month=month,
            net_worth=service.net_worth(s),
            cash_flow=service.cash_flow(s, date_from=start, date_to=end),
            top_categories=service.spend_by_category(
                s, date_from=start, date_to=end)[:top],
        )
```

- [ ] **Step 5: Include the router and run the suite**

Add to `main.py`: `from app.modules.reports.router import router as reports_router` and `app.include_router(reports_router)`.

Run: `cd backend && .venv/bin/pytest -v`
Expected: all passed

- [ ] **Step 6: Commit**

```bash
git add backend/app/modules/reports backend/app/main.py backend/tests/modules/reports/test_reports_api.py
git commit -m "feat: reports api and single-request dashboard endpoint"
```

---

# Phase 7 — UI (Family Play)

> v1 ships **no frontend unit tests** (per spec §8). Each task's verification is a concrete browser check against the dev server. Slice 2 adds three Playwright smoke tests.

### Task 30: Frontend foundation — API client, session, route guard, app shell

**Files:**
- Create: `frontend/composables/useApi.ts`
- Create: `frontend/composables/useSession.ts`
- Create: `frontend/middleware/auth.global.ts`
- Create: `frontend/layouts/default.vue`
- Create: `frontend/components/AppShell.vue`
- Modify: `frontend/nuxt.config.ts`

**Interfaces:**
- Produces:
  - `useApi()` → `{ get, post, patch, del, upload }`; every call sends `credentials: 'include'`, sets/requires `Content-Type: application/json`, and throws `ApiError{code,message,details}` built from the server envelope.
  - `useSession()` → `{ member, needsSetup, refresh(), login(), logout(), setup() }`.
  - Route guard: unauthenticated → `/login`; `needs_setup` → `/setup`.

- [ ] **Step 1: Write `composables/useApi.ts`**

```ts
export class ApiError extends Error {
  code: string
  details: any
  constructor(code: string, message: string, details: any = null) {
    super(message)
    this.code = code
    this.details = details
  }
}

function envelope(body: any, status: number) {
  const err = body?.error
  if (err) return new ApiError(err.code ?? 'error', err.message ?? 'Terjadi kesalahan', err.details)
  return new ApiError('http_error', `Permintaan gagal (${status})`)
}

export function useApi() {
  const config = useRuntimeConfig()
  const base = config.public.apiBase as string

  async function request<T>(method: string, path: string, options: {
    body?: any, form?: FormData, query?: Record<string, any>
  } = {}): Promise<T> {
    const init: RequestInit = { method, credentials: 'include', headers: {} as Record<string, string> }
    if (options.body !== undefined) {
      ;(init.headers as Record<string, string>)['Content-Type'] = 'application/json'
      init.body = JSON.stringify(options.body)
    }
    if (options.form) {
      init.body = options.form          // browser sets multipart boundary; never set Content-Type here
    }
    const qs = options.query
      ? '?' + new URLSearchParams(
          Object.entries(options.query)
            .filter(([, v]) => v !== undefined && v !== null && v !== '')
            .map(([k, v]) => [k, String(v)])
        ).toString()
      : ''
    const res = await fetch(`${base}${path}${qs}`, init)
    const text = await res.text()
    const data = text ? JSON.parse(text) : null
    if (!res.ok) throw envelope(data, res.status)
    return data as T
  }

  return {
    get: <T>(p: string, query?: Record<string, any>) => request<T>('GET', p, { query }),
    post: <T>(p: string, body?: any) => request<T>('POST', p, { body }),
    patch: <T>(p: string, body?: any) => request<T>('PATCH', p, { body }),
    del: <T>(p: string) => request<T>('DELETE', p),
    upload: <T>(p: string, form: FormData) => request<T>('POST', p, { form }),
  }
}
```

- [ ] **Step 2: Write `composables/useSession.ts`**

```ts
interface Member { id: number, name: string, username: string, role: string, avatar_color: string }

export function useSession() {
  const api = useApi()
  const member = useState<Member | null>('member', () => null)
  const needsSetup = useState<boolean>('needsSetup', () => false)
  const ready = useState<boolean>('sessionReady', () => false)

  async function refresh() {
    const state = await api.get<{ needs_setup: boolean, member: Member | null }>('/api/auth/state')
    needsSetup.value = state.needs_setup
    member.value = state.member
    ready.value = true
  }

  async function setup(payload: { name: string, username: string, password: string, avatar_color?: string }) {
    member.value = await api.post<Member>('/api/setup', payload)
    needsSetup.value = false
  }

  async function login(username: string, password: string) {
    member.value = await api.post<Member>('/api/auth/login', { username, password })
  }

  async function logout() {
    await api.post('/api/auth/logout')
    member.value = null
    await navigateTo('/login')
  }

  return { member, needsSetup, ready, refresh, setup, login, logout }
}
```

- [ ] **Step 3: Write `middleware/auth.global.ts`**

```ts
export default defineNuxtRouteMiddleware(async (to) => {
  if (import.meta.server) return
  const { needsSetup, member, ready, refresh } = useSession()
  if (!ready.value) {
    try { await refresh() } catch { /* offline: fall through to login */ }
  }
  if (needsSetup.value) return to.path === '/setup' ? undefined : navigateTo('/setup')
  if (!member.value) return to.path === '/login' ? undefined : navigateTo('/login')
  if (to.path === '/login' || to.path === '/setup') return navigateTo('/')
})
```

- [ ] **Step 4: Write `components/AppShell.vue` and `layouts/default.vue`**

`AppShell.vue` renders the household name + member avatar, a desktop sidebar (`hidden md:flex`), a mobile bottom nav (`md:hidden`), and a slot. Nav items: Dashboard `/`, Transaksi `/transactions`, Tambah (opens the add modal), Laporan `/reports`, Akun `/accounts`.

```vue
<script setup lang="ts">
const { member, logout } = useSession()
const api = useApi()
const route = useRoute()
const household = ref('Keluarga Kita')

onMounted(async () => {
  try {
    const s = await api.get<{ household_name?: string }>('/api/settings')
    if (s.household_name) household.value = s.household_name
  } catch { /* settings are cosmetic; ignore */ }
})

const nav = [
  { to: '/', label: 'Home', icon: '🏠' },
  { to: '/transactions', label: 'Transaksi', icon: '≡' },
  { to: '/accounts', label: 'Akun', icon: '◫' },
  { to: '/reports', label: 'Laporan', icon: '◔' },
]
</script>

<template>
  <div class="min-h-screen bg-paper">
    <header class="border-brutal border-t-0 border-l-0 border-r-0 bg-hi px-4 py-3 flex items-center gap-3">
      <div class="w-7 h-7 rounded-brutal bg-info border-brutal"></div>
      <span class="font-extrabold">{{ household }}</span>
      <button
        class="ml-auto w-8 h-8 rounded-full border-brutal font-extrabold text-white"
        :style="{ background: member?.avatar_color }"
        :title="member?.name"
        @click="logout"
      >{{ member?.name?.[0] }}</button>
    </header>

    <div class="flex">
      <nav class="hidden md:flex flex-col gap-2 p-4 w-56">
        <NuxtLink
          v-for="item in nav" :key="item.to" :to="item.to"
          class="border-brutal shadow-brutal rounded-brutal bg-white px-3 py-2 font-extrabold active:press"
          :class="route.path === item.to ? 'bg-in' : ''"
        >{{ item.icon }} {{ item.label }}</NuxtLink>
      </nav>
      <main class="flex-1 p-4 pb-24 md:pb-4"><slot /></main>
    </div>

    <nav class="md:hidden fixed bottom-0 inset-x-0 border-brutal border-b-0 border-l-0 border-r-0 bg-ink flex">
      <NuxtLink
        v-for="item in nav" :key="item.to" :to="item.to"
        class="flex-1 text-center py-3 text-xs font-extrabold"
        :class="route.path === item.to ? 'text-in' : 'text-muted'"
      >{{ item.icon }}<br>{{ item.label }}</NuxtLink>
    </nav>
  </div>
</template>
```

`layouts/default.vue`:

```vue
<template><AppShell><slot /></AppShell></template>
```

- [ ] **Step 5: Expose config in `nuxt.config.ts`**

```ts
export default defineNuxtConfig({
  ssr: false,
  modules: ['@nuxtjs/tailwindcss'],
  css: ['~/assets/css/main.css'],
  devtools: { enabled: false },
  runtimeConfig: { public: { apiBase: '' } },
  app: {
    head: {
      title: 'Keluarga Kita',
      htmlAttrs: { lang: 'id' },
      meta: [{ name: 'viewport', content: 'width=device-width, initial-scale=1' }]
    }
  }
})
```

`apiBase: ''` means same-origin — correct for both `npm run dev` behind a proxy and the Docker image. Start the dev server with the API proxied:

```ts
  nitro: { devProxy: { '/api': { target: 'http://127.0.0.1:8000/api', changeOrigin: true } } },
```

- [ ] **Step 6: Verify**

Run: `cd backend && .venv/bin/uvicorn app.main:app --port 8000` then `cd frontend && npm run dev`
Expected: with no members, `http://localhost:3000/` redirects to `/setup`. `curl -s localhost:3000/api/health` proxies to the API and returns `{"status":"ok"}`.

- [ ] **Step 7: Commit**

```bash
git add frontend/composables frontend/middleware frontend/layouts frontend/components/AppShell.vue frontend/nuxt.config.ts
git commit -m "feat: frontend api client, session state, route guard, and app shell"
```

---

### Task 31: Setup and login screens

**Files:**
- Create: `frontend/pages/setup.vue`, `frontend/pages/login.vue`

- [ ] **Step 1: Write `pages/setup.vue`**

```vue
<script setup lang="ts">
definePageMeta({ layout: false })
const { setup } = useSession()
const form = reactive({ name: '', username: '', password: '', avatar_color: '#EF476F' })
const error = ref('')
const busy = ref(false)

async function submit() {
  error.value = ''; busy.value = true
  try {
    await setup({ ...form, username: form.username.toLowerCase().trim() })
    await navigateTo('/')
  } catch (e: any) {
    error.value = e.message
  } finally { busy.value = false }
}
</script>

<template>
  <div class="min-h-screen bg-paper flex items-center justify-center p-4">
    <form class="w-full max-w-md border-brutal shadow-brutal rounded-brutal bg-white p-6" @submit.prevent="submit">
      <h1 class="text-2xl font-extrabold mb-1">Selamat datang</h1>
      <p class="text-sm text-muted font-bold mb-5">Buat akun pertama untuk rumah tangga Anda.</p>

      <label class="block font-extrabold text-sm mb-1">Nama</label>
      <input v-model="form.name" required class="w-full border-brutal rounded-brutal px-3 py-2 mb-4" placeholder="Budi">

      <label class="block font-extrabold text-sm mb-1">Nama pengguna</label>
      <input v-model="form.username" required minlength="3" class="w-full border-brutal rounded-brutal px-3 py-2 mb-4" placeholder="budi">

      <label class="block font-extrabold text-sm mb-1">Kata sandi</label>
      <input v-model="form.password" type="password" required minlength="12" class="w-full border-brutal rounded-brutal px-3 py-2 mb-1">
      <p class="text-xs text-muted font-bold mb-4">Minimal 12 karakter.</p>

      <label class="block font-extrabold text-sm mb-1">Warna avatar</label>
      <input v-model="form.avatar_color" type="color" class="h-10 w-20 border-brutal rounded-brutal mb-5">

      <p v-if="error" class="border-brutal rounded-brutal bg-out text-ink font-extrabold p-2 mb-4">{{ error }}</p>

      <button :disabled="busy" class="w-full border-brutal shadow-brutal rounded-brutal bg-in font-extrabold py-3 active:press">
        {{ busy ? 'Menyimpan…' : 'Buat akun' }}
      </button>
    </form>
  </div>
</template>
```

The `bg-out` swatch carries `text-ink`, matching the accessibility rule (ink on colour, never white on colour).

- [ ] **Step 2: Write `pages/login.vue`**

Avatar-tile picker then password — so a returning member taps their face instead of typing a username.

```vue
<script setup lang="ts">
definePageMeta({ layout: false })
const { login } = useSession()
const api = useApi()
const members = ref<{ id: number, name: string, username: string, avatar_color: string }[]>([])
const chosen = ref<typeof members.value[number] | null>(null)
const password = ref('')
const error = ref('')
const busy = ref(false)

onMounted(async () => {
  try { members.value = await api.get('/api/members') } catch { /* state endpoint may be the only one open */ }
})

async function submit() {
  if (!chosen.value) return
  error.value = ''; busy.value = true
  try {
    await login(chosen.value.username, password.value)
    await navigateTo('/')
  } catch (e: any) {
    error.value = e.message
  } finally { busy.value = false }
}
</script>

<template>
  <div class="min-h-screen bg-paper flex items-center justify-center p-4">
    <div class="w-full max-w-md border-brutal shadow-brutal rounded-brutal bg-white p-6">
      <h1 class="text-2xl font-extrabold mb-5">Masuk</h1>

      <div v-if="!chosen" class="flex flex-wrap gap-3">
        <button
          v-for="m in members" :key="m.id"
          class="flex flex-col items-center gap-2 border-brutal shadow-brutal rounded-brutal bg-white p-3 active:press"
          @click="chosen = m"
        >
          <span class="w-12 h-12 rounded-full border-brutal flex items-center justify-center font-extrabold text-white"
                :style="{ background: m.avatar_color }">{{ m.name[0] }}</span>
          <span class="font-extrabold text-sm">{{ m.name }}</span>
        </button>
        <p v-if="!members.length" class="text-sm text-muted font-bold">Memuat…</p>
      </div>

      <form v-else @submit.prevent="submit">
        <div class="flex items-center gap-3 mb-4">
          <span class="w-12 h-12 rounded-full border-brutal flex items-center justify-center font-extrabold text-white"
                :style="{ background: chosen.avatar_color }">{{ chosen.name[0] }}</span>
          <span class="font-extrabold">{{ chosen.name }}</span>
          <button type="button" class="ml-auto text-sm font-extrabold underline" @click="chosen = null">Ganti</button>
        </div>
        <input v-model="password" type="password" required autofocus
               class="w-full border-brutal rounded-brutal px-3 py-2 mb-4" placeholder="Kata sandi">
        <p v-if="error" class="border-brutal rounded-brutal bg-out text-ink font-extrabold p-2 mb-4">{{ error }}</p>
        <button :disabled="busy" class="w-full border-brutal shadow-brutal rounded-brutal bg-in font-extrabold py-3 active:press">
          {{ busy ? 'Memeriksa…' : 'Masuk' }}
        </button>
      </form>
    </div>
  </div>
</template>
```

- [ ] **Step 3: Verify**

Run both servers. Expected: `/setup` creates an account and lands on `/`. Log out → avatar tiles appear → tapping one reveals the password field → wrong password shows the error, correct password lands on `/`.

- [ ] **Step 4: Commit**

```bash
git add frontend/pages/setup.vue frontend/pages/login.vue
git commit -m "feat: setup wizard and avatar-tile login screens"
```

---

### Task 32: Shared components and dashboard

**Files:**
- Create: `frontend/components/MoneyText.vue`, `CategoryChip.vue`, `MemberAvatar.vue`, `ProgressBar.vue`, `TransactionRow.vue`
- Create: `frontend/pages/index.vue`

- [ ] **Step 1: Write `components/MoneyText.vue`**

The single place money is formatted. Tabular numerals so columns never shift.

```vue
<script setup lang="ts">
const props = defineProps<{ amount: number, signed?: boolean, size?: 'sm' | 'md' | 'lg' | 'xl' }>()
const text = computed(() => {
  const formatted = new Intl.NumberFormat('id-ID').format(Math.abs(props.amount))
  if (!props.signed) return formatted
  return (props.amount < 0 ? '−' : '+') + ' ' + formatted
})
const colour = computed(() => (props.amount < 0 ? 'text-out' : 'text-in'))
const sizes = { sm: 'text-xs', md: 'text-sm', lg: 'text-lg', xl: 'text-3xl' }
</script>

<template>
  <span class="money" :class="[sizes[size ?? 'md'], signed ? colour : '']">{{ text }}</span>
</template>
```

- [ ] **Step 2: Write the remaining shared components**

`CategoryChip.vue` — emoji + name pill; dashed and muted when `category` is null:

```vue
<script setup lang="ts">
const props = defineProps<{ category?: { name: string, emoji: string, color: string } | null, clickable?: boolean }>()
const emit = defineEmits<{ click: [] }>()
</script>

<template>
  <button
    type="button" :disabled="!clickable" @click="emit('click')"
    class="border-brutal rounded-full px-3 py-1 font-extrabold text-xs whitespace-nowrap"
    :class="props.category ? 'bg-white' : 'bg-white border-dashed text-info'"
  >
    <template v-if="props.category">{{ props.category.emoji }} {{ props.category.name }}</template>
    <template v-else>❓ belum dikategorikan</template>
  </button>
</template>
```

`MemberAvatar.vue` — circle initial, colour from the member, `+` dashed when no member:

```vue
<script setup lang="ts">
const props = defineProps<{ member?: { name: string, avatar_color: string } | null, size?: number }>()
</script>

<template>
  <span
    class="rounded-full border-brutal inline-flex items-center justify-center font-extrabold"
    :class="props.member ? 'text-white' : 'border-dashed text-muted'"
    :style="{ width: (size ?? 32) + 'px', height: (size ?? 32) + 'px',
              background: props.member?.avatar_color ?? 'white' }"
  >{{ props.member ? props.member.name[0] : '—' }}</span>
</template>
```

`ProgressBar.vue` — the room-readable bar. Turns red when over the limit:

```vue
<script setup lang="ts">
const props = defineProps<{ value: number, max: number, emoji?: string, label?: string }>()
const pct = computed(() => (props.max <= 0 ? 0 : Math.min(100, Math.round((props.value / props.max) * 100))))
const over = computed(() => props.value > props.max)
</script>

<template>
  <div>
    <div class="flex justify-between font-extrabold text-sm mb-1">
      <span>{{ emoji }} {{ label }}</span>
      <MoneyText :amount="value" size="sm" />
    </div>
    <div class="h-3 rounded-brutal border-brutal bg-white overflow-hidden">
      <div class="h-full" :class="over ? 'bg-out' : 'bg-in'" :style="{ width: pct + '%' }"></div>
    </div>
  </div>
</template>
```

`TransactionRow.vue` — one card per transaction; dashed when uncategorised:

```vue
<script setup lang="ts">
defineProps<{
  tx: { id: number, date: string, amount: number, description: string,
        category?: { name: string, emoji: string, color: string } | null,
        account_name?: string, paid_by_name?: string }
}>()
defineEmits<{ select: [id: number] }>()
</script>

<template>
  <button
    class="w-full text-left border-brutal shadow-brutal rounded-brutal bg-white p-3 flex items-center gap-3 active:press"
    :class="tx.category ? '' : 'bg-white border-dashed'"
    @click="$emit('select', tx.id)"
  >
    <span class="w-9 h-9 rounded-brutal border-brutal flex items-center justify-center text-lg"
          :style="{ background: tx.category?.color ?? 'white' }">{{ tx.category?.emoji ?? '❓' }}</span>
    <span class="flex-1 min-w-0">
      <span class="block font-extrabold truncate">{{ tx.description || 'Tanpa keterangan' }}</span>
      <span class="block text-xs font-bold text-muted">
        {{ tx.account_name }}<template v-if="tx.paid_by_name"> · oleh {{ tx.paid_by_name }}</template>
      </span>
    </span>
    <MoneyText :amount="tx.amount" signed size="md" />
  </button>
</template>
```

- [ ] **Step 3: Write `pages/index.vue` — the dashboard**

Huge net worth, this month in/out, top spending bars, and a prominent import button.

```vue
<script setup lang="ts">
const api = useApi()

interface Dashboard {
  month: string
  net_worth: { total: number, accounts: { id: number, name: string, type: string, balance: number }[] }
  cash_flow: { income: number, expense: number, net: number }
  top_categories: { category_id: number | null, name: string, emoji: string, color: string, total: number }[]
}

const month = ref(new Date().toISOString().slice(0, 7))
const data = ref<Dashboard | null>(null)
const { member } = useSession()

async function load() {
  data.value = await api.get<Dashboard>('/api/reports/dashboard', { month: month.value })
}

onMounted(load)
watch(month, load)

const monthLabel = computed(() => {
  if (!month.value) return ''
  const [y, m] = month.value.split('-').map(Number)
  return new Date(y, m - 1, 1).toLocaleDateString('id-ID', { month: 'long', year: 'numeric' })
})
</script>

<template>
  <div v-if="data" class="max-w-3xl mx-auto space-y-4">
    <div class="flex items-center gap-2">
      <h1 class="text-xl font-extrabold">Hai, {{ member?.name }} 👋</h1>
      <input v-model="month" type="month" class="ml-auto border-brutal rounded-brutal bg-hi px-2 py-1 font-extrabold text-sm">
    </div>

    <section class="border-brutal shadow-brutal rounded-brutal bg-white p-5">
      <p class="text-xs font-extrabold tracking-wide text-muted">TOTAL RUMAH TANGGA</p>
      <MoneyText :amount="data.net_worth.total" size="xl" />
      <div class="flex gap-3 mt-4">
        <div class="flex-1 border-brutal shadow-brutal rounded-brutal bg-in p-3">
          <p class="text-xs font-extrabold">MASUK</p>
          <MoneyText :amount="data.cash_flow.income" size="lg" />
        </div>
        <div class="flex-1 border-brutal shadow-brutal rounded-brutal bg-out text-ink p-3">
          <p class="text-xs font-extrabold">KELUAR</p>
          <MoneyText :amount="data.cash_flow.expense" size="lg" />
        </div>
      </div>
    </section>

    <section class="border-brutal shadow-brutal rounded-brutal bg-white p-5">
      <div class="flex items-center mb-4">
        <h2 class="font-extrabold">{{ monthLabel }}</h2>
        <NuxtLink to="/reports" class="ml-auto text-sm font-extrabold underline">Semua laporan</NuxtLink>
      </div>
      <p v-if="!data.top_categories.length" class="text-sm font-bold text-muted">
        Belum ada pengeluaran bulan ini.
      </p>
      <div v-else class="space-y-3">
        <ProgressBar
          v-for="c in data.top_categories" :key="c.name"
          :value="c.total" :max="data.top_categories[0].total"
          :emoji="c.emoji" :label="c.name"
        />
      </div>
    </section>

    <section class="grid grid-cols-2 gap-3">
      <NuxtLink to="/transactions" class="border-brutal shadow-brutal rounded-brutal bg-hi p-4 text-center font-extrabold active:press">
        ＋ Catat pengeluaran
      </NuxtLink>
      <NuxtLink to="/import" class="border-brutal shadow-brutal rounded-brutal bg-info text-ink p-4 text-center font-extrabold active:press">
        📥 Impor CSV bank
      </NuxtLink>
    </section>

    <section class="border-brutal shadow-brutal rounded-brutal bg-white p-5">
      <h2 class="font-extrabold mb-3">Akun</h2>
      <div class="flex flex-wrap gap-3">
        <NuxtLink
          v-for="a in data.net_worth.accounts" :key="a.id" :to="`/accounts/${a.id}`"
          class="flex-1 min-w-[45%] border-brutal shadow-brutal rounded-brutal bg-white p-3 active:press"
        >
          <p class="text-xs font-bold text-muted">{{ a.name }}</p>
          <MoneyText :amount="a.balance" size="lg" />
        </NuxtLink>
      </div>
    </section>
  </div>
  <p v-else class="font-bold text-muted">Memuat…</p>
</template>
```

Note `bg-out` is paired with `text-ink` on the KELUAR card — never white on pink (spec §7 accessibility).

- [ ] **Step 4: Verify**

Run both servers with a few transactions. Expected: net worth is large and readable; transfers do not move the total; an uncategorised transaction shows a dashed row and an "belum dikategorikan" entry; narrowing the month input changes the bars.

- [ ] **Step 5: Commit**

```bash
git add frontend/components frontend/pages/index.vue
git commit -m "feat: money, chip, avatar, progress, row components and dashboard"
```

---

### Task 33: Transactions screen with add/edit modal and transfer

**Files:**
- Create: `frontend/components/TransactionModal.vue`
- Create: `frontend/pages/transactions/index.vue`

- [ ] **Step 1: Write `components/TransactionModal.vue`**

Big fields, a built-in calculator, emoji category picker, "who paid" and "for whom" tags, and a transfer toggle.

```vue
<script setup lang="ts">
const props = defineProps<{
  open: boolean
  tx?: any | null
  accounts: { id: number, name: string }[]
  categories: any[]
  members: { id: number, name: string, avatar_color: string }[]
}>()
const emit = defineEmits<{ close: [], saved: [] }>()

const api = useApi()
const mode = ref<'expense' | 'income' | 'transfer'>('expense')
const form = reactive({
  account_id: null as number | null,
  to_account_id: null as number | null,
  date: new Date().toISOString().slice(0, 10),
  amount: '',
  description: '',
  category_id: null as number | null,
  paid_by: null as number | null,
  for_whom: null as number | null,
  note: '',
})
const error = ref('')
const busy = ref(false)

watch(() => props.open, (isOpen) => {
  if (!isOpen) return
  error.value = ''
  if (props.tx) {
    Object.assign(form, {
      account_id: props.tx.account_id, date: props.tx.date,
      amount: String(Math.abs(props.tx.amount)), description: props.tx.description,
      category_id: props.tx.category_id, paid_by: props.tx.paid_by,
      for_whom: props.tx.for_whom, note: props.tx.note ?? '',
    })
    mode.value = props.tx.transfer_group_id ? 'transfer' : (props.tx.amount > 0 ? 'income' : 'expense')
  } else {
    Object.assign(form, { account_id: props.accounts[0]?.id ?? null, to_account_id: null,
      date: new Date().toISOString().slice(0, 10), amount: '', description: '',
      category_id: null, paid_by: null, for_whom: null, note: '' })
    mode.value = 'expense'
  }
})

const visibleCategories = computed(() =>
  props.categories.filter(c => c.kind === (mode.value === 'income' ? 'income' : 'expense')))

const signedAmount = computed(() => {
  const n = Number(String(form.amount).replace(/[^\d]/g, ''))
  if (!n) return 0
  return mode.value === 'income' ? n : -n
})

async function save() {
  error.value = ''; busy.value = true
  try {
    if (mode.value === 'transfer') {
      await api.post('/api/transfers', {
        from_account_id: form.account_id, to_account_id: form.to_account_id,
        date: form.date, amount: Math.abs(signedAmount.value),
        description: form.description, paid_by: form.paid_by, note: form.note || null,
      })
    } else if (props.tx) {
      await api.patch(`/api/transactions/${props.tx.id}`, {
        account_id: form.account_id, date: form.date, amount: signedAmount.value,
        description: form.description, category_id: form.category_id,
        paid_by: form.paid_by, for_whom: form.for_whom, note: form.note || null,
      })
    } else {
      await api.post('/api/transactions', {
        account_id: form.account_id, date: form.date, amount: signedAmount.value,
        description: form.description, category_id: form.category_id,
        paid_by: form.paid_by, for_whom: form.for_whom, note: form.note || null,
      })
    }
    emit('saved')
  } catch (e: any) {
    error.value = e.message
  } finally { busy.value = false }
}

async function remove() {
  if (!props.tx || !confirm('Hapus transaksi ini?')) return
  await api.del(`/api/transactions/${props.tx.id}`)
  emit('saved')
}
</script>

<template>
  <div v-if="open" class="fixed inset-0 bg-ink/40 flex items-end md:items-center justify-center p-0 md:p-4 z-50" @click.self="emit('close')">
    <form class="w-full max-w-lg border-brutal shadow-brutal rounded-brutal bg-paper p-5 max-h-[92vh] overflow-y-auto"
          @submit.prevent="save">
      <div class="flex gap-2 mb-5">
        <button v-for="m in (['expense','income','transfer'] as const)" :key="m" type="button"
                class="flex-1 border-brutal rounded-brutal py-2 font-extrabold text-sm"
                :class="mode === m ? 'bg-hi shadow-brutal' : 'bg-white'"
                @click="mode = m">
          {{ m === 'expense' ? 'Pengeluaran' : m === 'income' ? 'Pemasukan' : 'Transfer' }}
        </button>
      </div>

      <label class="block font-extrabold text-sm mb-1">Jumlah</label>
      <input v-model="form.amount" inputmode="numeric" required
             class="w-full border-brutal rounded-brutal px-3 py-3 mb-4 money text-2xl" placeholder="0">

      <label class="block font-extrabold text-sm mb-1">Tanggal</label>
      <input v-model="form.date" type="date" required class="w-full border-brutal rounded-brutal px-3 py-2 mb-4">

      <label class="block font-extrabold text-sm mb-1">{{ mode === 'transfer' ? 'Dari akun' : 'Akun' }}</label>
      <select v-model="form.account_id" required class="w-full border-brutal rounded-brutal px-3 py-2 mb-4 bg-white">
        <option v-for="a in accounts" :key="a.id" :value="a.id">{{ a.name }}</option>
      </select>

      <template v-if="mode === 'transfer'">
        <label class="block font-extrabold text-sm mb-1">Ke akun</label>
        <select v-model="form.to_account_id" required class="w-full border-brutal rounded-brutal px-3 py-2 mb-4 bg-white">
          <option :value="null" disabled>Pilih akun tujuan</option>
          <option v-for="a in accounts.filter(a => a.id !== form.account_id)" :key="a.id" :value="a.id">{{ a.name }}</option>
        </select>
      </template>

      <label class="block font-extrabold text-sm mb-1">Keterangan</label>
      <input v-model="form.description" class="w-full border-brutal rounded-brutal px-3 py-2 mb-4" placeholder="Superindo">

      <template v-if="mode !== 'transfer'">
        <label class="block font-extrabold text-sm mb-1">Kategori</label>
        <div class="flex flex-wrap gap-2 mb-4">
          <CategoryChip v-for="c in visibleCategories" :key="c.id"
                        :category="c" clickable
                        :class="form.category_id === c.id ? 'bg-hi' : ''"
                        @click="form.category_id = form.category_id === c.id ? null : c.id" />
          <NuxtLink to="/categories" class="border-brutal rounded-full px-3 py-1 font-extrabold text-xs bg-white">＋ baru</NuxtLink>
        </div>
      </template>

      <template v-if="mode !== 'transfer'">
        <label class="block font-extrabold text-sm mb-1">Siapa yang bayar</label>
        <div class="flex flex-wrap gap-2 mb-4">
          <MemberAvatar v-for="m in members" :key="m.id" :member="m" :size="34"
                        :class="form.paid_by === m.id ? 'shadow-brutal' : ''"
                        @click="form.paid_by = form.paid_by === m.id ? null : m.id" />
        </div>

        <label class="block font-extrabold text-sm mb-1">Untuk siapa</label>
        <div class="flex flex-wrap gap-2 mb-4">
          <MemberAvatar v-for="m in members" :key="m.id" :member="m" :size="34"
                        :class="form.for_whom === m.id ? 'shadow-brutal' : ''"
                        @click="form.for_whom = form.for_whom === m.id ? null : m.id" />
        </div>
      </template>

      <p v-if="error" class="border-brutal rounded-brutal bg-out text-ink font-extrabold p-2 mb-4">{{ error }}</p>

      <div class="flex gap-2">
        <button type="button" class="border-brutal shadow-brutal rounded-brutal bg-white font-extrabold px-4 py-3 active:press"
                @click="emit('close')">Batal</button>
        <button v-if="tx" type="button" class="border-brutal shadow-brutal rounded-brutal bg-out text-ink font-extrabold px-4 py-3 active:press"
                @click="remove">Hapus</button>
        <button :disabled="busy" class="flex-1 border-brutal shadow-brutal rounded-brutal bg-in font-extrabold py-3 active:press">
          {{ busy ? 'Menyimpan…' : 'Simpan' }}
        </button>
      </div>
    </form>
  </div>
</template>
```

- [ ] **Step 2: Write `pages/transactions/index.vue`**

```vue
<script setup lang="ts">
const api = useApi()
const { member } = useSession()

const filters = reactive({ account_id: '' as string | number, category_id: '' as string | number,
                           member_id: '' as string | number, uncategorized: false,
                           search: '', month: new Date().toISOString().slice(0, 7) })
const page = reactive({ limit: 50, offset: 0 })
const rows = ref<any[]>([])
const total = ref(0)
const accounts = ref<any[]>([])
const categories = ref<any[]>([])
const members = ref<any[]>([])
const modalOpen = ref(false)
const editing = ref<any | null>(null)
const loading = ref(false)

const monthBounds = computed(() => {
  const [y, m] = filters.month.split('-').map(Number)
  const last = new Date(y, m, 0).getDate()
  return { date_from: `${filters.month}-01`, date_to: `${filters.month}-${String(last).padStart(2, '0')}` }
})

async function load() {
  loading.value = true
  try {
    const body = await api.get<{ items: any[], total: number }>('/api/transactions', {
      ...monthBounds.value,
      account_id: filters.account_id || undefined,
      category_id: filters.category_id || undefined,
      member_id: filters.member_id || undefined,
      uncategorized: filters.uncategorized || undefined,
      search: filters.search || undefined,
      limit: page.limit, offset: page.offset,
    })
    rows.value = body.items
    total.value = body.total
  } finally { loading.value = false }
}

onMounted(async () => {
  ;[accounts.value, categories.value, members.value] = await Promise.all([
    api.get('/api/accounts'), api.get('/api/categories'), api.get('/api/members'),
  ])
  await load()
})
watch(filters, () => { page.offset = 0; load() }, { deep: true })
watch(page, load, { deep: true })

function openNew() { editing.value = null; modalOpen.value = true }
function openTx(id: number) { editing.value = rows.value.find(r => r.id === id); modalOpen.value = true }

const accountName = (id: number) => accounts.value.find(a => a.id === id)?.name ?? ''
const categoryOf = (id: number | null) => categories.value.find(c => c.id === id) ?? null
const memberName = (id: number | null) => members.value.find(m => m.id === id)?.name ?? null
const isTransfer = (tx: any) => Boolean(tx.transfer_group_id)
</script>

<template>
  <div class="max-w-3xl mx-auto space-y-4">
    <div class="flex items-center gap-2">
      <h1 class="text-xl font-extrabold">Transaksi</h1>
      <input v-model="filters.month" type="month" class="ml-auto border-brutal shadow-brutal rounded-brutal bg-hi px-2 py-1 font-extrabold text-sm">
    </div>

    <div class="border-brutal shadow-brutal rounded-brutal bg-white p-3 space-y-2">
      <input v-model="filters.search" class="w-full border-brutal rounded-brutal px-3 py-2"
             placeholder="Cari keterangan…">
      <div class="flex flex-wrap gap-2">
        <select v-model="filters.account_id" class="border-brutal rounded-brutal px-2 py-1 font-extrabold text-sm bg-white">
          <option value="">Semua akun</option>
          <option v-for="a in accounts" :key="a.id" :value="a.id">{{ a.name }}</option>
        </select>
        <select v-model="filters.category_id" class="border-brutal rounded-brutal px-2 py-1 font-extrabold text-sm bg-white">
          <option value="">Semua kategori</option>
          <option v-for="c in categories" :key="c.id" :value="c.id">{{ c.emoji }} {{ c.name }}</option>
        </select>
        <select v-model="filters.member_id" class="border-brutal rounded-brutal px-2 py-1 font-extrabold text-sm bg-white">
          <option value="">Semua anggota</option>
          <option v-for="m in members" :key="m.id" :value="m.id">{{ m.name }}</option>
        </select>
        <label class="border-brutal rounded-brutal px-2 py-1 font-extrabold text-sm bg-white flex items-center gap-2">
          <input v-model="filters.uncategorized" type="checkbox"> Belum dikategorikan
        </label>
      </div>
    </div>

    <p class="font-extrabold text-sm text-muted">{{ total }} transaksi</p>

    <div class="space-y-2">
      <template v-for="tx in rows" :key="tx.id">
        <div v-if="isTransfer(tx)" class="border-brutal border-dashed rounded-brutal bg-white p-3 flex items-center gap-3">
          <span class="w-9 h-9 rounded-brutal border-brutal bg-info flex items-center justify-center text-lg">🔁</span>
          <span class="flex-1 min-w-0">
            <span class="block font-extrabold truncate">{{ tx.description || 'Transfer' }}</span>
            <span class="block text-xs font-bold text-muted">{{ accountName(tx.account_id) }} · transfer</span>
          </span>
          <MoneyText :amount="tx.amount" signed />
        </div>
        <TransactionRow v-else
          :tx="{ ...tx, category: categoryOf(tx.category_id),
                 account_name: accountName(tx.account_id),
                 paid_by_name: memberName(tx.paid_by) }"
          @select="openTx" />
      </template>
      <p v-if="!loading && !rows.length" class="font-bold text-muted">Tidak ada transaksi pada periode ini.</p>
    </div>

    <div v-if="total > page.limit" class="flex gap-2 justify-center">
      <button :disabled="page.offset === 0" class="border-brutal rounded-brutal bg-white font-extrabold px-4 py-2 disabled:opacity-40"
              @click="page.offset = Math.max(0, page.offset - page.limit)">← Sebelumnya</button>
      <button :disabled="page.offset + page.limit >= total" class="border-brutal rounded-brutal bg-white font-extrabold px-4 py-2 disabled:opacity-40"
              @click="page.offset += page.limit">Berikutnya →</button>
    </div>

    <button class="fixed bottom-20 md:bottom-6 right-4 md:right-6 border-brutal shadow-brutal rounded-brutal bg-hi font-extrabold px-5 py-3 active:press z-40"
            @click="openNew">＋ Catat</button>

    <TransactionModal :open="modalOpen" :tx="editing" :accounts="accounts"
                      :categories="categories" :members="members"
                      @close="modalOpen = false" @saved="modalOpen = false; load()" />
  </div>
</template>
```

- [ ] **Step 3: Verify**

Expected: adding an expense appears immediately and changes the dashboard total. Editing works. Deleting a transfer removes **both** legs from the list. Setting a transfer between two accounts leaves the net worth unchanged.

- [ ] **Step 4: Commit**

```bash
git add frontend/components/TransactionModal.vue frontend/pages/transactions/index.vue
git commit -m "feat: transactions screen with add/edit modal and transfer mode"
```

---

### Task 34: Accounts, categories, rules, and settings screens

**Files:**
- Create: `frontend/pages/accounts/index.vue`, `frontend/pages/accounts/[id].vue`
- Create: `frontend/pages/categories.vue`, `frontend/pages/rules.vue`, `frontend/pages/settings.vue`
- Create: `backend/app/modules/accounts/router.py` addition: `/api/settings` endpoints (see Step 1)
- Create: `backend/app/modules/imports/router.py` addition: rules CRUD (see Step 2)

- [ ] **Step 1: Add household settings endpoints**

Settings is core infrastructure, not a domain module, so its routes live in `backend/app/core/settings_router.py`:

```python
from fastapi import APIRouter, Request
from pydantic import BaseModel

from app.core.deps import current_member
from app.core.settings_store import household_name, set_setting

router = APIRouter(prefix="/api/settings", tags=["settings"])


class SettingsIn(BaseModel):
    household_name: str | None = None


@router.get("")
def get_settings(request: Request):
    current_member(request)
    with request.app.state.db.session() as s:
        return {"household_name": household_name(s, request.app.state.settings.household_name),
                "currency": "IDR"}


@router.patch("")
def patch_settings(payload: SettingsIn, request: Request):
    current_member(request)
    with request.app.state.db.session() as s:
        if payload.household_name:
            set_setting(s, "household_name", payload.household_name)
        return {"household_name": household_name(s, request.app.state.settings.household_name)}
```

Include it in `main.py`: `from app.core.settings_router import router as settings_router` and `app.include_router(settings_router)`.

- [ ] **Step 2: Add rules CRUD to the imports router**

Append to `backend/app/modules/imports/router.py`:

```python
from pydantic import BaseModel

from app.modules.imports import rules as rules_mod
from app.modules.imports.models import Rule


class RuleIn(BaseModel):
    priority: int = 100
    match_type: str
    match_value: str
    set_category_id: int | None = None
    set_paid_by: int | None = None
    is_active: bool = True


class RuleOut(RuleIn):
    id: int

    model_config = ConfigDict(from_attributes=True)


@router.get("/rules", response_model=list[RuleOut])
def list_rules(request: Request):
    current_member(request)
    with _db(request).session() as s:
        return [RuleOut.model_validate(r) for r in rules_mod.active_rules(s)]


@router.post("/rules", response_model=RuleOut)
def create_rule(payload: RuleIn, request: Request):
    current_member(request)
    with _db(request).session() as s:
        rule = Rule(**payload.model_dump())
        s.add(rule)
        s.flush()
        return RuleOut.model_validate(rule)


@router.patch("/rules/{rule_id}", response_model=RuleOut)
def update_rule(rule_id: int, payload: RuleIn, request: Request):
    current_member(request)
    with _db(request).session() as s:
        rule = s.get(Rule, rule_id)
        if rule is None:
            raise ValidationFailed("rule_not_found", "Aturan tidak ditemukan")
        for key, value in payload.model_dump().items():
            setattr(rule, key, value)
        s.flush()
        return RuleOut.model_validate(rule)


@router.delete("/rules/{rule_id}")
def delete_rule(rule_id: int, request: Request):
    current_member(request)
    with _db(request).session() as s:
        rule = s.get(Rule, rule_id)
        if rule is not None:
            s.delete(rule)
    return {"ok": True}


@router.post("/rules/reapply")
def reapply_rules(request: Request):
    current_member(request)
    with _db(request).session() as s:
        return {"updated": rules_mod.reapply_rules(s)}
```

Add `ConfigDict` to the pydantic import in that file.

- [ ] **Step 3: Write `pages/accounts/index.vue` and `pages/accounts/[id].vue`**

`accounts/index.vue` lists accounts with balances (`GET /api/accounts`), plus an add form (name, type select with the four Indonesian labels, opening balance) and an archive button. Show archived accounts behind a toggle.

`accounts/[id].vue` is the ledger: the account's balance header, then the same `TransactionRow` list filtered by `account_id`, with the add-modal defaulted to that account.

```vue
<!-- accounts/[id].vue -->
<script setup lang="ts">
const api = useApi()
const route = useRoute()
const accountId = Number(route.params.id)
const account = ref<any>(null)
const rows = ref<any[]>([])
const categories = ref<any[]>([])
const members = ref<any[]>([])
const accounts = ref<any[]>([])

onMounted(async () => {
  ;[accounts.value, categories.value, members.value] = await Promise.all([
    api.get('/api/accounts'), api.get('/api/categories'), api.get('/api/members'),
  ])
  account.value = accounts.value.find((a: any) => a.id === accountId)
  const body = await api.get<{ items: any[] }>('/api/transactions', { account_id: accountId, limit: 200 })
  rows.value = body.items
})

const categoryOf = (id: number | null) => categories.value.find(c => c.id === id) ?? null
const memberName = (id: number | null) => members.value.find(m => m.id === id)?.name ?? null
</script>

<template>
  <div v-if="account" class="max-w-3xl mx-auto space-y-4">
    <NuxtLink to="/accounts" class="font-extrabold text-sm underline">← Semua akun</NuxtLink>
    <section class="border-brutal shadow-brutal rounded-brutal bg-white p-5">
      <p class="text-xs font-extrabold tracking-wide text-muted">{{ account.name }}</p>
      <MoneyText :amount="account.balance" size="xl" />
    </section>
    <div class="space-y-2">
      <TransactionRow v-for="tx in rows" :key="tx.id"
        :tx="{ ...tx, category: categoryOf(tx.category_id), account_name: account.name,
               paid_by_name: memberName(tx.paid_by) }" />
    </div>
  </div>
</template>
```

- [ ] **Step 4: Write `pages/categories.vue`, `pages/rules.vue`, `pages/settings.vue`**

`categories.vue` — a grid of `CategoryChip`s grouped by kind, each editable inline (name, emoji, colour picker, parent select, archive). Emoji is a plain text input, not a picker library.

`rules.vue` — a list ordered by `priority`; each row shows `match_type` + `match_value` + what it sets, with priority up/down buttons that PATCH the two swapped rows. Includes an "Terapkan ke transaksi lama" button calling `POST /api/rules/reapply` and reporting `{updated}`.

`settings.vue` — household name (PATCH `/api/settings`), the currency shown as read-only `IDR`, an "add member" form (POST `/api/members`), and a "Keluar dari semua perangkat" button calling POST `/api/auth/logout-all`.

- [ ] **Step 5: Verify**

Expected: creating an account appears in the dashboard's account row. A new rule of `contains: "superindo"` → `Makanan`, then "Terapkan ke transaksi lama", reduces the count of uncategorised rows. Renaming the household changes the header.

- [ ] **Step 6: Commit**

```bash
git add frontend/pages/accounts frontend/pages/categories.vue frontend/pages/rules.vue frontend/pages/settings.vue backend/app/core/settings_router.py backend/app/modules/imports/router.py backend/app/main.py
git commit -m "feat: accounts, ledger, categories, rules, and settings screens"
```

---

### Task 35: Import wizard

**Files:**
- Create: `frontend/pages/import/index.vue`
- Create: `frontend/components/ImportWizard.vue`

**Interfaces:**
- Consumes: `POST /api/imports/analyze`, `POST /api/imports/commit`, `GET /api/imports`, `POST /api/imports/{id}/rollback`.

- [ ] **Step 1: Write `pages/import/index.vue`**

Four steps: **upload → map → review → done**, with a stepper, plus a history list of past imports each with a rollback button.

```vue
<script setup lang="ts">
const api = useApi()

const step = ref(1)
const file = ref<File | null>(null)
const accounts = ref<any[]>([])
const profiles = ref<any[]>([])
const accountId = ref<number | null>(null)
const profileId = ref<number | null>(null)
const saveProfileName = ref('')

const mapping = reactive({
  delimiter: '', encoding: '', has_header: undefined as boolean | undefined, skip_rows: 0,
  date_format: '', amount_mode: 'signed' as string,
  columns: { date: null, description: null, amount: null, debit: null, credit: null, type: null } as Record<string, number | null>,
})

const analysis = ref<any>(null)
const included = ref<number[]>([])
const history = ref<any[]>([])
const result = ref<any>(null)
const error = ref('')
const busy = ref(false)

onMounted(async () => {
  ;[accounts.value, profiles.value, history.value] = await Promise.all([
    api.get('/api/accounts'), api.get('/api/import-profiles'), api.get('/api/imports'),
  ])
  accountId.value = accounts.value[0]?.id ?? null
})

function onFile(e: Event) {
  file.value = (e.target as HTMLInputElement).files?.[0] ?? null
}

async function analyze() {
  if (!file.value || !accountId.value) return
  error.value = ''; busy.value = true
  try {
    const form = new FormData()
    form.append('file', file.value)
    form.append('account_id', String(accountId.value))
    if (profileId.value) form.append('profile_id', String(profileId.value))
    if (mapping.delimiter) form.append('delimiter', mapping.delimiter)
    if (mapping.encoding) form.append('encoding', mapping.encoding)
    if (mapping.has_header !== undefined) form.append('has_header', String(mapping.has_header))
    if (mapping.skip_rows) form.append('skip_rows', String(mapping.skip_rows))
    if (mapping.date_format) form.append('date_format', mapping.date_format)
    form.append('amount_mode', mapping.amount_mode)

    const column_map: Record<string, number> = {}
    for (const [key, value] of Object.entries(mapping.columns)) {
      if (value !== null && value !== '') column_map[key] = Number(value)
    }
    form.append('column_map', JSON.stringify(column_map))

    analysis.value = await api.upload('/api/imports/analyze', form)
    included.value = analysis.value.rows
      .filter((r: any) => r.status === 'new')
      .map((r: any) => r.index)
    step.value = 3
  } catch (e: any) {
    error.value = e.message
  } finally { busy.value = false }
}

async function commit() {
  error.value = ''; busy.value = true
  try {
    const rows = analysis.value.rows
      .filter((r: any) => included.value.includes(r.index))
      .map((r: any) => ({ date: r.date, amount: r.amount, description: r.description,
                          category_id: r.category_id, paid_by: r.paid_by }))
    result.value = await api.post('/api/imports/commit', {
      account_id: accountId.value,
      filename: file.value?.name ?? 'upload.csv',
      profile_id: profileId.value,
      save_profile: saveProfileName.value
        ? { name: saveProfileName.value, delimiter: analysis.value.delimiter,
            encoding: analysis.value.encoding, has_header: analysis.value.has_header,
            skip_rows: mapping.skip_rows, date_format: mapping.date_format || null,
            amount_mode: mapping.amount_mode,
            column_map: Object.fromEntries(Object.entries(mapping.columns)
              .filter(([, v]) => v !== null && v !== '')) }
        : null,
      rows,
    })
    step.value = 4
  } catch (e: any) {
    error.value = e.message
  } finally { busy.value = false }
}

function reset() {
  step.value = 1; file.value = null; analysis.value = null; result.value = null
  included.value = []; saveProfileName.value = ''
  api.get('/api/imports').then(h => (history.value = h))
}

async function rollback(id: number) {
  if (!confirm('Batalkan impor ini beserta semua transaksinya?')) return
  await api.post(`/api/imports/${id}/rollback`)
  history.value = await api.get('/api/imports')
}
</script>
```

The template renders the stepper, then per step:

- **1 Upload** — file input, account select, optional profile select.
- **2 Map** — detected values shown as editable fields (`delimiter`, `encoding`, `has_header`, `skip_rows`, `date_format`, `amount_mode`) and one column-index input per mapped field. A prominent warning when `amount_mode` needs column confirmation. `date_format` left empty shows: *"Kosongkan hanya jika tanggal tidak ambigu. Tanggal seperti 03/04/2026 akan ditolak, bukan ditebak."*
- **3 Review** — counts summary (*baru / duplikat / kemungkinan duplikat / error*), then a checkbox row per row showing status badge, description, `MoneyText`, and the matched rule. `possible_duplicate` rows are **unchecked by default** (which the `included` initialisation already does, since it only includes `new`).
- **4 Done** — the report: imported / skipped duplicates / possible duplicates / errors, with a "Lihat transaksi" link.

History list at the bottom: filename, date, counts, and a "Batalkan" button per non-rolled-back batch.

- [ ] **Step 2: Verify end to end**

Export a real CSV from one of your banks. Expected:
- It detects delimiter, encoding and header correctly.
- Left empty, `03/04/2026` produces a **red row error**, not a guess. Setting `%d/%m/%Y` fixes it.
- Review shows the true new/duplicate split.
- Committing updates the account balance by exactly the sum of imported rows.
- **Re-uploading the same file shows everything as duplicate and imports zero.**
- Rollback restores the previous balance exactly.

- [ ] **Step 3: Commit**

```bash
git add frontend/pages/import/index.vue frontend/components/ImportWizard.vue
git commit -m "feat: four-step csv import wizard with review and rollback"
```

---

### Task 36: Reports screen and accessibility pass

**Files:**
- Create: `frontend/pages/reports.vue`
- Modify: any component failing the accessibility check

- [ ] **Step 1: Write `pages/reports.vue`**

Three panels driven by the reports API: net worth per account, income vs expense for the selected month, and spending by category as bars; plus a 12-month expense bar chart from twelve `cash-flow` calls (or one `by-category` per month — twelve requests against a local SQLite is fine, and simpler than adding a new endpoint).

```vue
<script setup lang="ts">
const api = useApi()
const month = ref(new Date().toISOString().slice(0, 7))

interface NetWorth { total: number, accounts: { id: number, name: string, type: string, balance: number }[] }
interface CashFlow { income: number, expense: number, net: number }
interface CategorySpend { category_id: number | null, name: string, emoji: string, color: string, total: number }

const netWorth = ref<NetWorth | null>(null)
const cashFlow = ref<CashFlow | null>(null)
const byCategory = ref<CategorySpend[]>([])
const byCategoryIncome = ref<CategorySpend[]>([])
const trend = ref<{ month: string, expense: number }[]>([])

function bounds(m: string) {
  const [y, mm] = m.split('-').map(Number)
  const last = new Date(y, mm, 0).getDate()
  return { date_from: `${m}-01`, date_to: `${m}-${String(last).padStart(2, '0')}` }
}

async function load() {
  const b = bounds(month.value)
  ;[netWorth.value, cashFlow.value, byCategory.value, byCategoryIncome.value] = await Promise.all([
    api.get<NetWorth>('/api/reports/net-worth'),
    api.get<CashFlow>('/api/reports/cash-flow', b),
    api.get<CategorySpend[]>('/api/reports/by-category', { ...b, kind: 'expense' }),
    api.get<CategorySpend[]>('/api/reports/by-category', { ...b, kind: 'income' }),
  ])
  const months = Array.from({ length: 12 }, (_, i) => {
    const d = new Date(Number(month.value.slice(0, 4)), Number(month.value.slice(5, 7)) - 1 - (11 - i), 1)
    return d.toISOString().slice(0, 7)
  })
  trend.value = await Promise.all(months.map(async m => ({
    month: m,
    expense: (await api.get<CashFlow>('/api/reports/cash-flow', bounds(m))).expense,
  })))
}

onMounted(load)
watch(month, load)

const trendMax = computed(() => Math.max(1, ...trend.value.map(t => Math.abs(t.expense))))
const monthLabel = (m: string) => new Date(Number(m.slice(0, 4)), Number(m.slice(5, 7)) - 1, 1)
  .toLocaleDateString('id-ID', { month: 'short' })
</script>
```

The template shows the net-worth card with per-account breakdown, the in/out cards, a 12-bar trend (bar height by `Math.abs(expense)/trendMax`), and category bars via `ProgressBar` normalised to the largest category. Income categories get their own list so a salary does not look like a food expense.

- [ ] **Step 2: Run the accessibility check**

For every page, confirm by inspection of the rendered CSS:

```bash
cd frontend && grep -rn "text-white" components pages | grep -v avatar
```

Expected: **no matches.** White text appears only on `MemberAvatar` and the login avatar tiles, where the background is a per-member colour chosen by the user. Everything else uses `text-ink` on colour.

Then check contrast of each background used with text: `bg-in`, `bg-hi`, `bg-info`, `bg-cat` must all pair with `text-ink` (or no explicit colour, inheriting ink). `bg-out` must pair with `text-ink`.

- [ ] **Step 3: Verify the whole app at 375 px width**

Use the browser device toolbar at 375 × 812. Expected on every page: the bottom nav is reachable and does not cover the "＋ Catat" button; no horizontal scrolling; the net-worth number fits without truncation; the add-transaction modal scrolls internally on a short screen.

- [ ] **Step 4: Commit**

```bash
git add frontend/pages/reports.vue frontend/components frontend/pages
git commit -m "feat: reports screen and accessibility pass on colour and contrast"
```

---

### Task 37: Cold-boot end-to-end test

**Files:**
- Create: `backend/tests/test_end_to_end.py`

One test that boots the app against an empty database and walks the entire v1 flow. If a later refactor breaks the wiring between modules, this is the test that says so.

- [ ] **Step 1: Write the test**

```python
# backend/tests/test_end_to_end.py
"""Cold boot: fresh database, fresh app, the whole v1 flow in one pass."""
import json

from fastapi.testclient import TestClient

from app.core.config import Settings
from app.core.db import Database
from app.main import create_app

CSV = ("Tanggal,Keterangan,Jumlah\n"
       "01/02/2026,Superindo Jakarta,-186000\n"
       "02/02/2026,Gaji Februari,4200000\n").encode()
COLUMN_MAP = json.dumps({"date": 0, "description": 1, "amount": 2})


def test_cold_boot_walks_the_whole_v1_flow(tmp_path):
    db = Database(tmp_path / "cold.db")
    app = create_app(settings=Settings(db_path=db.path, household_name="Keluarga Kita"))

    with TestClient(app) as client:
        assert client.get("/api/health").json() == {"status": "ok"}
        assert client.get("/api/auth/state").json()["needs_setup"] is True

        # first-run setup
        assert client.post("/api/setup", json={
            "name": "Budi", "username": "budi",
            "password": "supersecret123"}).status_code == 200

        bca = client.post("/api/accounts", json={
            "name": "BCA", "type": "bank", "opening_balance": 1_000_000}).json()
        cash = client.post("/api/accounts", json={
            "name": "Tunai", "type": "cash", "opening_balance": 0}).json()
        food = client.post("/api/categories", json={
            "name": "Makanan", "kind": "expense", "emoji": "🍚"}).json()

        # a rule that will auto-categorise on import
        client.post("/api/rules", json={
            "priority": 10, "match_type": "contains", "match_value": "superindo",
            "set_category_id": food["id"]})

        # manual entry and a transfer
        client.post("/api/transactions", json={
            "account_id": bca["id"], "date": "2026-02-01",
            "amount": -50_000, "description": "manual"})
        client.post("/api/transfers", json={
            "from_account_id": bca["id"], "to_account_id": cash["id"],
            "date": "2026-02-02", "amount": 500_000, "description": "Tarik tunai"})

        # import, with the rule applying
        analyzed = client.post("/api/imports/analyze",
                               data={"account_id": str(bca["id"]), "amount_mode": "signed",
                                     "date_format": "%d/%m/%Y", "column_map": COLUMN_MAP},
                               files={"file": ("bca.csv", CSV, "text/csv")}).json()
        assert analyzed["counts"]["new"] == 2
        assert analyzed["rows"][0]["category_id"] == food["id"]

        rows = [{"date": r["date"], "amount": r["amount"], "description": r["description"],
                 "category_id": r["category_id"]} for r in analyzed["rows"]]
        batch = client.post("/api/imports/commit", json={
            "account_id": bca["id"], "filename": "bca.csv", "rows": rows}).json()
        assert batch["imported_count"] == 2

        # re-importing the same file changes nothing
        again = client.post("/api/imports/analyze",
                            data={"account_id": str(bca["id"]), "amount_mode": "signed",
                                  "date_format": "%d/%m/%Y", "column_map": COLUMN_MAP},
                            files={"file": ("bca.csv", CSV, "text/csv")}).json()
        assert again["counts"].get("duplicate") == 2

        # the dashboard reflects the whole household, transfers netted out
        dash = client.get("/api/reports/dashboard?month=2026-02").json()
        assert dash["net_worth"]["total"] == 1_000_000 - 50_000 - 186_000 + 4_200_000
        assert dash["cash_flow"]["income"] == 4_200_000
        assert dash["cash_flow"]["expense"] == 236_000
        assert dash["top_categories"][0]["name"] == "Makanan"

        # rollback removes exactly the imported rows
        assert client.post(f"/api/imports/{batch['id']}/rollback").status_code == 200
        assert client.get("/api/reports/net-worth").json()["total"] == 1_000_000 - 50_000
```

- [ ] **Step 2: Run the whole backend suite**

Run: `cd backend && .venv/bin/pytest -v`
Expected: all passed, including this test

- [ ] **Step 3: Commit**

```bash
git add backend/tests/test_end_to_end.py
git commit -m "test: cold-boot end-to-end test of the full v1 flow"
```

---

## Self-Review

Run against the spec with fresh eyes. Four gaps found, all fixed.

### 1. Spec coverage

| Spec section | Owning tasks | Status |
|---|---|---|
| §1 Intent, success criteria | 32 (dashboard), 33 (fast entry), 35 (clean import), 6 (one-file backup) | covered |
| §2 In-v1 scope | 8–13 auth · 15 accounts · 16 categories · 17–20 transactions · 21–27 import · 25+34 rules · 28–29 reports | covered |
| §2 Out-of-v1 | nothing built for budgets/goals/investments/FX/event log | correct |
| §3 Architecture, module contract, Docker, money | 1–2 (layout, db), 5 (money), 6 (docker), 27+30 (single-origin serving) | covered |
| §4 Data model, all 9 tables, indexes, 9 decisions | 8, 14, 15, 16, 17, 21 | covered |
| §5 Auth: argon2id, roles, first-run setup, no signup, rate limit, CSRF/origin, expiry, log-out-everywhere | 9, 10, 11, 12, 13, 34 | covered |
| §5 **adult resets another member's password** | — | **gap → added `POST /api/members/{id}/password` to Task 13 plus a test** |
| §6 Import: sniff, 4 amount modes, ambiguity gate, dedupe, rules, review, commit, rollback, re-apply, limits | 21, 22, 23, 24, 25, 26, 27, 35 | covered |
| §6 Row errors are data, never a 500 | 26 (`_build_drafts` collects `error` results) | covered |
| §7 UI system, tokens, primitives, 5 UX rules, no component kit | 7, 30, 32, 33, 36 | covered |
| §7 Accessibility: ink on colour, never white on colour | 36 Step 2 (grep gate) | covered |
| §8 Testing: pytest on real SQLite, hypothesis, API integration, E2E cold boot | 5, 22, 24, 26, 27, 37 | covered |
| §8 **E2E cold-boot test** | — | **gap → added Task 37** |
| §8 Error envelope, SQLite PRAGMAs, Alembic on boot, `/api/health` | 2, 3, 4 | covered |
| §8 nightly `app.db` copy | — | **deliberate gap:** host cron, not code. Belongs in a README, not the plan. |
| §9 Risks | mitigation for risk 5 (reference numbers) is Task 24's `possible_duplicate` | covered |

### 2. Placeholder scan

No `TBD`, `TODO`, "implement later", or "add appropriate error handling" remain. Every code step carries real code. Four working-through-it notes were found and removed during writing (Tasks 10, 12, 17, 25, 28, 34) — the plan now contains only final content.

Two steps are intentionally verification-only rather than code, and are labelled as such: Task 30 Step 6, Task 33 Step 3, Task 35 Step 2, Task 36 Steps 2–3. These are UI checks against a running server, which is what the spec prescribes for v1 (no frontend unit tests).

### 3. Type and name consistency

Verified across tasks:

- `parse_rupiah` returns `int`, raises `MoneyParseError` — used in Tasks 5, 22, 18, 19.
- `RowResult` fields (`index, date, amount, description, status, error, category_id, paid_by, matched_rule_id, final_hash, include`) match their use in Tasks 24, 26, 27.
- `classify_rows(s, *, account_id, drafts)` — the `drafts` shape (`{"date", "amount", "description"}`) is identical in Tasks 24, 26.
- `sniff_csv(raw, *, max_rows, forced)` — `forced` keys (`delimiter`, `encoding`, `has_header`, `skip_rows`) are produced by Task 27's router and consumed by Task 21.
- `analyse_import(..., forced=...)` — matches Task 27's call site.
- `create_transfer` returns `(out_leg, in_leg)` and is unpacked the same way in Tasks 19, 20.
- `COOKIE_NAME = "ff_session"` defined in Task 9, read in Tasks 12, 13; the frontend relies on the cookie implicitly via `credentials: 'include'` (Task 30).
- `ApiError{code, message, details}` matches the server envelope from Task 4.
- Token names `--color-paper|ink|in|out|hi|info|cat|muted` match the Tailwind classes used in every UI task.

One ordering note, not a bug: `AppShell` (Task 30) calls `GET /api/settings`, which Task 34 creates. The call is wrapped in `try/catch` and the household name falls back to a constant, so the app is correct before Task 34 lands.

### 4. Review Focus coverage

Each item in the plan header's Review Focus has a test in its owning task:

| Review Focus item | Owning task and test |
|---|---|
| 1. Non-UTF-8 / BOM encodings | Task 21 — `test_utf8_bom_is_stripped_not_left_in_the_first_header`, `test_cp1252_high_bytes_decode_without_mojibake`, `test_forced_encoding_is_honoured` |
| 2. Hostile amount strings | Task 22 — `test_hostile_values_raise_a_row_error_not_a_crash`, `test_indonesian_amount_formats_survive_the_full_path` |
| 3. Transfers must not change net worth | Task 19 — `test_transfer_does_not_change_net_worth` and Task 28 — `test_transfer_does_not_change_net_worth`, `test_transfer_is_excluded_from_cash_flow` |
| 4. Degenerate files | Task 21 — `test_empty_file_is_rejected`, `test_binary_file_is_rejected`, `test_row_cap_is_enforced` |
| 5. Two distinct same-key rows, and idempotent re-import | Task 24 — `test_two_distinct_same_day_rows_both_import`; Task 26 — `test_reimporting_the_same_rows_is_a_no_op`; Task 37 — the cold-boot re-import assertion |

---

## Execution Handoff

**Plan complete.** 37 tasks across 7 phases. Each phase ends in something runnable: a login (2), a working ledger (4), a clean import (5), a pleasant app (7).

Method recommendation is in the handoff message.

