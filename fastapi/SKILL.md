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
- Architecting a new FastAPI service: project layout, dependency injection,
  lifespan events, settings management

## Do not use this skill when

- The question is about Django, Flask, or another framework unrelated to FastAPI
- The task is frontend-only (React, Vue, CSS)
- The question is purely infrastructure (Kubernetes YAML, Terraform) with no
  FastAPI code involved
- The user just wants a general Python answer with no API context

---

## Reference files — read on demand

This SKILL.md covers project setup, async patterns, settings, CORS, shared
enums, database/ORM basics, Redis caching, rate limiting, API versioning,
performance spot-checks, and project layout. For deeper topics, read the
matching reference file **before** writing code:

| If the task involves… | Read |
|---|---|
| Implementing JWT auth, password hashing, current-user deps, or RBAC | `references/auth.md` |
| Designing `ErrorResponse`, exception handlers, or auditing `HTTPException` use | `references/error-handling.md` |
| Pydantic response/request schemas, GET/POST/PATCH/DELETE route patterns | `references/response-models.md` |
| Writing endpoint tests (single-route, in-process, fast) | `references/testing.md` |
| Writing E2E tests (real server, real DB/Redis, user journeys, ARQ workers, CORS preflight, migrations, CI) | `references/e2e-testing.md` |
| Offloading work to background tasks or ARQ workers | `references/background-tasks.md` |
| Setting up structured logging with structlog + request IDs | `references/logging.md` |
| Cursor or offset pagination on list endpoints | `references/pagination.md` |

Read the reference file in full before answering — these contain working
patterns, not just hints. Don't reconstruct from memory.

**Tightly coupled pairs.** For auth questions, read **both** `auth.md` and
`error-handling.md` — the auth dependencies raise domain exceptions that
only become proper HTTP responses through the registered handlers.

**Two testing layers.** `testing.md` covers endpoint tests (fast, in-process,
swap DB session per test) — every endpoint should have these. `e2e-testing.md`
covers end-to-end tests (real ASGI server, real Postgres, real Redis, real
lifespan, real ARQ worker) — every critical user journey should additionally
have one of these.

---

## Requirements

```
FastAPI ≥ 0.136.1 · Pydantic V2 (≥ 2.0) · SQLAlchemy ≥ 2.0 · Python ≥ 3.10 · uv ≥ 0.4
```

These patterns rely on APIs introduced in these versions — earlier releases
will produce confusing errors. Specifically: `lifespan=` (FastAPI 0.93+),
`Annotated` dependencies (0.95+), `mapped_column` + `Mapped[]` (SQLAlchemy
2.0), `model_dump()` / `model_validate()` (Pydantic V2), `async_sessionmaker`
(SQLAlchemy 2.0). Python 3.12 is the recommended runtime for new projects.

Use **uv** as the package manager throughout — never `pip` or `pip-tools`.
It's 10–100x faster, manages the virtualenv automatically, and produces a
lockfile that guarantees reproducible installs across dev, CI, and production.

---

## Project setup (greenfield)

```bash
uv init my-api
cd my-api
uv python pin 3.12
```

**Runtime dependencies:**

```bash
uv add "fastapi[standard]>=0.136.1"   # includes uvicorn + fastapi-cli
uv add "sqlalchemy[asyncio]>=2.0" asyncpg
uv add "pydantic-settings>=2.0"
uv add "PyJWT>=2.8" "bcrypt>=4.0"     # see auth.md
uv add httpx                          # async HTTP client
```

**Optional, add only what you need:**

```bash
uv add redis arq                      # caching + async task queue
uv add structlog                      # JSON logging
uv add slowapi                        # rate limiting
```

**Dev/test dependencies** (full setup in `testing.md` and `e2e-testing.md`):

```bash
uv add --dev pytest pytest-asyncio httpx anyio pytest-cov pytest-mock
uv add --dev asgi-lifespan            # E2E — actually run lifespan in tests
uv add --dev ruff mypy
```

**`pyproject.toml` tool config:**

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]

[tool.ruff]
line-length = 88
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "ASYNC"]   # ASYNC catches blocking calls in async fns

[tool.mypy]
python_version = "3.12"
strict = true
plugins = ["pydantic.mypy"]
```

**Run:**

```bash
uv run fastapi dev app/main.py        # hot-reload, OpenAPI at /docs
uv add <pkg>                          # never edit pyproject.toml by hand
uv remove <pkg>
uv sync                               # re-sync venv after pulling
```

> **Auth library choice (full details in `auth.md`):** use `PyJWT` + `bcrypt`
> for rolling your own. Use `fastapi-users` if you want registration, email
> verification, and password reset out of the box. Avoid `python-jose` and
> `passlib` — both unmaintained.

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

## Async patterns

**Async SQLAlchemy session factory:**

```python
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/db",
    pool_size=10,
    max_overflow=20,
    pool_timeout=30,
    pool_recycle=1800,
)
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        yield session
```

**Lifespan for startup/shutdown:**

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # startup
    yield
    # shutdown
    await engine.dispose()

app = FastAPI(lifespan=lifespan)
```

**The lifespan-managed resource pattern.** For any pooled resource (HTTP
client, Redis pool, ARQ pool): create once in `lifespan`, store on
`app.state`, expose via a `Depends()` that yields from `request.app.state`.
Never create new clients per request, never store as module-level globals.

```python
# example: httpx client. Same shape for redis, arq, etc.
@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.http_client = httpx.AsyncClient(timeout=10.0)
    yield
    await app.state.http_client.aclose()

async def get_http_client(request: Request) -> AsyncGenerator[httpx.AsyncClient, None]:
    yield request.app.state.http_client

@router.get("/external")
async def fetch_external(client: httpx.AsyncClient = Depends(get_http_client)) -> dict:
    response = await client.get("https://api.example.com/data")
    return response.json()
```

Always `httpx`, never `requests`. Always `await asyncio.sleep`, never
`time.sleep`.

---

## Settings management

Use `pydantic-settings`. Never read `os.environ` directly in routes —
inject settings via `Depends()`.

```python
# app/core/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict
from functools import lru_cache

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
    )

    # database
    database_url: str
    db_pool_size: int = 10
    db_max_overflow: int = 20

    # auth — see auth.md
    secret_key: str
    access_token_expire_minutes: int = 15
    refresh_token_expire_days: int = 7

    # redis (optional)
    redis_url: str | None = None

    # cors — explicit allowlist, never ["*"] in prod
    cors_origins: list[str] = []

    # app
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

Override in tests:

```python
app.dependency_overrides[get_settings] = lambda: Settings(
    database_url="postgresql+asyncpg://user:pass@localhost/test_db",
    secret_key="test-secret",
)
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
    allow_origins=settings.cors_origins,       # e.g. ["https://app.example.com"]
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"],
    allow_headers=["Authorization", "Content-Type", "X-Request-ID"],
    expose_headers=["X-Request-ID"],
    max_age=600,
)
```

Per-environment:
- **development** — `["http://localhost:3000", "http://localhost:5173"]`
- **staging/production** — explicit production domains, never wildcards

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

```python
from app.core.enums import Role   # use this import everywhere
```

`Role` is used by the User model, the RBAC dependency in `auth.md`, and
test fixtures in `testing.md`.

---

## Database & ORM (SQLAlchemy 2.0 async)

**Model definition:**

```python
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship
from sqlalchemy import ForeignKey, String, func
import datetime

from app.core.enums import Role

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    role: Mapped[str] = mapped_column(String(50), default=Role.VIEWER)
    hashed_password: Mapped[str] = mapped_column(String(255))
    created_at: Mapped[datetime.datetime] = mapped_column(server_default=func.now())
    posts: Mapped[list["Post"]] = relationship(back_populates="author", lazy="selectin")
```

**Async queries — avoid N+1 with `selectinload` / `joinedload`:**

```python
from sqlalchemy import select
from sqlalchemy.orm import selectinload

async def get_user_with_posts(db: AsyncSession, user_id: int) -> User | None:
    result = await db.execute(
        select(User)
        .where(User.id == user_id)
        .options(selectinload(User.posts))   # ✅ not lazy-loaded on access
    )
    return result.scalar_one_or_none()
```

**Repository pattern — keep DB access out of route handlers:**

```python
class UserRepository:
    def __init__(self, db: AsyncSession):
        self.db = db

    async def get_by_email(self, email: str) -> User | None:
        result = await self.db.execute(select(User).where(User.email == email))
        return result.scalar_one_or_none()

    async def create(self, email: str, hashed_password: str) -> User:
        user = User(email=email, hashed_password=hashed_password)
        self.db.add(user)
        await self.db.commit()
        await self.db.refresh(user)
        return user
```

---

## Redis caching (optional)

Same lifespan-managed resource pattern as `httpx`:

```python
# app/core/cache.py
import redis.asyncio as aioredis
import json
from fastapi import Request

async def get_redis(request: Request) -> AsyncGenerator[aioredis.Redis, None]:
    yield request.app.state.redis

async def get_cached(key: str, ttl: int, fetch_fn, redis: aioredis.Redis):
    cached = await redis.get(key)
    if cached:
        return json.loads(cached)
    value = await fetch_fn()
    await redis.setex(key, ttl, json.dumps(value))
    return value
```

```python
# in lifespan
app.state.redis = aioredis.from_url(settings.redis_url, decode_responses=True)
# at shutdown
await app.state.redis.aclose()
```

---

## Rate limiting

`slowapi` — apply at the router level, not inside handlers.

```python
# app/core/rate_limit.py
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)
```

```python
# app/main.py
from slowapi import _rate_limit_exceeded_handler
from slowapi.errors import RateLimitExceeded
from app.core.rate_limit import limiter

app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)
```

```python
@router.get("/items")
@limiter.limit("60/minute")
async def list_items(request: Request, ...):
    ...

@router.post("/auth/token")
@limiter.limit("5/minute")              # tight limit on auth — prevents brute force
async def login(request: Request, ...):
    ...
```

---

## API versioning

Default to URL versioning (`/v1/`) — explicit, cacheable, easy to route at
the infrastructure level.

```python
# app/routers/v1/users.py
router = APIRouter(prefix="/users", tags=["users"])

# app/main.py
app.include_router(users_v1.router, prefix="/v1")
app.include_router(users_v2.router, prefix="/v2")
```

**Rules:**
- New version only for breaking changes (field removal, type change, behaviour change)
- Additive changes (new optional fields, new endpoints) don't need a new version
- Mark deprecated endpoints with `deprecated=True` so they appear in OpenAPI

```python
@router.get("/{user_id}", deprecated=True, response_model=UserOutV1)
async def get_user_v1(...):
    ...
```

---

## Performance spot-checks

Always perform on any endpoint:

- No `time.sleep` or sync `requests` inside `async def` — use `await asyncio.sleep` and `httpx.AsyncClient`
- No missing `await` before `db.execute()` / `db.commit()`
- No default `lazy="lazy"` on relationships accessed in the same request — use `selectin` or `joined`
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
│   ├── arq.py           # get_arq (ARQ pool, if used)
│   ├── cache.py         # get_redis (optional)
│   ├── rate_limit.py    # slowapi Limiter instance
│   ├── logging.py       # structlog configuration (see logging.md)
│   └── exception_handlers.py  # register_exception_handlers (see error-handling.md)
├── middleware/
│   └── request_id.py    # RequestIDMiddleware (see logging.md)
├── models/              # SQLAlchemy ORM models
├── schemas/             # Pydantic V2 schemas (see response-models.md) + ErrorResponse
├── repositories/        # async data-access layer
├── routers/
│   ├── v1/              # versioned routers
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
