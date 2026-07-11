# Phase 1 Scaffolding Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up the Career Asset Manager monorepo — a FastAPI backend and a Next.js frontend — with real, working JWT authentication (register/login/refresh/`/users/me`) end to end, plus the tooling and test scaffolding every later Phase 1 feature (Resume Manager, Version Control, Tags) will build on.

**Architecture:** Backend is a layered FastAPI app (`api` → `services` → `repositories` → `models`, with `schemas`/`core` cross-cutting) backed by async SQLAlchemy + Postgres + Alembic. Frontend is a Next.js App Router app where the browser only ever talks to the frontend's own Route Handlers; those Route Handlers proxy to the FastAPI backend server-to-server and manage the refresh token as an httpOnly cookie, keeping it out of reach of any XSS in the React tree.

**Tech Stack:** FastAPI, SQLAlchemy 2.0 (async, `asyncpg`), Alembic, Pydantic v2 / pydantic-settings, PyJWT, `bcrypt`, pytest + pytest-asyncio + httpx, uv (backend); Next.js (App Router) + TypeScript + Tailwind CSS, Vitest + React Testing Library, npm (frontend).

## Global Constraints

- Backend: FastAPI, SQLAlchemy, Alembic, Pydantic, JWT, PostgreSQL (per `what_to_do.txt` → Backend).
- Frontend: React, Next.js, TypeScript, Tailwind CSS, dark & light mode (per `what_to_do.txt` → Frontend).
- Clean architecture: separate API / business logic / services / repositories / models / schemas / utils / configuration / tests (per `what_to_do.txt` → Folder Structure).
- Production-quality, modular, strongly typed, no duplicated code, no unnecessary abstractions (per `what_to_do.txt` → Coding Rules).
- Schema changes only via Alembic migrations, never manual DDL (per `CLAUDE.md`).
- Docker Compose / CI is Phase 5, not this plan — local Postgres only (per `docs/superpowers/specs/2026-07-12-phase1-scaffolding-design.md`).
- Forgot-password, email verification, OAuth, and RBAC beyond a role-ready schema are out of scope for this plan (per the same design doc).
- Backend package manager: `uv`. Frontend package manager: `npm`.

---

## Prerequisites (do once, before Task 1)

- Postgres is installed and running locally.
- Create the dev and test databases and a role for the app:

```bash
sudo -u postgres psql -c "CREATE ROLE cam_user WITH LOGIN PASSWORD 'cam_password';"
sudo -u postgres psql -c "CREATE DATABASE cam_dev OWNER cam_user;"
sudo -u postgres psql -c "CREATE DATABASE cam_test OWNER cam_user;"
```

(If your local Postgres uses a different auth setup, adjust — the rest of this plan only assumes a working `DATABASE_URL` for `cam_dev` and `cam_test`.)

- `uv` is installed (`curl -LsSf https://astral.sh/uv/install.sh | sh` or your package manager).
- Node.js (current LTS) and `npm` are installed.

---

### Task 1: Backend project scaffolding

**Files:**
- Create: `backend/pyproject.toml`
- Create: `backend/.gitignore` — actually goes at repo root, see below
- Create: `.gitignore` (repo root)
- Create: `backend/.env.example`
- Create: `backend/app/__init__.py`, `backend/app/api/__init__.py`, `backend/app/api/routes/__init__.py`, `backend/app/core/__init__.py`, `backend/app/models/__init__.py`, `backend/app/repositories/__init__.py`, `backend/app/schemas/__init__.py`, `backend/app/services/__init__.py`, `backend/tests/__init__.py`

**Interfaces:**
- Produces: a `uv`-managed Python 3.12 project at `backend/` with FastAPI, SQLAlchemy(async), asyncpg, Alembic, pydantic-settings, email-validator, bcrypt, PyJWT as runtime deps, and pytest/pytest-asyncio/httpx/ruff/mypy as dev deps.

- [ ] **Step 1: Create the backend package tree**

```bash
mkdir -p backend/app/api/routes backend/app/core backend/app/models backend/app/repositories backend/app/schemas backend/app/services backend/tests
touch backend/app/__init__.py backend/app/api/__init__.py backend/app/api/routes/__init__.py \
      backend/app/core/__init__.py backend/app/models/__init__.py backend/app/repositories/__init__.py \
      backend/app/schemas/__init__.py backend/app/services/__init__.py backend/tests/__init__.py
```

- [ ] **Step 2: Write `backend/pyproject.toml`**

```toml
[project]
name = "cam-backend"
version = "0.1.0"
description = "Career Asset Manager backend"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.32.0",
    "sqlalchemy[asyncio]>=2.0.35",
    "asyncpg>=0.30.0",
    "alembic>=1.13.3",
    "pydantic-settings>=2.5.2",
    "email-validator>=2.2.0",
    "bcrypt>=4.2.0",
    "pyjwt>=2.9.0",
]

[dependency-groups]
dev = [
    "pytest>=8.3.3",
    "pytest-asyncio>=0.24.0",
    "httpx>=0.27.2",
    "ruff>=0.7.0",
    "mypy>=1.11.2",
]

[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B"]

[tool.mypy]
python_version = "3.12"
disallow_untyped_defs = true
warn_unused_ignores = true
ignore_missing_imports = true

[tool.pytest.ini_options]
asyncio_mode = "auto"
```

- [ ] **Step 3: Install dependencies and create the lockfile**

Run: `cd backend && uv sync`
Expected: creates `backend/.venv/` and `backend/uv.lock`, prints a resolved dependency list ending in something like `Installed 4X packages`.

- [ ] **Step 4: Write `backend/.env.example`**

```
DATABASE_URL=postgresql+asyncpg://cam_user:cam_password@localhost:5432/cam_dev
SECRET_KEY=change-me-to-a-random-secret
ACCESS_TOKEN_EXPIRE_MINUTES=15
REFRESH_TOKEN_EXPIRE_DAYS=7
CORS_ORIGINS=http://localhost:3000
```

- [ ] **Step 5: Copy it to a real local `.env`**

Run: `cp backend/.env.example backend/.env`
Then edit `backend/.env` and replace `SECRET_KEY` with a real random value, e.g. output of `python3 -c "import secrets; print(secrets.token_urlsafe(32))"`.

- [ ] **Step 6: Write the repo-root `.gitignore`**

```
# Python
backend/.venv/
**/__pycache__/
backend/.env

# Node
frontend/node_modules/
frontend/.next/
frontend/.env.local

# Editors/OS
.DS_Store
```

- [ ] **Step 7: Commit**

```bash
git add backend/pyproject.toml backend/uv.lock backend/.env.example .gitignore \
        backend/app backend/tests
git commit -m "chore: scaffold backend project with uv"
```

---

### Task 2: Settings & environment config

**Files:**
- Create: `backend/app/core/config.py`
- Test: `backend/tests/test_config.py`

**Interfaces:**
- Produces: `app.core.config.settings` — a module-level `Settings` instance with `DATABASE_URL: str`, `SECRET_KEY: str`, `ACCESS_TOKEN_EXPIRE_MINUTES: int`, `REFRESH_TOKEN_EXPIRE_DAYS: int`, `CORS_ORIGINS: str`.
- Consumes: `backend/.env` (created in Task 1) must exist with real `DATABASE_URL`/`SECRET_KEY` for any test run to succeed, since `settings` is instantiated at import time.

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/test_config.py
from app.core.config import Settings


def test_settings_load_required_fields_from_env(monkeypatch):
    monkeypatch.setenv("DATABASE_URL", "postgresql+asyncpg://u:p@localhost:5432/db")
    monkeypatch.setenv("SECRET_KEY", "test-secret")
    settings = Settings(_env_file=None)
    assert settings.DATABASE_URL == "postgresql+asyncpg://u:p@localhost:5432/db"
    assert settings.SECRET_KEY == "test-secret"
    assert settings.ACCESS_TOKEN_EXPIRE_MINUTES == 15
    assert settings.REFRESH_TOKEN_EXPIRE_DAYS == 7
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && uv run pytest tests/test_config.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'app.core.config'`

- [ ] **Step 3: Write the implementation**

```python
# backend/app/core/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_file_encoding="utf-8", extra="ignore")

    DATABASE_URL: str
    SECRET_KEY: str
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 15
    REFRESH_TOKEN_EXPIRE_DAYS: int = 7
    CORS_ORIGINS: str = "http://localhost:3000"


settings = Settings()
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd backend && uv run pytest tests/test_config.py -v`
Expected: `1 passed`

- [ ] **Step 5: Commit**

```bash
git add backend/app/core/config.py backend/tests/test_config.py
git commit -m "feat: add settings config loaded from .env"
```

---

### Task 3: Async DB engine/session + Base model

**Files:**
- Create: `backend/app/models/base.py`
- Create: `backend/app/core/db.py`

**Interfaces:**
- Consumes: `app.core.config.settings.DATABASE_URL` (Task 2).
- Produces: `app.models.base.Base` (SQLAlchemy `DeclarativeBase` for all models to inherit); `app.core.db.get_db` — an async generator FastAPI dependency yielding an `AsyncSession`.

- [ ] **Step 1: Write `backend/app/models/base.py`**

```python
from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    pass
```

- [ ] **Step 2: Write `backend/app/core/db.py`**

```python
from collections.abc import AsyncGenerator

from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

from app.core.config import settings

engine = create_async_engine(settings.DATABASE_URL, echo=False)
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False, class_=AsyncSession)


async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        yield session
```

There's no isolated unit test for this file — it's exercised end-to-end by Task 4's smoke test and every DB-backed test after it. Confirm it at least imports cleanly:

- [ ] **Step 3: Verify it imports without error**

Run: `cd backend && uv run python -c "from app.core.db import get_db, engine; print('ok')"`
Expected: prints `ok`

- [ ] **Step 4: Commit**

```bash
git add backend/app/models/base.py backend/app/core/db.py
git commit -m "feat: add async SQLAlchemy engine/session and declarative base"
```

---

### Task 4: Minimal FastAPI app + health check + test infrastructure

**Files:**
- Create: `backend/app/main.py`
- Create: `backend/tests/conftest.py`
- Test: `backend/tests/test_health.py`

**Interfaces:**
- Consumes: `app.core.db.get_db` (Task 3), `app.models.base.Base` (Task 3).
- Produces: `app.main.app` (the FastAPI instance, importable by Uvicorn and by tests); `backend/tests/conftest.py` fixtures `client` (an `httpx.AsyncClient` wired to the app with a real Postgres test DB) and `db_session` (a raw `AsyncSession` against `cam_test`), reused by every later backend test.

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/test_health.py
import pytest

pytestmark = pytest.mark.asyncio


async def test_health_check_returns_ok(client):
    response = await client.get("/health")
    assert response.status_code == 200
    assert response.json() == {"status": "ok"}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && uv run pytest tests/test_health.py -v`
Expected: FAIL — `client` fixture not found / `ModuleNotFoundError: No module named 'app.main'`

- [ ] **Step 3: Write `backend/app/main.py`**

```python
from fastapi import FastAPI


def create_app() -> FastAPI:
    app = FastAPI(title="Career Asset Manager API")

    @app.get("/health")
    async def health() -> dict[str, str]:
        return {"status": "ok"}

    return app


app = create_app()
```

- [ ] **Step 4: Write `backend/tests/conftest.py`**

The test suite points at `cam_test` (created in Prerequisites) and creates/drops tables around every test for isolation, bypassing Alembic for speed.

```python
import pytest_asyncio
from httpx import ASGITransport, AsyncClient
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

from app.core.config import settings
from app.core.db import get_db
from app.main import app
from app.models.base import Base

TEST_DATABASE_URL = settings.DATABASE_URL.rsplit("/", 1)[0] + "/cam_test"

test_engine = create_async_engine(TEST_DATABASE_URL, echo=False)
TestSessionLocal = async_sessionmaker(test_engine, expire_on_commit=False, class_=AsyncSession)


@pytest_asyncio.fixture(autouse=True)
async def setup_db():
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)


@pytest_asyncio.fixture
async def db_session():
    async with TestSessionLocal() as session:
        yield session


@pytest_asyncio.fixture
async def client(db_session):
    async def override_get_db():
        yield db_session

    app.dependency_overrides[get_db] = override_get_db
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as ac:
        yield ac
    app.dependency_overrides.clear()
```

- [ ] **Step 5: Run test to verify it passes**

Run: `cd backend && uv run pytest tests/test_health.py -v`
Expected: `1 passed`

- [ ] **Step 6: Commit**

```bash
git add backend/app/main.py backend/tests/conftest.py backend/tests/test_health.py
git commit -m "feat: add FastAPI app with health check and test infrastructure"
```

---

### Task 5: User model + Alembic migration

**Files:**
- Create: `backend/app/models/user.py`
- Modify: `backend/app/models/__init__.py`
- Modify: `backend/tests/conftest.py`
- Create: `backend/alembic.ini`, `backend/alembic/env.py`, `backend/alembic/script.py.mako`, `backend/alembic/versions/` (via `alembic init`)
- Create: `backend/alembic/versions/<rev>_create_users_table.py` (via autogenerate)

**Interfaces:**
- Consumes: `app.models.base.Base` (Task 3).
- Produces: `app.models.user.User` with fields `id: uuid.UUID` (PK), `email: str` (unique), `hashed_password: str`, `created_at: datetime`.

- [ ] **Step 1: Write `backend/app/models/user.py`**

```python
import uuid
from datetime import datetime

from sqlalchemy import DateTime, String, func
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column

from app.models.base import Base


class User(Base):
    __tablename__ = "users"

    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True, nullable=False)
    hashed_password: Mapped[str] = mapped_column(String(255), nullable=False)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
```

- [ ] **Step 2: Register it on `Base.metadata`**

```python
# backend/app/models/__init__.py
from app.models.user import User

__all__ = ["User"]
```

- [ ] **Step 3: Make sure the test suite picks up the model**

Add this import near the top of `backend/tests/conftest.py`, right after the existing imports:

```python
import app.models  # noqa: F401  ensures all models are registered on Base.metadata before create_all
```

- [ ] **Step 4: Run the existing tests to confirm nothing broke**

Run: `cd backend && uv run pytest -v`
Expected: `2 passed` (health check + config test)

- [ ] **Step 5: Initialize Alembic**

Run: `cd backend && uv run alembic init alembic`
Expected: creates `backend/alembic.ini`, `backend/alembic/env.py`, `backend/alembic/script.py.mako`, `backend/alembic/versions/`

- [ ] **Step 6: Replace the generated `backend/alembic/env.py`**

```python
import asyncio
from logging.config import fileConfig

from alembic import context
from sqlalchemy import pool
from sqlalchemy.ext.asyncio import async_engine_from_config

import app.models  # noqa: F401  registers all models on Base.metadata
from app.core.config import settings
from app.models.base import Base

config = context.config
config.set_main_option("sqlalchemy.url", settings.DATABASE_URL)

if config.config_file_name is not None:
    fileConfig(config.config_file_name)

target_metadata = Base.metadata


def run_migrations_offline() -> None:
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )
    with context.begin_transaction():
        context.run_migrations()


def do_run_migrations(connection) -> None:
    context.configure(connection=connection, target_metadata=target_metadata)
    with context.begin_transaction():
        context.run_migrations()


async def run_migrations_online() -> None:
    connectable = async_engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)

    await connectable.dispose()


if context.is_offline_mode():
    run_migrations_offline()
else:
    asyncio.run(run_migrations_online())
```

- [ ] **Step 7: Generate the migration**

Run: `cd backend && uv run alembic revision --autogenerate -m "create users table"`
Expected: creates a file at `backend/alembic/versions/<hash>_create_users_table.py`. Open it and confirm `upgrade()` creates a `users` table with columns `id` (UUID, PK), `email` (String(255), unique index), `hashed_password` (String(255), not null), `created_at` (DateTime with timezone). If autogenerate produced something different (e.g. missed the unique index), edit the migration by hand to match.

- [ ] **Step 8: Apply the migration to the dev database**

Run: `cd backend && uv run alembic upgrade head`
Expected: prints `Running upgrade -> <hash>, create users table`

- [ ] **Step 9: Verify the table exists**

Run: `psql postgresql://cam_user:cam_password@localhost:5432/cam_dev -c "\d users"`
Expected: shows the `users` table with the four columns above.

- [ ] **Step 10: Commit**

```bash
git add backend/app/models/user.py backend/app/models/__init__.py backend/tests/conftest.py \
        backend/alembic.ini backend/alembic/env.py backend/alembic/script.py.mako backend/alembic/versions
git commit -m "feat: add User model and initial Alembic migration"
```

---

### Task 6: Security utilities (password hashing + JWT)

**Files:**
- Create: `backend/app/core/security.py`
- Test: `backend/tests/test_security.py`

**Interfaces:**
- Consumes: `app.core.config.settings.SECRET_KEY` (Task 2).
- Produces: `hash_password(password: str) -> str`, `verify_password(password: str, hashed: str) -> bool`, `create_access_token(subject: str, expires_delta: timedelta) -> str`, `create_refresh_token(subject: str, expires_delta: timedelta) -> str`, `decode_token(token: str) -> dict`.

- [ ] **Step 1: Write the failing tests**

```python
# backend/tests/test_security.py
from datetime import timedelta

import jwt
import pytest

from app.core.config import settings
from app.core.security import (
    create_access_token,
    create_refresh_token,
    decode_token,
    hash_password,
    verify_password,
)


def test_hash_password_is_not_the_plain_password():
    hashed = hash_password("correct-horse-battery-staple")
    assert hashed != "correct-horse-battery-staple"


def test_verify_password_accepts_correct_password():
    hashed = hash_password("correct-horse-battery-staple")
    assert verify_password("correct-horse-battery-staple", hashed) is True


def test_verify_password_rejects_wrong_password():
    hashed = hash_password("correct-horse-battery-staple")
    assert verify_password("wrong-password", hashed) is False


def test_access_token_round_trips_subject_and_type():
    token = create_access_token("user-123", timedelta(minutes=15))
    payload = decode_token(token)
    assert payload["sub"] == "user-123"
    assert payload["type"] == "access"


def test_refresh_token_round_trips_subject_and_type():
    token = create_refresh_token("user-123", timedelta(days=7))
    payload = decode_token(token)
    assert payload["sub"] == "user-123"
    assert payload["type"] == "refresh"


def test_decode_token_rejects_bad_signature():
    forged = jwt.encode({"sub": "user-123", "type": "access"}, "wrong-secret", algorithm="HS256")
    with pytest.raises(jwt.InvalidTokenError):
        decode_token(forged)
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && uv run pytest tests/test_security.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'app.core.security'`

- [ ] **Step 3: Write `backend/app/core/security.py`**

```python
from datetime import datetime, timedelta, timezone

import bcrypt
import jwt

from app.core.config import settings

ALGORITHM = "HS256"


def hash_password(password: str) -> str:
    return bcrypt.hashpw(password.encode("utf-8"), bcrypt.gensalt()).decode("utf-8")


def verify_password(password: str, hashed: str) -> bool:
    return bcrypt.checkpw(password.encode("utf-8"), hashed.encode("utf-8"))


def _create_token(subject: str, token_type: str, expires_delta: timedelta) -> str:
    expire = datetime.now(timezone.utc) + expires_delta
    payload = {"sub": subject, "type": token_type, "exp": expire}
    return jwt.encode(payload, settings.SECRET_KEY, algorithm=ALGORITHM)


def create_access_token(subject: str, expires_delta: timedelta) -> str:
    return _create_token(subject, "access", expires_delta)


def create_refresh_token(subject: str, expires_delta: timedelta) -> str:
    return _create_token(subject, "refresh", expires_delta)


def decode_token(token: str) -> dict:
    return jwt.decode(token, settings.SECRET_KEY, algorithms=[ALGORITHM])
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && uv run pytest tests/test_security.py -v`
Expected: `6 passed`

- [ ] **Step 5: Commit**

```bash
git add backend/app/core/security.py backend/tests/test_security.py
git commit -m "feat: add password hashing and JWT token utilities"
```

---

### Task 7: User repository

**Files:**
- Create: `backend/app/repositories/user_repository.py`
- Test: `backend/tests/test_user_repository.py`

**Interfaces:**
- Consumes: `app.models.user.User` (Task 5), `db_session` fixture (Task 4).
- Produces: `UserRepository(db: AsyncSession)` with `async get_by_email(email: str) -> User | None`, `async get_by_id(user_id: str) -> User | None`, `async create(email: str, hashed_password: str) -> User`.

- [ ] **Step 1: Write the failing tests**

```python
# backend/tests/test_user_repository.py
import pytest

from app.repositories.user_repository import UserRepository

pytestmark = pytest.mark.asyncio


async def test_create_and_get_by_email(db_session):
    repo = UserRepository(db_session)
    created = await repo.create("a@example.com", "hashed-value")

    found = await repo.get_by_email("a@example.com")

    assert found is not None
    assert found.id == created.id
    assert found.hashed_password == "hashed-value"


async def test_get_by_email_returns_none_when_missing(db_session):
    repo = UserRepository(db_session)
    assert await repo.get_by_email("nobody@example.com") is None


async def test_get_by_id_finds_created_user(db_session):
    repo = UserRepository(db_session)
    created = await repo.create("b@example.com", "hashed-value")

    found = await repo.get_by_id(str(created.id))

    assert found is not None
    assert found.email == "b@example.com"


async def test_get_by_id_returns_none_for_invalid_uuid(db_session):
    repo = UserRepository(db_session)
    assert await repo.get_by_id("not-a-uuid") is None
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && uv run pytest tests/test_user_repository.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'app.repositories.user_repository'`

- [ ] **Step 3: Write `backend/app/repositories/user_repository.py`**

```python
import uuid

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.models.user import User


class UserRepository:
    def __init__(self, db: AsyncSession):
        self.db = db

    async def get_by_email(self, email: str) -> User | None:
        result = await self.db.execute(select(User).where(User.email == email))
        return result.scalar_one_or_none()

    async def get_by_id(self, user_id: str) -> User | None:
        try:
            uid = uuid.UUID(user_id)
        except ValueError:
            return None
        result = await self.db.execute(select(User).where(User.id == uid))
        return result.scalar_one_or_none()

    async def create(self, email: str, hashed_password: str) -> User:
        user = User(email=email, hashed_password=hashed_password)
        self.db.add(user)
        await self.db.commit()
        await self.db.refresh(user)
        return user
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && uv run pytest tests/test_user_repository.py -v`
Expected: `4 passed`

- [ ] **Step 5: Commit**

```bash
git add backend/app/repositories/user_repository.py backend/tests/test_user_repository.py
git commit -m "feat: add user repository"
```

---

### Task 8: Auth service

**Files:**
- Create: `backend/app/services/auth_service.py`
- Test: `backend/tests/test_auth_service.py`

**Interfaces:**
- Consumes: `UserRepository` (Task 7), `hash_password`/`verify_password`/`create_access_token`/`create_refresh_token`/`decode_token` (Task 6), `settings` (Task 2).
- Produces: `AuthService(user_repo: UserRepository)` with `async register(email, password) -> User` (raises `EmailAlreadyRegisteredError`), `async authenticate(email, password) -> User` (raises `InvalidCredentialsError`), `issue_tokens(user: User) -> tuple[str, str]` (access, refresh), `async refresh_access_token(refresh_token: str) -> str` (raises `InvalidRefreshTokenError`). Also produces the three exception classes.

- [ ] **Step 1: Write the failing tests**

```python
# backend/tests/test_auth_service.py
import pytest

from app.repositories.user_repository import UserRepository
from app.services.auth_service import (
    AuthService,
    EmailAlreadyRegisteredError,
    InvalidCredentialsError,
    InvalidRefreshTokenError,
)

pytestmark = pytest.mark.asyncio


@pytest.fixture
def service(db_session):
    return AuthService(UserRepository(db_session))


async def test_register_creates_a_user_with_hashed_password(service):
    user = await service.register("a@example.com", "correct-horse-battery")
    assert user.email == "a@example.com"
    assert user.hashed_password != "correct-horse-battery"


async def test_register_rejects_duplicate_email(service):
    await service.register("dupe@example.com", "password123")
    with pytest.raises(EmailAlreadyRegisteredError):
        await service.register("dupe@example.com", "different-pass")


async def test_authenticate_succeeds_with_correct_password(service):
    await service.register("b@example.com", "correct-horse-battery")
    user = await service.authenticate("b@example.com", "correct-horse-battery")
    assert user.email == "b@example.com"


async def test_authenticate_rejects_wrong_password(service):
    await service.register("c@example.com", "correct-horse-battery")
    with pytest.raises(InvalidCredentialsError):
        await service.authenticate("c@example.com", "wrong-password")


async def test_authenticate_rejects_unknown_email(service):
    with pytest.raises(InvalidCredentialsError):
        await service.authenticate("nobody@example.com", "whatever")


async def test_issue_tokens_returns_access_and_refresh(service):
    user = await service.register("d@example.com", "correct-horse-battery")
    access, refresh = service.issue_tokens(user)
    assert access != refresh
    assert isinstance(access, str)
    assert isinstance(refresh, str)


async def test_refresh_access_token_returns_a_new_access_token(service):
    user = await service.register("e@example.com", "correct-horse-battery")
    _, refresh = service.issue_tokens(user)
    new_access = await service.refresh_access_token(refresh)
    assert isinstance(new_access, str)


async def test_refresh_access_token_rejects_an_access_token(service):
    user = await service.register("f@example.com", "correct-horse-battery")
    access, _ = service.issue_tokens(user)
    with pytest.raises(InvalidRefreshTokenError):
        await service.refresh_access_token(access)


async def test_refresh_access_token_rejects_garbage(service):
    with pytest.raises(InvalidRefreshTokenError):
        await service.refresh_access_token("not-a-jwt")
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && uv run pytest tests/test_auth_service.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'app.services.auth_service'`

- [ ] **Step 3: Write `backend/app/services/auth_service.py`**

```python
from datetime import timedelta

from app.core.config import settings
from app.core.security import (
    create_access_token,
    create_refresh_token,
    decode_token,
    hash_password,
    verify_password,
)
from app.models.user import User
from app.repositories.user_repository import UserRepository


class EmailAlreadyRegisteredError(Exception):
    pass


class InvalidCredentialsError(Exception):
    pass


class InvalidRefreshTokenError(Exception):
    pass


class AuthService:
    def __init__(self, user_repo: UserRepository):
        self.user_repo = user_repo

    async def register(self, email: str, password: str) -> User:
        existing = await self.user_repo.get_by_email(email)
        if existing is not None:
            raise EmailAlreadyRegisteredError(email)
        return await self.user_repo.create(email, hash_password(password))

    async def authenticate(self, email: str, password: str) -> User:
        user = await self.user_repo.get_by_email(email)
        if user is None or not verify_password(password, user.hashed_password):
            raise InvalidCredentialsError()
        return user

    def issue_tokens(self, user: User) -> tuple[str, str]:
        access = create_access_token(
            str(user.id), timedelta(minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES)
        )
        refresh = create_refresh_token(
            str(user.id), timedelta(days=settings.REFRESH_TOKEN_EXPIRE_DAYS)
        )
        return access, refresh

    async def refresh_access_token(self, refresh_token: str) -> str:
        try:
            payload = decode_token(refresh_token)
        except Exception as exc:
            raise InvalidRefreshTokenError() from exc
        if payload.get("type") != "refresh":
            raise InvalidRefreshTokenError()
        return create_access_token(
            payload["sub"], timedelta(minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES)
        )
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && uv run pytest tests/test_auth_service.py -v`
Expected: `9 passed`

- [ ] **Step 5: Commit**

```bash
git add backend/app/services/auth_service.py backend/tests/test_auth_service.py
git commit -m "feat: add auth service (register/authenticate/token issuance/refresh)"
```

---

### Task 9: Auth API (schemas, deps, routes) wired into the app

**Files:**
- Create: `backend/app/schemas/user.py`
- Create: `backend/app/schemas/auth.py`
- Create: `backend/app/api/deps.py`
- Create: `backend/app/api/routes/auth.py`
- Modify: `backend/app/main.py`
- Test: `backend/tests/test_auth_api.py`

**Interfaces:**
- Consumes: `AuthService` + its exceptions (Task 8), `UserRepository` (Task 7), `get_db` (Task 3).
- Produces: `POST /api/v1/auth/register`, `POST /api/v1/auth/login`, `POST /api/v1/auth/refresh`, `GET /api/v1/users/me` on `app.main.app`; `get_current_user` dependency reusable by future protected routes.

- [ ] **Step 1: Write the failing tests**

```python
# backend/tests/test_auth_api.py
import pytest

pytestmark = pytest.mark.asyncio


async def test_register_returns_created_user(client):
    response = await client.post(
        "/api/v1/auth/register", json={"email": "a@example.com", "password": "secretpass123"}
    )
    assert response.status_code == 201
    body = response.json()
    assert body["email"] == "a@example.com"
    assert "hashed_password" not in body


async def test_register_duplicate_email_returns_409(client):
    payload = {"email": "b@example.com", "password": "secretpass123"}
    await client.post("/api/v1/auth/register", json=payload)
    response = await client.post("/api/v1/auth/register", json=payload)
    assert response.status_code == 409


async def test_login_returns_token_pair(client):
    payload = {"email": "c@example.com", "password": "secretpass123"}
    await client.post("/api/v1/auth/register", json=payload)
    response = await client.post("/api/v1/auth/login", json=payload)
    assert response.status_code == 200
    body = response.json()
    assert "access_token" in body
    assert "refresh_token" in body


async def test_login_wrong_password_returns_401(client):
    payload = {"email": "d@example.com", "password": "secretpass123"}
    await client.post("/api/v1/auth/register", json=payload)
    response = await client.post(
        "/api/v1/auth/login", json={"email": "d@example.com", "password": "wrong"}
    )
    assert response.status_code == 401


async def test_refresh_returns_new_access_token(client):
    payload = {"email": "e@example.com", "password": "secretpass123"}
    await client.post("/api/v1/auth/register", json=payload)
    login_response = await client.post("/api/v1/auth/login", json=payload)
    refresh_token = login_response.json()["refresh_token"]

    response = await client.post("/api/v1/auth/refresh", json={"refresh_token": refresh_token})
    assert response.status_code == 200
    assert "access_token" in response.json()


async def test_users_me_requires_a_token(client):
    response = await client.get("/api/v1/users/me")
    assert response.status_code == 401


async def test_users_me_returns_the_authenticated_user(client):
    payload = {"email": "f@example.com", "password": "secretpass123"}
    await client.post("/api/v1/auth/register", json=payload)
    login_response = await client.post("/api/v1/auth/login", json=payload)
    access_token = login_response.json()["access_token"]

    response = await client.get(
        "/api/v1/users/me", headers={"Authorization": f"Bearer {access_token}"}
    )
    assert response.status_code == 200
    assert response.json()["email"] == "f@example.com"
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && uv run pytest tests/test_auth_api.py -v`
Expected: FAIL with 404s / import errors (routes don't exist yet)

- [ ] **Step 3: Write `backend/app/schemas/user.py`**

```python
import uuid

from pydantic import BaseModel, ConfigDict, EmailStr, Field


class UserCreate(BaseModel):
    email: EmailStr
    password: str = Field(min_length=8, max_length=128)


class UserRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: uuid.UUID
    email: EmailStr
```

- [ ] **Step 4: Write `backend/app/schemas/auth.py`**

```python
from pydantic import BaseModel, EmailStr


class LoginRequest(BaseModel):
    email: EmailStr
    password: str


class RefreshRequest(BaseModel):
    refresh_token: str


class TokenPair(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str = "bearer"


class AccessToken(BaseModel):
    access_token: str
    token_type: str = "bearer"
```

- [ ] **Step 5: Write `backend/app/api/deps.py`**

```python
from typing import Annotated

from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.db import get_db
from app.core.security import decode_token
from app.models.user import User
from app.repositories.user_repository import UserRepository

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/v1/auth/login", auto_error=False)


async def get_current_user(
    token: Annotated[str | None, Depends(oauth2_scheme)],
    db: Annotated[AsyncSession, Depends(get_db)],
) -> User:
    credentials_error = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED, detail="Could not validate credentials"
    )
    if token is None:
        raise credentials_error
    try:
        payload = decode_token(token)
    except Exception as exc:
        raise credentials_error from exc
    if payload.get("type") != "access":
        raise credentials_error
    user_id = payload.get("sub")
    if user_id is None:
        raise credentials_error
    user = await UserRepository(db).get_by_id(user_id)
    if user is None:
        raise credentials_error
    return user
```

- [ ] **Step 6: Write `backend/app/api/routes/auth.py`**

```python
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.ext.asyncio import AsyncSession

from app.api.deps import get_current_user
from app.core.db import get_db
from app.models.user import User
from app.repositories.user_repository import UserRepository
from app.schemas.auth import AccessToken, LoginRequest, RefreshRequest, TokenPair
from app.schemas.user import UserCreate, UserRead
from app.services.auth_service import (
    AuthService,
    EmailAlreadyRegisteredError,
    InvalidCredentialsError,
    InvalidRefreshTokenError,
)

router = APIRouter(prefix="/api/v1/auth", tags=["auth"])
users_router = APIRouter(prefix="/api/v1/users", tags=["users"])


def get_auth_service(db: AsyncSession = Depends(get_db)) -> AuthService:
    return AuthService(UserRepository(db))


@router.post("/register", response_model=UserRead, status_code=status.HTTP_201_CREATED)
async def register(payload: UserCreate, service: AuthService = Depends(get_auth_service)):
    try:
        return await service.register(payload.email, payload.password)
    except EmailAlreadyRegisteredError:
        raise HTTPException(
            status_code=status.HTTP_409_CONFLICT, detail="Email already registered"
        ) from None


@router.post("/login", response_model=TokenPair)
async def login(payload: LoginRequest, service: AuthService = Depends(get_auth_service)):
    try:
        user = await service.authenticate(payload.email, payload.password)
    except InvalidCredentialsError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid email or password"
        ) from None
    access, refresh = service.issue_tokens(user)
    return TokenPair(access_token=access, refresh_token=refresh)


@router.post("/refresh", response_model=AccessToken)
async def refresh(payload: RefreshRequest, service: AuthService = Depends(get_auth_service)):
    try:
        access = await service.refresh_access_token(payload.refresh_token)
    except InvalidRefreshTokenError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid refresh token"
        ) from None
    return AccessToken(access_token=access)


@users_router.get("/me", response_model=UserRead)
async def read_current_user(current_user: User = Depends(get_current_user)):
    return current_user
```

- [ ] **Step 7: Wire the routers and CORS into `backend/app/main.py`**

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.api.routes.auth import router as auth_router
from app.api.routes.auth import users_router
from app.core.config import settings


def create_app() -> FastAPI:
    app = FastAPI(title="Career Asset Manager API")

    app.add_middleware(
        CORSMiddleware,
        allow_origins=settings.CORS_ORIGINS.split(","),
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )

    @app.get("/health")
    async def health() -> dict[str, str]:
        return {"status": "ok"}

    app.include_router(auth_router)
    app.include_router(users_router)
    return app


app = create_app()
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `cd backend && uv run pytest -v`
Expected: all tests pass (health, config, security, repository, service, and the 7 new auth API tests — 24 total)

- [ ] **Step 9: Manually verify the running server**

Run: `cd backend && uv run uvicorn app.main:app --reload` then in another terminal:
```bash
curl -s -X POST http://localhost:8000/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"manual@example.com","password":"secretpass123"}'
```
Expected: `{"id":"...","email":"manual@example.com"}`. Stop the server (Ctrl-C) when done.

- [ ] **Step 10: Commit**

```bash
git add backend/app/schemas backend/app/api backend/app/main.py backend/tests/test_auth_api.py
git commit -m "feat: add auth API routes (register/login/refresh/me)"
```

---

### Task 10: Frontend project scaffolding

**Files:**
- Create: `frontend/` (via `create-next-app`)
- Create: `frontend/vitest.config.ts`
- Create: `frontend/vitest.setup.ts`
- Modify: `frontend/package.json`
- Test: `frontend/src/lib/sanity.test.ts`

**Interfaces:**
- Produces: a Next.js (App Router) + TypeScript + Tailwind project at `frontend/` with Vitest + React Testing Library wired and passing.

- [ ] **Step 1: Scaffold the Next.js app**

Run from the repo root:
```bash
npx create-next-app@latest frontend --typescript --tailwind --eslint --app --src-dir --import-alias "@/*" --use-npm
```
If prompted interactively, answer: TypeScript = Yes, ESLint = Yes, Tailwind = Yes, `src/` directory = Yes, App Router = Yes, import alias = `@/*`, Turbopack = No.
Expected: creates `frontend/` with `package.json`, `tsconfig.json`, `src/app/`, `tailwind.config.ts` (or `postcss`-based Tailwind v4 config), `.eslintrc`/`eslint.config.*`.

- [ ] **Step 2: Install test dependencies**

Run: `cd frontend && npm install --save-dev vitest @vitejs/plugin-react jsdom @testing-library/react @testing-library/jest-dom @testing-library/user-event`

- [ ] **Step 3: Write `frontend/vitest.config.ts`**

```ts
import path from "node:path";
import react from "@vitejs/plugin-react";
import { defineConfig } from "vitest/config";

export default defineConfig({
  plugins: [react()],
  test: {
    environment: "jsdom",
    setupFiles: ["./vitest.setup.ts"],
    globals: true,
  },
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "./src"),
    },
  },
});
```

- [ ] **Step 4: Write `frontend/vitest.setup.ts`**

```ts
import "@testing-library/jest-dom/vitest";
```

- [ ] **Step 5: Add a `test` script to `frontend/package.json`**

Add `"test": "vitest run"` to the `"scripts"` object (alongside the existing `dev`/`build`/`start`/`lint` scripts `create-next-app` already wrote).

- [ ] **Step 6: Write a failing sanity test**

```ts
// frontend/src/lib/sanity.test.ts
import { describe, expect, it } from "vitest";

describe("test setup", () => {
  it("can run a basic assertion", () => {
    expect(1 + 1).toBe(3);
  });
});
```

- [ ] **Step 7: Run it to verify it fails**

Run: `cd frontend && npm test`
Expected: FAIL — `expected 2 to be 3`

- [ ] **Step 8: Fix the assertion**

```ts
// frontend/src/lib/sanity.test.ts
import { describe, expect, it } from "vitest";

describe("test setup", () => {
  it("can run a basic assertion", () => {
    expect(1 + 1).toBe(2);
  });
});
```

- [ ] **Step 9: Run it to verify it passes**

Run: `cd frontend && npm test`
Expected: `1 passed`

- [ ] **Step 10: Commit**

```bash
git add frontend
git commit -m "chore: scaffold frontend project with Next.js and Vitest"
```

---

### Task 11: API client helper

**Files:**
- Create: `frontend/src/lib/api-client.ts`
- Test: `frontend/src/lib/api-client.test.ts`
- Delete: `frontend/src/lib/sanity.test.ts` (superseded by this task's real test)

**Interfaces:**
- Produces: `postJson<T>(path: string, body: unknown): Promise<T>` — throws `Error` with the backend's `detail` message on a non-OK response.

- [ ] **Step 1: Remove the sanity test**

Run: `rm frontend/src/lib/sanity.test.ts`

- [ ] **Step 2: Write the failing tests**

```ts
// frontend/src/lib/api-client.test.ts
import { describe, expect, it, vi } from "vitest";
import { postJson } from "./api-client";

describe("postJson", () => {
  it("returns parsed JSON on success", async () => {
    global.fetch = vi.fn().mockResolvedValue({
      ok: true,
      json: async () => ({ ok: true }),
    }) as unknown as typeof fetch;

    const result = await postJson<{ ok: boolean }>("/api/test", { a: 1 });
    expect(result).toEqual({ ok: true });
  });

  it("throws with the backend's detail message on failure", async () => {
    global.fetch = vi.fn().mockResolvedValue({
      ok: false,
      json: async () => ({ detail: "nope" }),
    }) as unknown as typeof fetch;

    await expect(postJson("/api/test", {})).rejects.toThrow("nope");
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `cd frontend && npm test`
Expected: FAIL — `Cannot find module './api-client'`

- [ ] **Step 4: Write `frontend/src/lib/api-client.ts`**

```ts
export async function postJson<T>(path: string, body: unknown): Promise<T> {
  const response = await fetch(path, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(body),
  });
  const data = await response.json();
  if (!response.ok) {
    throw new Error(data.detail ?? "Request failed");
  }
  return data as T;
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd frontend && npm test`
Expected: `2 passed`

- [ ] **Step 6: Commit**

```bash
git add frontend/src/lib/api-client.ts frontend/src/lib/api-client.test.ts
git rm frontend/src/lib/sanity.test.ts
git commit -m "feat: add typed JSON POST helper for calling Next.js API routes"
```

---

### Task 12: Theme context + toggle (dark/light mode)

**Files:**
- Create: `frontend/src/contexts/theme-context.tsx`
- Create: `frontend/src/components/theme-toggle.tsx`
- Test: `frontend/src/components/theme-toggle.test.tsx`

**Interfaces:**
- Produces: `ThemeProvider` (context provider), `useTheme() -> { theme: "light" | "dark", toggleTheme: () => void }`, `<ThemeToggle />` component.

- [ ] **Step 1: Write the failing test**

```tsx
// frontend/src/components/theme-toggle.test.tsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { describe, expect, it } from "vitest";
import { ThemeProvider } from "@/contexts/theme-context";
import { ThemeToggle } from "./theme-toggle";

describe("ThemeToggle", () => {
  it("toggles the document theme attribute when clicked", async () => {
    render(
      <ThemeProvider>
        <ThemeToggle />
      </ThemeProvider>,
    );

    expect(document.documentElement.dataset.theme).toBe("light");
    await userEvent.click(screen.getByText("Switch to dark mode"));
    expect(document.documentElement.dataset.theme).toBe("dark");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd frontend && npm test`
Expected: FAIL — `Cannot find module '@/contexts/theme-context'`

- [ ] **Step 3: Write `frontend/src/contexts/theme-context.tsx`**

```tsx
"use client";

import { createContext, useContext, useEffect, useState, type ReactNode } from "react";

type Theme = "light" | "dark";

interface ThemeContextValue {
  theme: Theme;
  toggleTheme: () => void;
}

const ThemeContext = createContext<ThemeContextValue | undefined>(undefined);

export function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState<Theme>("light");

  useEffect(() => {
    document.documentElement.dataset.theme = theme;
  }, [theme]);

  function toggleTheme() {
    setTheme((current) => (current === "light" ? "dark" : "light"));
  }

  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>{children}</ThemeContext.Provider>
  );
}

export function useTheme(): ThemeContextValue {
  const context = useContext(ThemeContext);
  if (!context) {
    throw new Error("useTheme must be used within a ThemeProvider");
  }
  return context;
}
```

- [ ] **Step 4: Write `frontend/src/components/theme-toggle.tsx`**

```tsx
"use client";

import { useTheme } from "@/contexts/theme-context";

export function ThemeToggle() {
  const { theme, toggleTheme } = useTheme();

  return (
    <button type="button" onClick={toggleTheme}>
      {theme === "light" ? "Switch to dark mode" : "Switch to light mode"}
    </button>
  );
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `cd frontend && npm test`
Expected: `1 passed` (plus the 2 from Task 11 — `3 passed`)

- [ ] **Step 6: Commit**

```bash
git add frontend/src/contexts/theme-context.tsx frontend/src/components/theme-toggle.tsx \
        frontend/src/components/theme-toggle.test.tsx
git commit -m "feat: add dark/light theme context and toggle"
```

---

### Task 13: Auth proxy Route Handlers

**Files:**
- Create: `frontend/src/app/api/auth/register/route.ts`
- Create: `frontend/src/app/api/auth/register/route.test.ts`
- Create: `frontend/src/app/api/auth/login/route.ts`
- Create: `frontend/src/app/api/auth/login/route.test.ts`
- Create: `frontend/src/app/api/auth/refresh/route.ts`
- Create: `frontend/src/app/api/auth/logout/route.ts`
- Create: `frontend/.env.local.example`

**Interfaces:**
- Consumes: backend `POST /api/v1/auth/register|login|refresh` (Task 9), `BACKEND_API_URL` env var.
- Produces: `POST /api/auth/register`, `POST /api/auth/login` (sets httpOnly `cam_refresh_token` cookie, returns only `{ access_token }`), `POST /api/auth/refresh` (reads the cookie), `POST /api/auth/logout` (clears the cookie).

- [ ] **Step 1: Write `frontend/.env.local.example`**

```
BACKEND_API_URL=http://localhost:8000
```

Then run: `cp frontend/.env.local.example frontend/.env.local`

- [ ] **Step 2: Write the failing test for register**

```ts
// frontend/src/app/api/auth/register/route.test.ts
import { NextRequest } from "next/server";
import { beforeEach, describe, expect, it, vi } from "vitest";
import { POST } from "./route";

describe("POST /api/auth/register", () => {
  beforeEach(() => {
    vi.restoreAllMocks();
  });

  it("proxies the request to the backend and returns its response", async () => {
    global.fetch = vi.fn().mockResolvedValue({
      ok: true,
      status: 201,
      json: async () => ({ id: "123", email: "a@example.com" }),
    }) as unknown as typeof fetch;

    const request = new NextRequest("http://localhost/api/auth/register", {
      method: "POST",
      body: JSON.stringify({ email: "a@example.com", password: "secretpass123" }),
    });

    const response = await POST(request);
    const data = await response.json();

    expect(response.status).toBe(201);
    expect(data.email).toBe("a@example.com");
  });
});
```

- [ ] **Step 3: Run it to verify it fails**

Run: `cd frontend && npm test`
Expected: FAIL — `Cannot find module './route'`

- [ ] **Step 4: Write `frontend/src/app/api/auth/register/route.ts`**

```ts
import { NextRequest, NextResponse } from "next/server";

const BACKEND_URL = process.env.BACKEND_API_URL ?? "http://localhost:8000";

export async function POST(request: NextRequest) {
  const body = await request.json();
  const backendResponse = await fetch(`${BACKEND_URL}/api/v1/auth/register`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(body),
  });
  const data = await backendResponse.json();
  return NextResponse.json(data, { status: backendResponse.status });
}
```

- [ ] **Step 5: Run it to verify it passes**

Run: `cd frontend && npm test`
Expected: passes (plus prior tests)

- [ ] **Step 6: Write the failing test for login**

```ts
// frontend/src/app/api/auth/login/route.test.ts
import { NextRequest } from "next/server";
import { beforeEach, describe, expect, it, vi } from "vitest";
import { POST } from "./route";

describe("POST /api/auth/login", () => {
  beforeEach(() => {
    vi.restoreAllMocks();
  });

  it("sets an httpOnly refresh cookie and returns only the access token", async () => {
    global.fetch = vi.fn().mockResolvedValue({
      ok: true,
      status: 200,
      json: async () => ({
        access_token: "access123",
        refresh_token: "refresh456",
        token_type: "bearer",
      }),
    }) as unknown as typeof fetch;

    const request = new NextRequest("http://localhost/api/auth/login", {
      method: "POST",
      body: JSON.stringify({ email: "a@example.com", password: "secretpass123" }),
    });

    const response = await POST(request);
    const data = await response.json();

    expect(data).toEqual({ access_token: "access123" });
    const cookie = response.cookies.get("cam_refresh_token");
    expect(cookie?.value).toBe("refresh456");
    expect(cookie?.httpOnly).toBe(true);
  });

  it("passes through a failed backend login as-is", async () => {
    global.fetch = vi.fn().mockResolvedValue({
      ok: false,
      status: 401,
      json: async () => ({ detail: "Invalid email or password" }),
    }) as unknown as typeof fetch;

    const request = new NextRequest("http://localhost/api/auth/login", {
      method: "POST",
      body: JSON.stringify({ email: "a@example.com", password: "wrong" }),
    });

    const response = await POST(request);
    expect(response.status).toBe(401);
  });
});
```

- [ ] **Step 7: Run it to verify it fails**

Run: `cd frontend && npm test`
Expected: FAIL — `Cannot find module './route'`

- [ ] **Step 8: Write `frontend/src/app/api/auth/login/route.ts`**

```ts
import { NextRequest, NextResponse } from "next/server";

const BACKEND_URL = process.env.BACKEND_API_URL ?? "http://localhost:8000";
const REFRESH_COOKIE = "cam_refresh_token";

export async function POST(request: NextRequest) {
  const body = await request.json();
  const backendResponse = await fetch(`${BACKEND_URL}/api/v1/auth/login`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(body),
  });
  const data = await backendResponse.json();

  if (!backendResponse.ok) {
    return NextResponse.json(data, { status: backendResponse.status });
  }

  const response = NextResponse.json({ access_token: data.access_token });
  response.cookies.set(REFRESH_COOKIE, data.refresh_token, {
    httpOnly: true,
    secure: process.env.NODE_ENV === "production",
    sameSite: "lax",
    path: "/",
    maxAge: 60 * 60 * 24 * 7,
  });
  return response;
}
```

- [ ] **Step 9: Run it to verify it passes**

Run: `cd frontend && npm test`
Expected: passes (plus prior tests)

- [ ] **Step 10: Write `frontend/src/app/api/auth/refresh/route.ts`** (no dedicated test — thin proxy exercised by the auth-context test in Task 14)

```ts
import { NextRequest, NextResponse } from "next/server";

const BACKEND_URL = process.env.BACKEND_API_URL ?? "http://localhost:8000";
const REFRESH_COOKIE = "cam_refresh_token";

export async function POST(request: NextRequest) {
  const refreshToken = request.cookies.get(REFRESH_COOKIE)?.value;
  if (!refreshToken) {
    return NextResponse.json({ detail: "No refresh token" }, { status: 401 });
  }

  const backendResponse = await fetch(`${BACKEND_URL}/api/v1/auth/refresh`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ refresh_token: refreshToken }),
  });
  const data = await backendResponse.json();
  return NextResponse.json(data, { status: backendResponse.status });
}
```

- [ ] **Step 11: Write `frontend/src/app/api/auth/logout/route.ts`**

```ts
import { NextResponse } from "next/server";

const REFRESH_COOKIE = "cam_refresh_token";

export async function POST() {
  const response = NextResponse.json({ ok: true });
  response.cookies.delete(REFRESH_COOKIE);
  return response;
}
```

- [ ] **Step 12: Commit**

```bash
git add frontend/src/app/api/auth frontend/.env.local.example
git commit -m "feat: add Next.js auth proxy routes with httpOnly refresh cookie"
```

---

### Task 14: Auth context

**Files:**
- Create: `frontend/src/contexts/auth-context.tsx`
- Test: `frontend/src/contexts/auth-context.test.tsx`

**Interfaces:**
- Consumes: `postJson` (Task 11), the `/api/auth/*` routes (Task 13).
- Produces: `AuthProvider`, `useAuth() -> { accessToken: string | null, isAuthenticated: boolean, register, login, logout }`.

- [ ] **Step 1: Write the failing test**

```tsx
// frontend/src/contexts/auth-context.test.tsx
import { render, screen, waitFor } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { beforeEach, describe, expect, it, vi } from "vitest";
import { AuthProvider, useAuth } from "./auth-context";

function Probe() {
  const { isAuthenticated, login } = useAuth();
  return (
    <div>
      <span data-testid="status">{isAuthenticated ? "in" : "out"}</span>
      <button onClick={() => login("a@example.com", "secretpass123")}>login</button>
    </div>
  );
}

describe("AuthProvider", () => {
  beforeEach(() => {
    vi.restoreAllMocks();
  });

  it("becomes authenticated after a successful login", async () => {
    global.fetch = vi.fn().mockResolvedValue({
      ok: true,
      json: async () => ({ access_token: "access123" }),
    }) as unknown as typeof fetch;

    render(
      <AuthProvider>
        <Probe />
      </AuthProvider>,
    );

    expect(screen.getByTestId("status")).toHaveTextContent("out");
    await userEvent.click(screen.getByText("login"));
    await waitFor(() => expect(screen.getByTestId("status")).toHaveTextContent("in"));
  });
});
```

- [ ] **Step 2: Run it to verify it fails**

Run: `cd frontend && npm test`
Expected: FAIL — `Cannot find module './auth-context'`

- [ ] **Step 3: Write `frontend/src/contexts/auth-context.tsx`**

```tsx
"use client";

import { createContext, useCallback, useContext, useState, type ReactNode } from "react";
import { postJson } from "@/lib/api-client";

interface AuthContextValue {
  accessToken: string | null;
  isAuthenticated: boolean;
  register: (email: string, password: string) => Promise<void>;
  login: (email: string, password: string) => Promise<void>;
  logout: () => Promise<void>;
}

const AuthContext = createContext<AuthContextValue | undefined>(undefined);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [accessToken, setAccessToken] = useState<string | null>(null);

  const register = useCallback(async (email: string, password: string) => {
    await postJson("/api/auth/register", { email, password });
  }, []);

  const login = useCallback(async (email: string, password: string) => {
    const data = await postJson<{ access_token: string }>("/api/auth/login", { email, password });
    setAccessToken(data.access_token);
  }, []);

  const logout = useCallback(async () => {
    await postJson("/api/auth/logout", {});
    setAccessToken(null);
  }, []);

  return (
    <AuthContext.Provider
      value={{ accessToken, isAuthenticated: accessToken !== null, register, login, logout }}
    >
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth(): AuthContextValue {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error("useAuth must be used within an AuthProvider");
  }
  return context;
}
```

- [ ] **Step 4: Run it to verify it passes**

Run: `cd frontend && npm test`
Expected: passes (plus prior tests)

- [ ] **Step 5: Commit**

```bash
git add frontend/src/contexts/auth-context.tsx frontend/src/contexts/auth-context.test.tsx
git commit -m "feat: add auth context (register/login/logout)"
```

---

### Task 15: Login form + page

**Files:**
- Create: `frontend/src/components/login-form.tsx`
- Test: `frontend/src/components/login-form.test.tsx`
- Create: `frontend/src/app/login/page.tsx`

**Interfaces:**
- Consumes: `useAuth` (Task 14).
- Produces: `<LoginForm />` component, `/login` page.

- [ ] **Step 1: Write the failing test**

```tsx
// frontend/src/components/login-form.test.tsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { describe, expect, it, vi } from "vitest";
import { AuthProvider } from "@/contexts/auth-context";
import { LoginForm } from "./login-form";

vi.mock("next/navigation", () => ({
  useRouter: () => ({ push: vi.fn() }),
}));

describe("LoginForm", () => {
  it("shows a validation error when submitted empty", async () => {
    render(
      <AuthProvider>
        <LoginForm />
      </AuthProvider>,
    );

    await userEvent.click(screen.getByText("Log in"));

    expect(screen.getByRole("alert")).toHaveTextContent("Email and password are required.");
  });
});
```

- [ ] **Step 2: Run it to verify it fails**

Run: `cd frontend && npm test`
Expected: FAIL — `Cannot find module './login-form'`

- [ ] **Step 3: Write `frontend/src/components/login-form.tsx`**

```tsx
"use client";

import { useRouter } from "next/navigation";
import { useState, type FormEvent } from "react";
import { useAuth } from "@/contexts/auth-context";

export function LoginForm() {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState<string | null>(null);
  const { login } = useAuth();
  const router = useRouter();

  async function handleSubmit(event: FormEvent<HTMLFormElement>) {
    event.preventDefault();
    setError(null);

    if (email.trim() === "" || password.trim() === "") {
      setError("Email and password are required.");
      return;
    }

    try {
      await login(email, password);
      router.push("/dashboard");
    } catch {
      setError("Invalid email or password.");
    }
  }

  return (
    <form onSubmit={handleSubmit} noValidate>
      <label htmlFor="email">Email</label>
      <input id="email" type="email" value={email} onChange={(e) => setEmail(e.target.value)} />

      <label htmlFor="password">Password</label>
      <input
        id="password"
        type="password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
      />

      {error && <p role="alert">{error}</p>}

      <button type="submit">Log in</button>
    </form>
  );
}
```

- [ ] **Step 4: Run it to verify it passes**

Run: `cd frontend && npm test`
Expected: passes (plus prior tests)

- [ ] **Step 5: Write `frontend/src/app/login/page.tsx`**

```tsx
import { LoginForm } from "@/components/login-form";

export default function LoginPage() {
  return (
    <main>
      <h1>Log in</h1>
      <LoginForm />
    </main>
  );
}
```

- [ ] **Step 6: Commit**

```bash
git add frontend/src/components/login-form.tsx frontend/src/components/login-form.test.tsx \
        frontend/src/app/login
git commit -m "feat: add login form and page"
```

---

### Task 16: Register form + page

**Files:**
- Create: `frontend/src/components/register-form.tsx`
- Test: `frontend/src/components/register-form.test.tsx`
- Create: `frontend/src/app/register/page.tsx`

**Interfaces:**
- Consumes: `useAuth` (Task 14).
- Produces: `<RegisterForm />` component, `/register` page.

- [ ] **Step 1: Write the failing test**

```tsx
// frontend/src/components/register-form.test.tsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { describe, expect, it, vi } from "vitest";
import { AuthProvider } from "@/contexts/auth-context";
import { RegisterForm } from "./register-form";

vi.mock("next/navigation", () => ({
  useRouter: () => ({ push: vi.fn() }),
}));

describe("RegisterForm", () => {
  it("shows a validation error when submitted empty", async () => {
    render(
      <AuthProvider>
        <RegisterForm />
      </AuthProvider>,
    );

    await userEvent.click(screen.getByText("Create account"));

    expect(screen.getByRole("alert")).toHaveTextContent("Email and password are required.");
  });

  it("shows a validation error for a short password", async () => {
    render(
      <AuthProvider>
        <RegisterForm />
      </AuthProvider>,
    );

    await userEvent.type(screen.getByLabelText("Email"), "a@example.com");
    await userEvent.type(screen.getByLabelText("Password"), "short");
    await userEvent.click(screen.getByText("Create account"));

    expect(screen.getByRole("alert")).toHaveTextContent(
      "Password must be at least 8 characters.",
    );
  });
});
```

- [ ] **Step 2: Run it to verify it fails**

Run: `cd frontend && npm test`
Expected: FAIL — `Cannot find module './register-form'`

- [ ] **Step 3: Write `frontend/src/components/register-form.tsx`**

```tsx
"use client";

import { useRouter } from "next/navigation";
import { useState, type FormEvent } from "react";
import { useAuth } from "@/contexts/auth-context";

export function RegisterForm() {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState<string | null>(null);
  const { register, login } = useAuth();
  const router = useRouter();

  async function handleSubmit(event: FormEvent<HTMLFormElement>) {
    event.preventDefault();
    setError(null);

    if (email.trim() === "" || password.trim() === "") {
      setError("Email and password are required.");
      return;
    }
    if (password.length < 8) {
      setError("Password must be at least 8 characters.");
      return;
    }

    try {
      await register(email, password);
      await login(email, password);
      router.push("/dashboard");
    } catch {
      setError("Could not create account. The email may already be registered.");
    }
  }

  return (
    <form onSubmit={handleSubmit} noValidate>
      <label htmlFor="email">Email</label>
      <input id="email" type="email" value={email} onChange={(e) => setEmail(e.target.value)} />

      <label htmlFor="password">Password</label>
      <input
        id="password"
        type="password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
      />

      {error && <p role="alert">{error}</p>}

      <button type="submit">Create account</button>
    </form>
  );
}
```

- [ ] **Step 4: Run it to verify it passes**

Run: `cd frontend && npm test`
Expected: passes (plus prior tests)

- [ ] **Step 5: Write `frontend/src/app/register/page.tsx`**

```tsx
import { RegisterForm } from "@/components/register-form";

export default function RegisterPage() {
  return (
    <main>
      <h1>Create your account</h1>
      <RegisterForm />
    </main>
  );
}
```

- [ ] **Step 6: Commit**

```bash
git add frontend/src/components/register-form.tsx frontend/src/components/register-form.test.tsx \
        frontend/src/app/register
git commit -m "feat: add register form and page"
```

---

### Task 17: Protected dashboard shell + route middleware

**Files:**
- Create: `frontend/src/lib/auth-guard.ts`
- Test: `frontend/src/lib/auth-guard.test.ts`
- Create: `frontend/src/middleware.ts`
- Create: `frontend/src/app/dashboard/page.tsx`

**Interfaces:**
- Produces: `isAuthenticatedRequest(refreshCookieValue: string | undefined) -> boolean`; Next.js middleware redirecting unauthenticated `/dashboard/*` requests to `/login`.

- [ ] **Step 1: Write the failing test**

```ts
// frontend/src/lib/auth-guard.test.ts
import { describe, expect, it } from "vitest";
import { isAuthenticatedRequest } from "./auth-guard";

describe("isAuthenticatedRequest", () => {
  it("returns false when there is no refresh cookie", () => {
    expect(isAuthenticatedRequest(undefined)).toBe(false);
  });

  it("returns false for an empty refresh cookie", () => {
    expect(isAuthenticatedRequest("")).toBe(false);
  });

  it("returns true when a non-empty refresh cookie is present", () => {
    expect(isAuthenticatedRequest("some-refresh-token")).toBe(true);
  });
});
```

- [ ] **Step 2: Run it to verify it fails**

Run: `cd frontend && npm test`
Expected: FAIL — `Cannot find module './auth-guard'`

- [ ] **Step 3: Write `frontend/src/lib/auth-guard.ts`**

```ts
export function isAuthenticatedRequest(refreshCookieValue: string | undefined): boolean {
  return typeof refreshCookieValue === "string" && refreshCookieValue.length > 0;
}
```

- [ ] **Step 4: Run it to verify it passes**

Run: `cd frontend && npm test`
Expected: passes (plus prior tests)

- [ ] **Step 5: Write `frontend/src/middleware.ts`**

```ts
import { NextRequest, NextResponse } from "next/server";
import { isAuthenticatedRequest } from "@/lib/auth-guard";

export function middleware(request: NextRequest) {
  if (!isAuthenticatedRequest(request.cookies.get("cam_refresh_token")?.value)) {
    return NextResponse.redirect(new URL("/login", request.url));
  }
  return NextResponse.next();
}

export const config = {
  matcher: ["/dashboard/:path*"],
};
```

- [ ] **Step 6: Write `frontend/src/app/dashboard/page.tsx`**

```tsx
export default function DashboardPage() {
  return (
    <main>
      <h1>Dashboard</h1>
      <p>You are logged in.</p>
    </main>
  );
}
```

- [ ] **Step 7: Commit**

```bash
git add frontend/src/lib/auth-guard.ts frontend/src/lib/auth-guard.test.ts \
        frontend/src/middleware.ts frontend/src/app/dashboard
git commit -m "feat: add protected dashboard shell and auth middleware"
```

---

### Task 18: Wire providers into the root layout

**Files:**
- Modify: `frontend/src/app/layout.tsx`

**Interfaces:**
- Consumes: `ThemeProvider` (Task 12), `AuthProvider` (Task 14).

- [ ] **Step 1: Update the root layout**

```tsx
// frontend/src/app/layout.tsx
import type { Metadata } from "next";
import { AuthProvider } from "@/contexts/auth-context";
import { ThemeProvider } from "@/contexts/theme-context";
import "./globals.css";

export const metadata: Metadata = {
  title: "Career Asset Manager",
  description: "Manage every asset in your job search from one place.",
};

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <ThemeProvider>
          <AuthProvider>{children}</AuthProvider>
        </ThemeProvider>
      </body>
    </html>
  );
}
```

- [ ] **Step 2: Run the full frontend test suite**

Run: `cd frontend && npm test`
Expected: all tests pass

- [ ] **Step 3: Run the linter**

Run: `cd frontend && npm run lint`
Expected: no errors

- [ ] **Step 4: Commit**

```bash
git add frontend/src/app/layout.tsx
git commit -m "feat: wire theme and auth providers into the root layout"
```

---

### Task 19: Root README + end-to-end manual verification

**Files:**
- Modify: `README.md`

**Interfaces:**
- None — this is the final verification pass over everything built in Tasks 1–18.

- [ ] **Step 1: Update `README.md`**

```markdown
# Career Asset Manager (CAM)

Personal career management platform — resumes, versioning, tagging, company tracking, and
application management. See `what_to_do.txt` for the full product spec and `CLAUDE.md` for
architecture/conventions.

## Prerequisites

- Postgres running locally with `cam_dev` and `cam_test` databases (see
  `docs/superpowers/plans/2026-07-12-phase1-scaffolding.md` → Prerequisites)
- `uv` (Python) and Node.js + `npm`

## Backend

```bash
cd backend
cp .env.example .env   # then edit SECRET_KEY
uv sync
uv run alembic upgrade head
uv run uvicorn app.main:app --reload
```

Runs on http://localhost:8000. Tests: `uv run pytest -v`.

## Frontend

```bash
cd frontend
cp .env.local.example .env.local
npm install
npm run dev
```

Runs on http://localhost:3000. Tests: `npm test`.
```

- [ ] **Step 2: Run the full backend test suite**

Run: `cd backend && uv run pytest -v`
Expected: all tests pass

- [ ] **Step 3: Run the full frontend test suite**

Run: `cd frontend && npm test`
Expected: all tests pass

- [ ] **Step 4: Manually verify the end-to-end flow**

With Postgres running, in one terminal: `cd backend && uv run uvicorn app.main:app --reload`.
In another: `cd frontend && npm run dev`.
In a browser, go to http://localhost:3000/register, create an account, confirm you land on
`/dashboard`. Go to http://localhost:3000/dashboard directly in a fresh incognito window (no
cookie) and confirm you're redirected to `/login`. Toggle dark/light mode and confirm
`document.documentElement`'s `data-theme` attribute changes (check via browser devtools).

- [ ] **Step 5: Commit**

```bash
git add README.md
git commit -m "docs: add setup instructions for backend and frontend"
```

---

## Self-Review Notes

- **Spec coverage:** Every item in the design doc's backend/frontend scaffold sections maps to a
  task: repo layout (Task 1), config (Task 2), DB/models/migrations (Tasks 3, 5), auth logic
  (Tasks 6–9), frontend scaffold + theme + auth UI + protected routes (Tasks 10–18), setup docs
  (Task 19). Explicitly-out-of-scope items (Resume Manager, Docker, forgot-password, etc.) have no
  tasks, by design.
- **Type/name consistency checked:** `UserRepository.get_by_id`/`get_by_email`/`create`,
  `AuthService.register`/`authenticate`/`issue_tokens`/`refresh_access_token`, and the three
  exception names are used identically across Tasks 7, 8, and 9. `postJson` (Task 11) is imported
  by name into `auth-context.tsx` (Task 14). `isAuthenticatedRequest` (Task 17) matches its usage
  in `middleware.ts`. The `cam_refresh_token` cookie name is consistent across Tasks 13 and 17.
- **No placeholders:** every step has runnable code or an exact command with expected output.
