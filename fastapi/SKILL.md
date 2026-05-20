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

This SKILL.md covers the foundation that nearly every FastAPI question touches:
setup, async patterns, settings, auth, error handling, DB, response models,
CORS, rate limiting, versioning, and the basic performance rules. For
specialised topics, read the matching reference file **before** writing code:

| If the task involves… | Read |
|---|---|
| Writing endpoint tests (single-route, in-process, fast) | `references/testing.md` |
| Writing E2E tests (real server, real DB/Redis, user journeys, ARQ workers, CORS preflight, migrations, CI) | `references/e2e-testing.md` |
| Offloading work to background tasks or ARQ workers | `references/background-tasks.md` |
| Setting up structured logging with structlog + request IDs | `references/logging.md` |
| Cursor or offset pagination on list endpoints | `references/pagination.md` |

Read the reference file in full before answering — these contain working
patterns, not just hints. Don't reconstruct from memory.

**Two testing layers.** `testing.md` covers endpoint tests (fast, in-process,
swap DB session per test) — every endpoint should have these. `e2e-testing.md`
covers end-to-end tests (real ASGI server, real Postgres, real Redis, real
lifespan, real ARQ worker) — every critical user journey should additionally
have one of these. Run both; they catch different failures.

---

## Requirements

```
FastAPI ≥ 0.136.1 · Pydantic V2 (≥ 2.0) · SQLAlchemy ≥ 2.0 · Python ≥ 3.10 · uv ≥ 0.4
```

These patterns rely on APIs introduced in these versions — earlier releases will
produce confusing errors. Specifically: `lifespan=` (FastAPI 0.93+),
`Annotated` dependencies (0.95+), `mapped_column` + `Mapped[]` (SQLAlchemy 2.0),
`model_dump()` / `model_validate()` (Pydantic V2), `async_sessionmaker`
(SQLAlchemy 2.0). Python 3.12 is the recommended runtime for new projects.

Use **uv** as the package manager throughout — never `pip` or `pip-tools`.
`uv` is 10–100x faster, manages the virtualenv automatically, and produces
a lockfile (`uv.lock`) that guarantees reproducible installs across dev,
CI, and production.

---

## Project setup (greenfield)

**1. Initialise the project:**

```bash
uv init my-api
cd my-api
uv python pin 3.12   # lock the interpreter version for the project
```

**2. Add runtime dependencies:**

```bash
uv add "fastapi[standard]>=0.136.1"   # includes uvicorn + fastapi-cli
uv add "sqlalchemy[asyncio]>=2.0"
uv add asyncpg                         # async PostgreSQL driver
uv add "pydantic-settings>=2.0"        # settings management
uv add "PyJWT>=2.8"                    # JWT — actively maintained, no extra deps
uv add "bcrypt>=4.0"                   # password hashing — use directly, not via passlib
# Alternative: uv add "fastapi-users[sqlalchemy]" — full auth stack (registration,
# verification, password reset, OAuth2) out of the box; skip PyJWT + bcrypt if using this
uv add httpx                           # async HTTP client
```

> **Why `bcrypt` directly and not `passlib`?** `passlib` had its last release
> in 2020 and is effectively unmaintained — it produces noisy warnings against
> modern `bcrypt` versions (`AttributeError: module 'bcrypt' has no attribute '__about__'`).
> Use `bcrypt` directly, or `argon2-cffi` if you prefer Argon2.

**Optional deps — add only what you need:**

```bash
uv add redis                           # caching or task queue broker (ARQ)
uv add arq                             # async task queue — see references/background-tasks.md
uv add structlog                       # structured JSON logging — see references/logging.md
uv add slowapi                         # rate limiting
```

**3. Add dev/test dependencies** (full setup in `references/testing.md` and `references/e2e-testing.md`):

```bash
uv add --dev pytest pytest-asyncio httpx anyio pytest-cov pytest-mock
uv add --dev asgi-lifespan                 # E2E — actually run lifespan in tests
uv add --dev ruff mypy                     # linting + type checking
```

**4. Recommended `pyproject.toml` tool config:**

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"          # no @pytest.mark.asyncio needed on every test
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

**5. Run the dev server:**

```bash
uv run fastapi dev app/main.py    # hot-reload, OpenAPI at /docs
```

**6. Add a dependency mid-project (never edit pyproject.toml by hand):**

```bash
uv add arq           # adds to pyproject.toml + updates uv.lock atomically
uv remove redis      # clean removal with lockfile update
uv sync              # re-sync venv after pulling changes from git
```

---

## Response approach

1. **Start async-first.** Every endpoint, database call, and external I/O must
   use `async def` and `await`. Never suggest sync alternatives unless the user
   explicitly asks or a library forces it (e.g. a sync-only third-party SDK).
2. **Design Pydantic models before endpoints.** Define request/response schemas
   with Pydantic V2 (`model_config`, `field_validator`, `model_validator`) before
   writing route handlers.
3. **Use dependency injection for shared resources.** DB sessions, current user,
   settings, and HTTP clients all belong in `Depends()`, not as module-level
   globals.
4. **Handle errors explicitly and consistently.** Never raise bare `HTTPException`
   with a plain string detail. All error responses must conform to the shared
   `ErrorResponse` schema. Define domain exceptions as typed Python classes and
   register a handler for each — never scatter `try/except HTTPException` blocks
   across route handlers. Override FastAPI's default 422 handler so validation
   errors return the same `ErrorResponse` shape as everything else.
5. **Write the test alongside the code.** Always provide a `pytest-asyncio` test
   for any non-trivial endpoint or utility shown — see `references/testing.md`.
   For critical user journeys (signup, login, payment, etc.) also add an E2E
   test — see `references/e2e-testing.md`.
6. **Call out performance risks inline.** Flag N+1 queries, missing `await`, or
   blocking calls (e.g. `time.sleep`, sync `requests`) with a short `# ⚠` comment.
7. **Never return unbounded collections.** Every list endpoint must paginate.
   Default to cursor-based pagination — see `references/pagination.md`.

---

## Async patterns

**Always prefer async SQLAlchemy over sync:**

```python
# ✅ async session factory
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/db", pool_size=10)
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        yield session
```

**Lifespan for startup/shutdown (FastAPI 0.136.1+):**

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # startup
    await engine.begin()
    yield
    # shutdown
    await engine.dispose()

app = FastAPI(lifespan=lifespan)
```

**External HTTP calls — always httpx, never requests.** Initialise one client
in lifespan and inject it via `Depends()` — never create a new client per request:

```python
# app/core/http.py
import httpx
from typing import AsyncGenerator
from fastapi import Request

async def get_http_client(request: Request) -> AsyncGenerator[httpx.AsyncClient, None]:
    # client is stored on app.state during lifespan — one client, reused per request
    yield request.app.state.http_client
```

Initialise in lifespan:

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.http_client = httpx.AsyncClient(timeout=10.0)
    yield
    await app.state.http_client.aclose()
```

Use in routes:

```python
@router.get("/external")
async def fetch_external(
    client: httpx.AsyncClient = Depends(get_http_client),
) -> dict:
    response = await client.get("https://api.example.com/data")
    return response.json()
```

---

## Settings management

Always use `pydantic-settings` for config. Never read `os.environ` directly
in routes or modules — inject settings via `Depends()`.

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
    database_url: str                        # postgresql+asyncpg://...
    db_pool_size: int = 10
    db_max_overflow: int = 20

    # auth
    secret_key: str                          # never hardcode — load from env
    access_token_expire_minutes: int = 15
    refresh_token_expire_days: int = 7

    # redis (optional — only set if using caching or task queues)
    redis_url: str | None = None

    # cors
    cors_origins: list[str] = []             # explicit allowlist, never ["*"] in prod

    # app
    environment: str = "development"         # development | staging | production
    debug: bool = False

@lru_cache
def get_settings() -> Settings:
    return Settings()
```

Inject via `Depends()` — never import the `Settings` instance as a global:

```python
# ✅ correct — settings injected, testable, overridable
@router.get("/health")
async def health(settings: Settings = Depends(get_settings)) -> dict:
    return {"env": settings.environment}

# ❌ wrong — module-level global, untestable, leaks across tests
settings = Settings()
```

Override in tests (full pattern in `references/testing.md`):

```python
app.dependency_overrides[get_settings] = lambda: Settings(
    database_url="postgresql+asyncpg://user:pass@localhost/test_db",
    secret_key="test-secret",
)
```

---

## CORS

Configure CORS explicitly with an allowlist driven from settings. Never use
wildcard origins (`["*"]`) in production — they are incompatible with
credentialed requests (browsers reject the combination).

```python
# app/main.py
from fastapi.middleware.cors import CORSMiddleware
from app.core.config import get_settings

settings = get_settings()

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.cors_origins,       # e.g. ["https://app.example.com"]
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"],
    allow_headers=["Authorization", "Content-Type", "X-Request-ID"],
    expose_headers=["X-Request-ID"],           # surface request ID to frontend
    max_age=600,                               # cache preflight for 10 min
)
```

In `.env`:

```
CORS_ORIGINS=["https://app.example.com","https://admin.example.com"]
```

Per-environment guidance:
- **development** — `["http://localhost:3000", "http://localhost:5173"]`
- **staging/production** — explicit production domains only, never wildcards

---

## Shared enums and types

Put cross-cutting types like `Role` in a single canonical module so every
layer imports from the same place:

```python
# app/core/enums.py
from enum import StrEnum

class Role(StrEnum):
    ADMIN = "admin"
    EDITOR = "editor"
    VIEWER = "viewer"
```

Import consistently — from models, routers, dependencies, and tests:

```python
from app.core.enums import Role
```

---

## Auth & security

> **Auth library choice:**
> - **`PyJWT`** — use when rolling your own JWT auth. Actively maintained, minimal deps.
> - **`fastapi-users`** — use when you want registration, email verification, password reset, and OAuth2 out of the box. Skip the manual JWT code below and follow its SQLAlchemy integration guide instead.
> - **`python-jose`** / **`fastapi-jwt-auth`** — avoid on new projects; unmaintained or superseded.

### Domain exception classes (defined first — referenced by everything else)

Define one exception class per error category. Never use raw `HTTPException`
inside business logic — it leaks HTTP concerns into your domain layer.

```python
# app/core/exceptions.py
class AppError(Exception):
    """Base for all domain errors."""
    status_code: int = 500
    code: str = "internal_error"
    message: str = "An unexpected error occurred."

class NotFoundError(AppError):
    status_code = 404
    code = "not_found"
    def __init__(self, resource: str, id: int | str):
        self.message = f"{resource} with id '{id}' not found."

class ConflictError(AppError):
    status_code = 409
    code = "conflict"
    def __init__(self, message: str):
        self.message = message

class PermissionDeniedError(AppError):
    status_code = 403
    code = "permission_denied"
    def __init__(self, message: str = "You do not have permission to perform this action."):
        self.message = message

class AuthenticationError(AppError):
    status_code = 401
    code = "unauthenticated"
    message = "Could not validate credentials."
```

### JWT access + refresh tokens (PyJWT)

```python
# app/core/security.py
from datetime import datetime, timedelta, timezone
import jwt
from jwt.exceptions import InvalidTokenError
from fastapi import Depends
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.config import Settings, get_settings
from app.core.database import get_db
from app.core.exceptions import AuthenticationError
from app.models import User

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/token")
ALGORITHM = "HS256"

def create_access_token(
    subject: str,
    settings: Settings,
    expires_delta: timedelta = timedelta(minutes=15),
) -> str:
    expire = datetime.now(timezone.utc) + expires_delta
    return jwt.encode({"sub": subject, "exp": expire}, settings.secret_key, algorithm=ALGORITHM)

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
    settings: Settings = Depends(get_settings),  # ✅ injected — no module-level global
) -> User:
    try:
        payload = jwt.decode(token, settings.secret_key, algorithms=[ALGORITHM])
        user_id: str = payload.get("sub")
        if user_id is None:
            raise AuthenticationError()
    except InvalidTokenError:
        raise AuthenticationError()
    user = await db.get(User, user_id)
    if user is None:
        raise AuthenticationError()
    return user
```

### Password hashing (bcrypt)

```python
# app/core/security.py (continued)
import bcrypt

def hash_password(password: str) -> str:
    return bcrypt.hashpw(password.encode("utf-8"), bcrypt.gensalt()).decode("utf-8")

def verify_password(password: str, hashed: str) -> bool:
    return bcrypt.checkpw(password.encode("utf-8"), hashed.encode("utf-8"))
```

### RBAC via dependency

```python
from app.core.enums import Role
from app.core.exceptions import PermissionDeniedError

def require_role(*roles: Role):
    async def checker(current_user: User = Depends(get_current_user)) -> User:
        if current_user.role not in roles:
            raise PermissionDeniedError()  # → 403 ErrorResponse via app_error_handler
        return current_user
    return checker

@router.delete("/{item_id}")
async def delete_item(
    item_id: int,
    _: User = Depends(require_role(Role.ADMIN)),
    db: AsyncSession = Depends(get_db),
):
    ...
```

---

## Response model discipline

Always declare `response_model=` explicitly. Never return ORM objects directly
— always project through a Pydantic schema.

```python
# app/schemas/users.py
from pydantic import BaseModel, ConfigDict, EmailStr

class UserOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)  # enables ORM → Pydantic conversion

    id: int
    email: EmailStr
    role: str

class UserCreate(BaseModel):
    email: EmailStr
    password: str

class UserPatch(BaseModel):
    email: EmailStr | None = None   # all fields optional for PATCH
    role: str | None = None
```

**GET — always declare `response_model`:**

```python
@router.get("/{user_id}", response_model=UserOut)
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)) -> UserOut:
    user = await db.get(User, user_id)
    if user is None:
        raise NotFoundError("User", user_id)
    return UserOut.model_validate(user)   # explicit ORM → schema conversion
```

**PATCH — use `model_dump(exclude_unset=True)`** so only provided fields are
written, preventing accidental overwrites:

```python
@router.patch("/{user_id}", response_model=UserOut)
async def patch_user(
    user_id: int,
    payload: UserPatch,
    db: AsyncSession = Depends(get_db),
) -> UserOut:
    user = await db.get(User, user_id)
    if user is None:
        raise NotFoundError("User", user_id)
    update_data = payload.model_dump(exclude_unset=True)  # only set fields
    for field, value in update_data.items():
        setattr(user, field, value)
    await db.commit()
    await db.refresh(user)
    return UserOut.model_validate(user)
```

**Annotate error responses in OpenAPI** so clients see the full contract:

```python
@router.get(
    "/{user_id}",
    response_model=UserOut,
    responses={
        404: {"model": ErrorResponse},
        403: {"model": ErrorResponse},
    },
)
async def get_user(...):
    ...
```

---

## Error handling & error responses

### Shared error response schema

Every error — validation, domain, or unexpected — must return this shape.
Never let FastAPI's default `{"detail": "..."}` string leak to clients.

```python
# app/schemas/errors.py
from pydantic import BaseModel

class ErrorDetail(BaseModel):
    field: str | None = None   # populated for validation errors
    message: str
    code: str                  # machine-readable slug, e.g. "not_found"

class ErrorResponse(BaseModel):
    errors: list[ErrorDetail]
```

### Exception handlers — register once on the app

```python
# app/core/exception_handlers.py
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError

from app.core.exceptions import AppError
from app.core.logging import logger
from app.schemas.errors import ErrorDetail, ErrorResponse

def register_exception_handlers(app: FastAPI) -> None:

    @app.exception_handler(AppError)
    async def app_error_handler(request: Request, exc: AppError) -> JSONResponse:
        return JSONResponse(
            status_code=exc.status_code,
            content=ErrorResponse(
                errors=[ErrorDetail(code=exc.code, message=exc.message)]
            ).model_dump(),
        )

    @app.exception_handler(RequestValidationError)
    async def validation_error_handler(request: Request, exc: RequestValidationError) -> JSONResponse:
        errors = [
            ErrorDetail(
                field=".".join(str(loc) for loc in err["loc"] if loc != "body"),
                message=err["msg"],
                code="validation_error",
            )
            for err in exc.errors()
        ]
        return JSONResponse(status_code=422, content=ErrorResponse(errors=errors).model_dump())

    @app.exception_handler(Exception)
    async def unhandled_error_handler(request: Request, exc: Exception) -> JSONResponse:
        # log the full traceback — structlog's exc_info=True captures the stack
        logger.exception(
            "unhandled_error",
            exc_info=exc,
            path=request.url.path,
            method=request.method,
        )
        return JSONResponse(
            status_code=500,
            content=ErrorResponse(
                errors=[ErrorDetail(code="internal_error", message="An unexpected error occurred.")]
            ).model_dump(),
        )
```

Wire them in `main.py`:

```python
# app/main.py
from app.core.exception_handlers import register_exception_handlers

app = FastAPI(lifespan=lifespan)
register_exception_handlers(app)
```

### Using domain exceptions in route handlers

Raise typed exceptions — never construct `HTTPException` manually in routes:

```python
# ✅ correct
@router.get("/{user_id}", response_model=UserOut)
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)) -> UserOut:
    user = await db.get(User, user_id)
    if user is None:
        raise NotFoundError("User", user_id)   # handler converts to 404 ErrorResponse
    return UserOut.model_validate(user)

# ❌ wrong — leaks HTTP concerns into route logic, inconsistent response shape
@router.get("/{user_id}")
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)):
    user = await db.get(User, user_id)
    if user is None:
        raise HTTPException(status_code=404, detail="User not found")
```

### What consistent error responses look like

```json
// 404 — NotFoundError
{
  "errors": [
    { "field": null, "code": "not_found", "message": "User with id '42' not found." }
  ]
}

// 422 — RequestValidationError (overridden handler)
{
  "errors": [
    { "field": "email", "code": "validation_error", "message": "value is not a valid email address" },
    { "field": "password", "code": "validation_error", "message": "String should have at least 8 characters" }
  ]
}

// 409 — ConflictError
{
  "errors": [
    { "field": null, "code": "conflict", "message": "A user with this email already exists." }
  ]
}
```

### Spot-checks for error handling

- No bare `raise HTTPException(...)` inside service or repository layers
- No `{"detail": "string"}` in API responses — all errors use `ErrorResponse`
- The 422 handler is overridden — FastAPI's default shape must not reach clients
- Every new domain exception class has a registered handler or inherits from `AppError`
- Unhandled exceptions are caught, logged with full traceback, and return a generic 500

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
        .options(selectinload(User.posts))  # ✅ not lazy-loaded on access
    )
    return result.scalar_one_or_none()
```

**Repository pattern:**

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

**Connection pool sizing:**

```python
engine = create_async_engine(
    settings.database_url,
    pool_size=settings.db_pool_size,
    max_overflow=settings.db_max_overflow,
    pool_timeout=30,
    pool_recycle=1800,     # recycle stale connections
)
```

---

## Redis caching (optional)

If using Redis for caching, initialise the pool once in lifespan and inject
via `Depends()` — same pattern as the HTTP client and ARQ pool:

```python
# app/core/cache.py
import redis.asyncio as aioredis
import json
from typing import AsyncGenerator
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

Use `slowapi` — the standard FastAPI rate limiting library. Apply limits at
the router level, not inside route handlers.

```python
# app/core/rate_limit.py
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)  # rate limit by IP by default
```

Wire into the app:

```python
# app/main.py
from slowapi import _rate_limit_exceeded_handler
from slowapi.errors import RateLimitExceeded
from app.core.rate_limit import limiter

app = FastAPI(lifespan=lifespan)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)
```

Apply to routes — use stricter limits on auth endpoints:

```python
@router.get("/items")
@limiter.limit("60/minute")
async def list_items(request: Request, ...):
    ...

@router.post("/auth/token")
@limiter.limit("5/minute")              # tighter limit on auth — prevent brute force
async def login(request: Request, ...):
    ...
```

---

## API versioning

Prefix routers with a version namespace. Default to URL versioning (`/v1/`)
— it's explicit, cacheable, and easy to route at the infrastructure level.

```python
# app/routers/v1/users.py
from fastapi import APIRouter
router = APIRouter(prefix="/users", tags=["users"])

# app/routers/v2/users.py
from fastapi import APIRouter
router = APIRouter(prefix="/users", tags=["users-v2"])
```

Register with version prefixes in `main.py`:

```python
from app.routers.v1 import users as users_v1
from app.routers.v2 import users as users_v2

app.include_router(users_v1.router, prefix="/v1")
app.include_router(users_v2.router, prefix="/v2")
# → GET /v1/users/{id}  and  GET /v2/users/{id}
```

**Versioning rules:**
- Introduce a new version only for breaking changes (field removal, type change, behaviour change)
- Additive changes (new optional fields, new endpoints) do not need a new version
- Maintain at least one prior version during a deprecation window — mark with `deprecated=True`

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
- No `lazy="lazy"` (default) on relationships that are accessed in the same request — use `selectin` or `joined`
- No unbounded list endpoints — see `references/pagination.md` for the cursor / offset patterns and the hard `MAX_PAGE_SIZE` cap

---

## Project layout (recommended)

```
app/
├── main.py              # FastAPI() instance, lifespan, middleware, router registration
├── core/
│   ├── config.py        # Pydantic Settings + get_settings()
│   ├── enums.py         # Role and other shared enums
│   ├── exceptions.py    # AppError subclasses
│   ├── security.py      # JWT helpers, password hashing, get_current_user
│   ├── database.py      # engine, AsyncSessionLocal, get_db
│   ├── http.py          # get_http_client (httpx pool)
│   ├── arq.py           # get_arq (ARQ pool, if used)
│   ├── cache.py         # get_redis(), get_cached() (optional)
│   ├── rate_limit.py    # slowapi Limiter instance
│   ├── logging.py       # structlog configuration
│   └── exception_handlers.py  # register_exception_handlers
├── middleware/
│   └── request_id.py    # RequestIDMiddleware
├── models/              # SQLAlchemy ORM models
├── schemas/             # Pydantic V2 request/response models + ErrorResponse
├── repositories/        # async data-access layer
├── routers/
│   ├── v1/              # versioned routers
│   └── v2/
├── tasks/               # background task modules (ARQ, if used)
│   ├── email.py
│   └── worker.py        # ARQ WorkerSettings
└── tests/
    ├── conftest.py
    ├── routers/          # endpoint tests (in-process, fast)
    └── e2e/              # end-to-end tests (real server + Postgres + Redis)
        ├── conftest.py   # LifespanManager / live_server fixtures
        └── test_*.py     # user journeys, ARQ workers, CORS preflight
```
