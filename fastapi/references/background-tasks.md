# Background & async task offloading

Read this file when offloading work from request handlers — sending emails,
generating reports, calling slow external services, or running periodic jobs.
The patterns here build on the lifespan + dependency injection conventions
from the main `SKILL.md`.

## Decision guide

Always pick the simplest tool that satisfies the requirements. Don't reach
for a distributed task queue unless the use case justifies it.

| Need | Recommended tool |
|---|---|
| Fire-and-forget after a response, no retries needed | `BackgroundTasks` (built-in) |
| Retries, result storage, async-native workers, periodic schedules | ARQ |

For greenfield async FastAPI projects, ARQ is the recommended task queue —
it's fully async, integrates cleanly with the rest of the stack, and avoids
Celery's sync-worker constraints. Only reach for Celery if you have existing
Celery infrastructure to integrate with.

---

## Option 1 — BackgroundTasks (default starting point)

Zero dependencies, runs in the same process. Good for sending emails,
logging, or webhooks where occasional loss on worker crash is acceptable.

```python
from fastapi import BackgroundTasks

async def send_welcome_email(user_id: int) -> None:
    # runs after the response is sent, in the same process
    ...

@router.post("/users", status_code=201)
async def create_user(
    payload: UserCreate,
    background_tasks: BackgroundTasks,
    db: AsyncSession = Depends(get_db),
) -> UserOut:
    user = await UserRepository(db).create(payload.email, hash_password(payload.password))
    background_tasks.add_task(send_welcome_email, user.id)  # fire-and-forget
    return UserOut.model_validate(user)
```

**Limitations:** no retries, no persistence, no cross-process distribution.
If the worker process crashes mid-task, the work is lost. Upgrade to ARQ
when that's unacceptable.

---

## Option 2 — ARQ (async-native, Redis-backed)

Fully async, Redis-backed, lightweight. Best fit when you need retries,
result storage, and periodic schedules without Celery's operational overhead.

### Install

```bash
uv add arq redis
```

### Task definition

Tasks are async functions that accept a `ctx` dict as their first argument:

```python
# app/tasks/email.py
async def send_welcome_email(ctx: dict, user_id: int) -> None:
    # async context — use async DB session here
    ...
```

### Pool lifecycle — initialise once in lifespan, inject via Depends

Same pattern as the HTTP client and Redis cache: one pool stored on
`app.state`, reused across requests, injected via a dependency.

```python
# app/core/arq.py
from typing import AsyncGenerator
from arq import ArqRedis
from fastapi import Request

async def get_arq(request: Request) -> AsyncGenerator[ArqRedis, None]:
    # pool is stored on app.state during lifespan — one pool, reused per request
    yield request.app.state.arq
```

```python
# app/main.py
from contextlib import asynccontextmanager
from arq import create_pool
from arq.connections import RedisSettings

@asynccontextmanager
async def lifespan(app: FastAPI):
    settings = get_settings()        # ⚠ direct call — see note below
    app.state.arq = await create_pool(RedisSettings.from_dsn(settings.redis_url))
    yield
    await app.state.arq.close()
```

> **Why this isn't `Depends(get_settings)`.** Lifespan runs *before* any
> request, so dependency resolution isn't available — `get_settings()` is
> called directly. The implication for tests: `app.dependency_overrides`
> does **not** affect lifespan-loaded resources. To redirect ARQ/Redis/DB
> at the test stack, set env vars before app import and `cache_clear()` the
> `@lru_cache`'d `get_settings` — see the `e2e_env` fixture in
> `e2e-testing.md`.

### Dispatch from an endpoint

```python
from arq import ArqRedis
from app.core.arq import get_arq

@router.post("/users", status_code=201)
async def create_user(
    payload: UserCreate,
    db: AsyncSession = Depends(get_db),
    arq: ArqRedis = Depends(get_arq),
) -> UserOut:
    user = await UserRepository(db).create(payload.email, hash_password(payload.password))
    await arq.enqueue_job("send_welcome_email", user.id)
    return UserOut.model_validate(user)
```

### Worker config (separate process)

```python
# app/tasks/worker.py
from app.tasks.email import send_welcome_email
from app.core.config import get_settings
from arq.connections import RedisSettings

class WorkerSettings:
    functions = [send_welcome_email]
    redis_settings = RedisSettings.from_dsn(get_settings().redis_url)
    max_tries = 3                # retry on failure
    job_timeout = 300            # 5 minutes per task
```

Run with:

```bash
uv run arq app.tasks.worker.WorkerSettings
```

### Periodic schedules

ARQ supports cron-style schedules via the `cron_jobs` attribute on
`WorkerSettings`:

```python
from arq import cron

async def daily_report(ctx: dict) -> None:
    ...

class WorkerSettings:
    functions = [send_welcome_email]
    cron_jobs = [
        cron(daily_report, hour=6, minute=0),   # every day at 06:00 UTC
    ]
    redis_settings = RedisSettings.from_dsn(get_settings().redis_url)
```

---

## Testing background tasks

Two layers:

- **Endpoint tests** (`testing.md`) — mock the dispatch. For `BackgroundTasks`,
  patch the task function and assert it was called. For ARQ, override `get_arq`
  with an `AsyncMock` and assert `enqueue_job` was called. Fast, deterministic,
  covers your code's dispatch logic.
- **E2E tests** (`e2e-testing.md`) — run a real ARQ worker in burst mode against
  a real Redis, enqueue a job, and assert the side effect. Slower, but the only
  way to verify the worker would actually pick up and process your jobs in
  production. Add one of these per critical async path.
