# End-to-end testing

Read this file when you need tests that exercise the **real** stack — a real
ASGI server process, real Postgres, real Redis, real lifespan startup and
shutdown, real ARQ workers. These complement (don't replace) the in-process
endpoint tests in `testing.md`.

## When to use E2E vs endpoint tests

| Concern | Endpoint tests (`testing.md`) | E2E tests (this file) |
|---|---|---|
| Single route logic, validation, error mapping | ✅ | overkill |
| Lifespan startup/shutdown actually running | ❌ skipped by default | ✅ |
| CORS preflight against real browser flow | ❌ | ✅ |
| Multi-request user journeys (login → act → logout) | possible but awkward | ✅ natural |
| ARQ worker actually processing the job | ❌ mocked | ✅ |
| Real connection pooling, timeouts, network behaviour | ❌ in-process | ✅ |
| Alembic migrations applied to a real DB | ❌ | ✅ |
| Speed | <1s per test | seconds per test |

**Rule of thumb:** every endpoint should have endpoint tests; the critical
user journeys (signup, login, checkout, etc.) should *additionally* have one
E2E test each. Don't try to E2E every error path — that's what endpoint
tests are for.

---

## Table of contents

- [Test stack with Docker Compose](#test-stack-with-docker-compose)
- [Pattern 1 — Real lifespan, in-process client](#pattern-1)
- [Pattern 2 — Real running server](#pattern-2)
- [Multi-step user-journey example](#multi-step-user-journey-example)
- [ARQ end-to-end](#arq-end-to-end)
- [CORS preflight](#cors-preflight)
- [Database migrations (Alembic)](#database-migrations-alembic)
- [CI workflow (GitHub Actions)](#ci-workflow-github-actions)
- [Speed and isolation rules](#speed-and-isolation-rules)

---

## Test stack with Docker Compose

E2E tests need real Postgres and Redis. Define a separate compose file so the
test stack runs alongside dev without colliding on ports.

```yaml
# docker-compose.test.yml
services:
  postgres-test:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
      POSTGRES_DB: test_db
    ports:
      - "5433:5432"        # 5433 to avoid colliding with dev postgres on 5432
    tmpfs:
      - /var/lib/postgresql/data    # in-memory — fast, no cleanup
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "test"]
      interval: 1s
      timeout: 3s
      retries: 30

  redis-test:
    image: redis:7-alpine
    ports:
      - "6380:6379"        # 6380 to avoid colliding with dev redis on 6379
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 1s
      timeout: 3s
      retries: 30
```

Bring it up before E2E tests:

```bash
docker compose -f docker-compose.test.yml up -d --wait
uv run pytest tests/e2e/
docker compose -f docker-compose.test.yml down -v
```

Test settings target the test stack ports:

```python
# tests/e2e/conftest.py
from app.core.config import Settings

E2E_SETTINGS = Settings(
    database_url="postgresql+asyncpg://test:test@localhost:5433/test_db",
    redis_url="redis://localhost:6380/0",
    secret_key="e2e-secret-key-do-not-use-in-production",
    environment="test",
    cors_origins=["http://localhost:3000"],
)
```

---

## Pattern 1 — Real lifespan, in-process client

Use this for E2E tests that need lifespan to run (real ARQ pool, real Redis
pool, real HTTP client on `app.state`) but don't need to test cross-process
networking. It's faster than spinning up a real server and catches most
lifespan-related bugs.

Install `asgi-lifespan`:

```bash
uv add --dev asgi-lifespan
```

```python
# tests/e2e/conftest.py
import pytest_asyncio
from asgi_lifespan import LifespanManager
from httpx import AsyncClient, ASGITransport

from app.main import app
from app.core.config import get_settings

@pytest_asyncio.fixture
async def e2e_app():
    # override settings before lifespan runs so the right URLs are used
    app.dependency_overrides[get_settings] = lambda: E2E_SETTINGS
    async with LifespanManager(app) as manager:
        # lifespan startup has now run — app.state.redis, app.state.arq, etc. are real
        yield manager.app
    app.dependency_overrides.pop(get_settings, None)

@pytest_asyncio.fixture
async def e2e_client(e2e_app):
    async with AsyncClient(transport=ASGITransport(app=e2e_app), base_url="http://test") as ac:
        yield ac
```

> **Why `LifespanManager`?** FastAPI's `TestClient` and `httpx.ASGITransport`
> on their own do **not** run lifespan handlers — they only invoke the ASGI
> `http` scope. `asgi-lifespan` wraps the app so startup/shutdown actually
> execute, which is what makes this a true integration test.

Use it like the regular client, but lifespan resources are real:

```python
async def test_redis_pool_is_initialised(e2e_app):
    # app.state.redis is a real aioredis pool, not a mock
    pong = await e2e_app.state.redis.ping()
    assert pong is True
```

---

## Pattern 2 — Real running server

Use this when you need to test cross-process behaviour (a real ARQ worker
picking up a job, a real browser-style HTTP call hitting a real socket).
Slower, but the most faithful.

Launch the server as a subprocess in a session-scoped fixture:

```python
# tests/e2e/conftest.py
import asyncio
import subprocess
import time
import pytest_asyncio
import httpx

@pytest_asyncio.fixture(scope="session")
async def live_server():
    # write E2E settings to a temp .env or pass via environment
    env = {
        "DATABASE_URL": E2E_SETTINGS.database_url,
        "REDIS_URL": E2E_SETTINGS.redis_url,
        "SECRET_KEY": E2E_SETTINGS.secret_key,
        "ENVIRONMENT": "test",
    }
    proc = subprocess.Popen(
        ["uv", "run", "fastapi", "run", "app/main.py", "--port", "8765"],
        env={**__import__("os").environ, **env},
    )
    # wait for the server to start accepting connections
    deadline = time.time() + 15
    while time.time() < deadline:
        try:
            httpx.get("http://localhost:8765/health", timeout=1)
            break
        except httpx.HTTPError:
            time.sleep(0.2)
    else:
        proc.terminate()
        raise RuntimeError("server did not start within 15s")
    yield "http://localhost:8765"
    proc.terminate()
    proc.wait(timeout=5)

@pytest_asyncio.fixture
async def live_client(live_server):
    async with httpx.AsyncClient(base_url=live_server) as ac:
        yield ac
```

Use it for real HTTP round-trips:

```python
async def test_real_health_endpoint(live_client):
    response = await live_client.get("/health")
    assert response.status_code == 200
```

---

## Multi-step user-journey example

This is the test that catches "the pieces work individually but break when
chained" bugs — exactly what unit tests can't see.

```python
# tests/e2e/test_user_journey.py
async def test_signup_login_create_list_delete_journey(e2e_client):
    # 1. sign up
    signup = await e2e_client.post(
        "/v1/users",
        json={"email": "journey@example.com", "password": "strongpass123"},
    )
    assert signup.status_code == 201
    user_id = signup.json()["id"]

    # 2. log in — real JWT issued by the app, not hand-crafted in a fixture
    login = await e2e_client.post(
        "/v1/auth/token",
        data={"username": "journey@example.com", "password": "strongpass123"},
        headers={"Content-Type": "application/x-www-form-urlencoded"},
    )
    assert login.status_code == 200
    token = login.json()["access_token"]
    auth_headers = {"Authorization": f"Bearer {token}"}

    # 3. create a resource using the token
    create = await e2e_client.post(
        "/v1/items",
        json={"name": "first item"},
        headers=auth_headers,
    )
    assert create.status_code == 201
    item_id = create.json()["id"]

    # 4. list and confirm the new item is there
    listing = await e2e_client.get("/v1/items", headers=auth_headers)
    assert listing.status_code == 200
    ids = [i["id"] for i in listing.json()["items"]]
    assert item_id in ids

    # 5. delete it
    delete = await e2e_client.delete(f"/v1/items/{item_id}", headers=auth_headers)
    assert delete.status_code == 204

    # 6. confirm it's gone
    gone = await e2e_client.get(f"/v1/items/{item_id}", headers=auth_headers)
    assert gone.status_code == 404
    assert gone.json()["errors"][0]["code"] == "not_found"
```

What this catches that endpoint tests don't:
- The token issued by `/auth/token` is actually accepted by other endpoints
  (same `secret_key`, same algorithm, same `sub` claim shape)
- The user created in step 1 is durable across requests (lifespan + real DB)
- The `ErrorResponse` shape is consistent across the whole journey, not just
  the specific endpoint being tested

---

## ARQ end-to-end

For ARQ, mocking `enqueue_job` (as in `testing.md`) only verifies that *your
code* dispatches. It does **not** verify that the worker would actually pick
up and process the job. For critical async paths, add one E2E test that runs
a real worker.

```python
# tests/e2e/test_arq_worker.py
import asyncio
from arq import create_pool
from arq.connections import RedisSettings
from arq.worker import Worker

from app.tasks.email import send_welcome_email

async def test_welcome_email_is_processed_by_worker():
    redis_settings = RedisSettings.from_dsn(E2E_SETTINGS.redis_url)
    pool = await create_pool(redis_settings)

    # enqueue a job
    job = await pool.enqueue_job("send_welcome_email", 42)

    # spin up a worker in-process for the duration of the test
    worker = Worker(
        functions=[send_welcome_email],
        redis_settings=redis_settings,
        burst=True,                  # exit when the queue is empty
        max_jobs=1,
        poll_delay=0.1,
    )
    await worker.main()
    await worker.close()

    # assert the job completed
    result = await job.result(timeout=5)
    assert result is None            # send_welcome_email returns None on success
    await pool.close()
```

`burst=True` makes the worker exit as soon as it has drained the queue,
which is exactly the behaviour you want in a test. For production the
worker is long-running; in tests, burst mode keeps things deterministic.

---

## CORS preflight

CORS is the canonical "looks right in unit tests, broken in browsers" bug.
E2E covers the real preflight flow:

```python
# tests/e2e/test_cors.py
async def test_cors_preflight_allows_configured_origin(e2e_client):
    response = await e2e_client.options(
        "/v1/items",
        headers={
            "Origin": "http://localhost:3000",
            "Access-Control-Request-Method": "POST",
            "Access-Control-Request-Headers": "Authorization, Content-Type",
        },
    )
    assert response.status_code == 200
    assert response.headers["access-control-allow-origin"] == "http://localhost:3000"
    assert "POST" in response.headers["access-control-allow-methods"]
    assert response.headers["access-control-allow-credentials"] == "true"

async def test_cors_preflight_rejects_unknown_origin(e2e_client):
    response = await e2e_client.options(
        "/v1/items",
        headers={
            "Origin": "https://evil.example.com",
            "Access-Control-Request-Method": "POST",
        },
    )
    # CORSMiddleware does not set allow-origin when the origin isn't on the allowlist
    assert "access-control-allow-origin" not in response.headers
```

---

## Database migrations (Alembic)

If the project uses Alembic, the E2E setup is also where you verify migrations
actually apply cleanly to an empty DB — not just that the ORM `create_all`
schema works.

```python
# tests/e2e/conftest.py
from alembic import command
from alembic.config import Config

@pytest_asyncio.fixture(scope="session", autouse=True)
async def apply_migrations():
    cfg = Config("alembic.ini")
    cfg.set_main_option("sqlalchemy.url", E2E_SETTINGS.database_url.replace("+asyncpg", ""))
    command.upgrade(cfg, "head")
    yield
    command.downgrade(cfg, "base")    # optional — tmpfs Postgres throws this away anyway
```

This catches:
- Migrations that work locally but fail on a fresh DB
- Migrations that depend on data that isn't in a fresh DB
- Drift between the ORM models and the migration history

---

## CI workflow (GitHub Actions)

Run the same E2E suite in CI using a `services` block — no Docker Compose
needed because Actions can launch service containers directly.

```yaml
# .github/workflows/test.yml
name: tests
on: [push, pull_request]

jobs:
  e2e:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test_db
        ports:
          - 5433:5432
        options: >-
          --health-cmd="pg_isready -U test"
          --health-interval=2s
          --health-timeout=3s
          --health-retries=15
      redis:
        image: redis:7-alpine
        ports:
          - 6380:6379
        options: >-
          --health-cmd="redis-cli ping"
          --health-interval=2s
          --health-timeout=3s
          --health-retries=15

    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
        with:
          enable-cache: true
      - run: uv python pin 3.12
      - run: uv sync --frozen
      - name: Run endpoint tests
        run: uv run pytest tests/ --ignore=tests/e2e -x
      - name: Run E2E tests
        env:
          DATABASE_URL: postgresql+asyncpg://test:test@localhost:5433/test_db
          REDIS_URL: redis://localhost:6380/0
          SECRET_KEY: ci-secret-key
        run: uv run pytest tests/e2e/ -x
```

Splitting the runs so endpoint tests fail fast (most likely to fail) before
the slower E2E suite starts.

---

## Speed and isolation rules

- **Don't run E2E in parallel** with itself unless you've sharded the DB and
  Redis namespaces — they share state by definition. `pytest-xdist` is fine
  for endpoint tests, dangerous for E2E.
- **Use `tmpfs` for the test Postgres data dir.** Disk I/O is the bottleneck
  on E2E suites; running Postgres entirely in RAM cuts test time by 5-10x.
- **Truncate, don't recreate.** Between E2E tests, `TRUNCATE ... CASCADE` is
  far faster than dropping and recreating tables. A session-scoped engine
  fixture + a function-scoped truncate fixture is the sweet spot.
- **Keep E2E surface narrow.** One E2E test per critical user journey, not
  one per endpoint. The math: 100 endpoints × 2s each = 3min E2E suite, vs.
  100 endpoints × 50ms each = 5s endpoint suite. Use the right tool.
- **Real network = real flakes.** Treat slow/unreliable E2E tests as bugs
  in the test, not "expected flakiness." Retries hide real concurrency bugs.

```python
# truncate-between-tests pattern
@pytest_asyncio.fixture(autouse=True)
async def truncate_tables(e2e_app):
    yield
    async with e2e_app.state.engine.begin() as conn:
        await conn.execute(text("TRUNCATE users, items, posts RESTART IDENTITY CASCADE"))
```
