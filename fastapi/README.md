# fastapi skill

Opinionated patterns for building production FastAPI services. Async-first
throughout, with SQLAlchemy 2.0, Pydantic V2, PyJWT, ARQ for background tasks,
and pytest-asyncio for testing.

## When this triggers

Claude consults this skill when you're working on a FastAPI codebase or design
problem — writing endpoints, designing async DB sessions, implementing
JWT/OAuth2 auth, writing tests, optimising slow routes, offloading background
work, or architecting a new service.

It does **not** trigger for Django, Flask, frontend-only work, or general
Python questions unrelated to an API.

## Layout

```
fastapi/
├── SKILL.md                          # Foundation: async spine (engine, lifespan,
│                                     # DI), settings, CORS, shared enums,
│                                     # performance, project layout
└── references/
    ├── project-setup.md              # uv init, dependencies, pyproject.toml
    ├── database.md                   # models, queries, repositories, N+1
    ├── auth.md                       # JWT, bcrypt, current-user, RBAC, domain exceptions
    ├── error-handling.md             # ErrorResponse, exception handlers, 422 override
    ├── response-models.md            # Pydantic schemas, GET/POST/PATCH/DELETE patterns
    ├── pagination.md                 # cursor + offset patterns
    ├── background-tasks.md           # BackgroundTasks + ARQ
    ├── cache.md                      # Redis caching
    ├── rate-limiting.md              # slowapi
    ├── versioning.md                 # URL versioning, deprecation, retirement
    ├── logging.md                    # structlog + request ID middleware
    ├── testing.md                    # endpoint tests — pytest-asyncio fixtures + patterns
    └── e2e-testing.md                # E2E tests — real server, DB, Redis, ARQ worker, CI
```

`SKILL.md` is always loaded when the skill triggers. The `references/` files
are loaded only when the task matches a pointer in `SKILL.md`, keeping
context lean for unrelated questions.

## Opinions baked in

- **uv** for package management (never pip / pip-tools)
- **Async everywhere** — every endpoint, every DB call, every external I/O
- **Pydantic V2** with explicit `response_model=` on every route
- **PyJWT** for auth (avoid the unmaintained `python-jose`)
- **bcrypt direct** for password hashing (avoid the unmaintained `passlib`)
- **ARQ** for background tasks (no Celery — sync workers conflict with the async-first stance)
- **Cursor pagination by default** with a hard `MAX_PAGE_SIZE` cap
- **Unified `ErrorResponse`** schema — FastAPI's default `{"detail": "..."}` never reaches clients
- **Domain exception classes** raised in routes; HTTP concerns isolated to handlers

## Requirements

```
FastAPI ≥ 0.136.1 · Pydantic V2 (≥ 2.0) · SQLAlchemy ≥ 2.0 · Python ≥ 3.10 · uv ≥ 0.4
```

## Version

`date_added: 2026-05-20`
