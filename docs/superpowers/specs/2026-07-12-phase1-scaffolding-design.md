# Phase 1 Scaffolding — Design

Date: 2026-07-12
Status: Approved

## Purpose

Before implementing Phase 1 features (Auth, Resume Manager, Version Control, Tags — per
`what_to_do.txt`), stand up the project scaffolding: repo structure, backend/frontend skeletons,
tooling, and one real working vertical slice (auth) that proves the whole stack is wired together
correctly. Resume Manager, Version Control, and Tags are separate implementation plans that build
on top of this scaffold.

## Decisions

| Area | Decision |
|---|---|
| Repo structure | Monorepo, single git repo, `backend/` + `frontend/` at the top level |
| Local Postgres | Installed locally on the dev machine, connected via `.env` `DATABASE_URL`; Docker Compose deferred to Phase 5 per spec |
| Backend package manager | `uv` |
| Frontend package manager | `npm` |
| Resume/file storage | Local disk, behind a `StorageService` interface (save/get/delete) so swapping to S3-compatible storage later doesn't require touching call sites |
| Testing | Backend: `pytest` + `pytest-asyncio` + `httpx.AsyncClient`. Frontend: Vitest + React Testing Library |
| Auth scope in scaffolding | Real, working: register, login, password hashing, JWT access + refresh tokens, protected `/users/me` endpoint. Forgot-password, email verification, and OAuth are explicitly deferred — they need an email-provider decision that doesn't belong in scaffolding |

## Backend (`backend/`)

- Layout: `app/api/`, `app/services/`, `app/repositories/`, `app/models/` (SQLAlchemy), `app/schemas/`
  (Pydantic), `app/core/` (config, security), `alembic/`, `tests/` — matches the layered
  architecture already committed to in `CLAUDE.md`.
- DB: async SQLAlchemy 2.0 with `asyncpg`. Alembic initialized with one migration creating a
  `users` table.
- Auth: `passlib`/`bcrypt` password hashing, JWT access (short-lived) + refresh (longer-lived)
  token pair, dependency-injected auth guard proven via `/users/me`.
- Config: `pydantic-settings` reading `.env`; `.env.example` committed.
- Tooling: `ruff` (lint + format) and `mypy`, configured in `pyproject.toml` under `uv`.
- Test proof: register → login → call `/users/me` with the access token, asserting the full loop
  works.

## Frontend (`frontend/`)

- Next.js (App Router) + TypeScript + Tailwind CSS, dark/light mode toggle.
- Real login/register pages wired to the backend endpoints above.
- Minimal authenticated dashboard shell (empty state cards) — exists only to prove the JWT
  round-trip and protected-route redirect behavior, not to implement the real dashboard (that's
  Phase 4).
- Typed `fetch`-based API client with access-token storage and refresh-token handling.
- ESLint + Prettier.
- Test proof: render the login form, assert a validation error appears on empty submit.

## Explicitly out of scope

- Resume Manager, Version Control, Tags, Company Database, and everything past Phase 1 in
  `what_to_do.txt` — each gets its own design/plan after this scaffold lands.
- Docker / Docker Compose / CI (Phase 5 per spec).
- Forgot-password, email verification, OAuth (need an email-provider decision first).
- Role-based access control beyond a data model that *can* support Admin/Editor/Viewer later.

## Risks / notes

- Storing the refresh token: for scaffolding, use an httpOnly cookie for the refresh token and
  in-memory (not localStorage) for the access token, to avoid an XSS-exfiltration footgun from day
  one rather than retrofitting it later.
- No CI is being set up yet, so "done" for this scaffold is defined as: both apps run locally,
  migrations apply cleanly against a fresh local Postgres DB, and both test-proof suites pass.
