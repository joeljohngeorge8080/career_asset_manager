# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

This repository currently contains only `what_to_do.txt`, the full product/technical spec for
**Career Asset Manager (CAM)**. No code has been written yet. `what_to_do.txt` is the source of
truth for scope, architecture, and conventions — read it in full before starting implementation,
and treat anything below as a summary extracted from it, not a replacement for it.

Before writing any code, per `what_to_do.txt`, produce (in this order): system architecture,
folder structure, database schema, API design, component hierarchy, and implementation roadmap.
Only then implement, module by module, following the phased plan below.

## Intended stack

- **Backend**: FastAPI, SQLAlchemy, Alembic (migrations), Pydantic, JWT auth, PostgreSQL,
  Background Tasks, Redis (optional), Celery (future-ready).
- **Frontend**: React + Next.js, TypeScript, Tailwind CSS, dark/light mode, responsive SaaS-style
  dashboard (design inspiration: Linear, Notion, Vercel, GitHub, Stripe Dashboard — minimal,
  premium, generous whitespace, no glassmorphism/gradient overload).
- **DevOps**: Docker, Docker Compose, GitHub Actions, Nginx reverse proxy, env-based config,
  structured logging, health checks.

## Architecture requirements

- Clean/layered architecture with hard separation between: API routes, business logic, services,
  repositories, models, schemas, utils, configuration, tests, static assets.
- Single-user now, but the auth/permissions model must be designed to support Admin/Editor/Viewer
  roles later without rework — don't hardcode single-user assumptions into the data model or
  authorization checks.
- Every resume edit must produce a new version (git-style diff: added/removed/modified sections),
  not an in-place mutation — version history, rollback, and version comparison are core, not
  optional add-ons.
- Company, job-description, and resume records are heavily cross-referenced (matching engine, skill
  gap analysis, ATS scoring all read across these three) — keep their schemas and services
  decoupled but expect join-heavy queries across them.

## Implementation phases (build in this order)

1. Auth, Resume Manager, Version Control, Tags
2. Company Database, Job Description Analyzer, Resume Matching
3. ATS Checker, Skill Gap Analysis, Application Tracker
4. Dashboard, Analytics, Search
5. Docker, CI/CD, Logging, Deployment, Performance Optimization

## Coding rules (from spec)

- Production-quality only — this is not a CRUD student project.
- Strong typing throughout (TypeScript on frontend, Pydantic/type hints on backend).
- No duplicated code, no unnecessary abstractions; follow SOLID where it actually applies.
- Schema changes go through Alembic migrations, never manual DDL.
