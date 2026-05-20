---
name: fastapi
description: >
  Use this skill when the user is actively working on a FastAPI codebase or
  design problem — specifically: writing or reviewing async endpoints, setting
  up SQLAlchemy 2.0+ async sessions, implementing JWT/OAuth2 auth, writing
  pytest-asyncio tests, optimising slow routes, offloading background work,
  or architecting a new FastAPI service from scratch. Do NOT use for
  pure Django/Flask questions, frontend-only work, infrastructure questions
  with no API component, or general Python questions unrelated to an API.
risk: low
source: curated
date_added: '2026-05-20'
---

## Use this skill when

- Writing or scaffolding FastAPI endpoints, routers, or middleware
- Designing or reviewing async SQLAlchemy 2.0 models, sessions, or queries
- Implementing authentication: JWT, OAuth2, API keys, RBAC
- Writing async tests with pytest and pytest-asyncio
- Diagnosing or optimising slow/blocking routes and database queries
- Offloading background or async work (BackgroundTasks or ARQ)
- Architecting a new FastAPI service: layout, DI, lifespan, settings

## Do not use this skill when

- The question is about Django, Flask, or another framework unrelated to FastAPI
- The task is frontend-only (React, Vue, CSS)
- The question is purely infrastructure (Kubernetes YAML, Terraform) with no
  FastAPI code involved
- The user just wants a general Python answer with no API context

---

## Reference files — read on demand

This SKILL.md covers the **spine** every task touches: async engine + lifespan,
settings, CORS, shared enums, performance spot-checks, project layout, and the
response approach. For everything else, read the matching reference **before**
writing code:

| If the task involves… | Read |
|---|---|
| Initialising a new project, uv commands, `pyproject.toml` | `references/project-setup.md` |
| Designing models, queries, repositories, fixing N+1 | `references/database.md` |
| Implementing JWT auth, password hashing, current-user deps, RBAC | `references/auth.md` |
| Designing `ErrorResponse`, exception handlers, auditing `HTTPException` use | `references/error-handling.md` |
| Pydantic schemas, GET/POST/PATCH/DELETE patterns | `references/response-models.md` |
| Cursor or offset pagination on list endpoints | `references/pagination.md` |
| Offloading work to background tasks or ARQ workers | `references/background-tasks.md` |
| Adding Redis caching | `references/cache.md` |
| Adding rate limits (slowapi) | `references/rate-limiting.md` |
| Introducing or retiring an API version | `references/versioning.md` |
| Setting up structured logging with structlog + request IDs | `references/logging.md` |
| Writing endpoint tests (single-route, in-process, fast) | `references/testing.md` |
| Writing E2E tests (real server, real DB/Redis, ARQ workers, CORS, migrations, CI) | `references/e2e-testing.md` |

Read the reference file in full — they contain working patterns, not hints.

**Tightly coupled pairs.** For auth questions, read **both** `auth.md` and
`error-handling.md` — the auth dependencies raise domain exceptions that only
become proper HTTP responses through the registered handlers.

**Two testing layers.** `testing.md` covers endpoint tests (fast, in-process,
swap DB session per test) — every endpoint should have these. `e2e-testing.md`
covers end-to-end tests (real ASGI server, real Postgres, real Redis, real
lifespan, real ARQ worker) — every critical user journey should additionally
have one of these.

---

## Requirements

`FastAPI ≥ 0.136.1 · Pydantic V2 · SQLAlchemy 2.0 async · Python ≥ 3.10 · uv`.
Patterns here rely on `lifespan=`, `Annotated` deps, `Mapped[]` /
`mapped_column`, `model_dump()` / `model_validate()`, and `async_sessionmaker`
— earlier releases will produce confusing errors. Python 3.12 recommended.

---

## Response approach

1. **Start async-first.** Every endpoint, database call, and external I/O must
   use `async def` and `await`. Never suggest sync alternatives unless the user
   explicitly asks or a library forces it.
2. **Design Pydantic models before endpoints.** See `response-models.md`.
3. **Use dependency injection for shared resources.** DB sessions, current user,
   settings, and HTTP clients all belong in `Depends()`, never as module-level
   globals.
4. **Handle errors with the shared `ErrorResponse` contract.** See
   `error-handling.md`. Never raise bare `HTTPException` in business logic;
   never let FastAPI's default `{"detail": "..."}` reach clients.
5. **Write the test alongside the code.** Endpoint test for every non-trivial
   route (`testing.md`). E2E test for every critical journey (`e2e-testing.md`).
6. **Call out performance risks inline.** Flag N+1 queries, missing `await`,
   or blocking calls with a short `# ⚠` comment.
7. **Never return unbounded collections.** Every list endpoint must paginate —
   see `pagination.md`.

---

## Async spine — engine, session, lifespan

The async engine + `get_db` dependency + lifespan are the foundation every
other reference builds on. Define them once, here:

```python
# app/core/database.py
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from typing import AsyncGenerator

engine = create_async_engine(
    settings.database_url,
    pool_size=10, max_overflow=20, pool_timeout=30, pool_recycle=1800,
)
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        yield session
```

```python
# app/main.py
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # startup
    yield
    # shutdown
    await engine.dispose()

app = FastAPI(lifespan=lifespan)
```

### Lifespan-managed resource pattern

For any pooled resource (HTTP client, Redis pool, ARQ pool): create once in
`lifespan`, store on `app.state`, expose via a `Depends()` that yields from
`request.app.state`. Never create new clients per request; never store as
module-level globals.

```python
# example: httpx. Same shape for redis (cache.md), arq (background-tasks.md).
@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.http_client = httpx.AsyncClient(timeout=10.0)
    yield
    await app.state.http_client.aclose()

async def get_http_client(request: Request) -> AsyncGenerator[httpx.AsyncClient, None]:
    yield request.app.state.http_client
```

Always `httpx`, never `requests`. Always `await asyncio.sleep`, never
`time.sleep`.

> **Lifespan + tests gotcha.** Lifespan calls `get_settings()` directly, not
> through `Depends()`, so `app.dependency_overrides` does **not** affect
> lifespan-loaded resources. Tests that need real lifespan (E2E) must set
> env vars and `cache_clear()` the settings — see `e2e-testing.md`.

---

## Settings management

Use `pydantic-settings`. Never read `os.environ` directly in routes — inject
settings via `Depends()`.

```python
# app/core/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict
from functools import lru_cache

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", case_sensitive=False)

    database_url: str
    secret_key: str
    access_token_expire_minutes: int = 15
    refresh_token_expire_days: int = 7
    redis_url: str | None = None
    cors_origins: list[str] = []           # never ["*"] in prod
    environment: str = "development"
    debug: bool = False

@lru_cache
def get_settings() -> Settings:
    return Settings()
```

```python
# ✅ correct — injected, testable
@router.get("/health")
async def health(settings: Settings = Depends(get_settings)) -> dict:
    return {"env": settings.environment}

# ❌ wrong — module-level global, untestable, leaks across tests
settings = Settings()
```

Override per-test:

```python
app.dependency_overrides[get_settings] = lambda: Settings(...)
```

---

## CORS

Configure with an allowlist driven from settings. **Never use `["*"]` in
production** — it's incompatible with credentialed requests (browsers reject
the combination).

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.cors_origins,      # e.g. ["https://app.example.com"]
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"],
    allow_headers=["Authorization", "Content-Type", "X-Request-ID"],
    expose_headers=["X-Request-ID"],
    max_age=600,
)
```

- **dev** — `["http://localhost:3000", "http://localhost:5173"]`
- **staging/prod** — explicit production domains, never wildcards

E2E tests for preflight live in `e2e-testing.md`.

---

## Shared enums and types

Put cross-cutting types in one canonical module so every layer imports
from the same place:

```python
# app/core/enums.py
from enum import StrEnum

class Role(StrEnum):
    ADMIN = "admin"
    EDITOR = "editor"
    VIEWER = "viewer"
```

`Role` is used by the User model (`database.md`), the RBAC dependency
(`auth.md`), and test fixtures (`testing.md`). Always import from
`app.core.enums`.

---

## Performance spot-checks

Always perform on any endpoint:

- No `time.sleep` or sync `requests` inside `async def` — use `await asyncio.sleep` and `httpx.AsyncClient`
- No missing `await` before `db.execute()` / `db.commit()`
- No default `lazy="lazy"` on relationships accessed in the same request — use `selectin` or `joined` (see `database.md`)
- No unbounded list endpoints — see `pagination.md` for cursor / offset patterns and the `MAX_PAGE_SIZE` cap

---

## Project layout (recommended)

```
app/
├── main.py              # FastAPI() instance, lifespan, middleware, router registration
├── core/
│   ├── config.py        # Pydantic Settings + get_settings()
│   ├── enums.py         # Role and other shared enums
│   ├── exceptions.py    # AppError subclasses (see auth.md + error-handling.md)
│   ├── security.py      # JWT helpers, password hashing, get_current_user (see auth.md)
│   ├── database.py      # engine, AsyncSessionLocal, get_db
│   ├── http.py          # get_http_client (httpx pool)
│   ├── arq.py           # get_arq (ARQ pool, if used — see background-tasks.md)
│   ├── cache.py         # get_redis (optional — see cache.md)
│   ├── rate_limit.py    # slowapi Limiter (see rate-limiting.md)
│   ├── logging.py       # structlog configuration (see logging.md)
│   └── exception_handlers.py  # register_exception_handlers (see error-handling.md)
├── middleware/
│   └── request_id.py    # RequestIDMiddleware (see logging.md)
├── models/              # SQLAlchemy ORM models (see database.md)
├── schemas/             # Pydantic V2 schemas (see response-models.md) + ErrorResponse
├── repositories/        # async data-access layer (see database.md)
├── routers/
│   ├── v1/              # versioned routers (see versioning.md)
│   └── v2/
├── tasks/               # background task modules (see background-tasks.md)
│   ├── email.py
│   └── worker.py        # ARQ WorkerSettings
└── tests/
    ├── conftest.py
    ├── routers/         # endpoint tests (see testing.md)
    └── e2e/             # E2E tests (see e2e-testing.md)
        ├── conftest.py
        └── test_*.py
```
