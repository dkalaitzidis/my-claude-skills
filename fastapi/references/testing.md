# Testing FastAPI applications

Read this file when writing pytest-asyncio tests, setting up fixtures, or
debugging test isolation issues. The patterns here assume the project layout
and settings/dependency-injection conventions from the main `SKILL.md`.

## Table of contents

- [Coverage target](#coverage-target)
- [Fixtures — async client, test DB, auth tokens](#fixtures)
- [Happy path tests](#happy-path-tests)
- [Error path tests](#error-path-tests)
- [422 override verification](#422-override-verification)
- [Auth path tests](#auth-path-tests)
- [Background task tests](#background-task-tests)
- [Pagination tests](#pagination-tests)
- [Settings override in individual tests](#settings-override-in-individual-tests)
- [Test isolation rules](#test-isolation-rules)

---

## Coverage target

Aim for 100% coverage. Add `pytest-cov` to track it on every run — the report
surfaces uncovered lines so gaps are visible. Enforcement is left to the team's
discretion.

```bash
uv add --dev pytest-cov
```

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
addopts = "--cov=app --cov-report=term-missing"

[tool.coverage.run]
omit = [
    "app/main.py",         # wiring only — tested implicitly via client fixture
    "app/core/logging.py", # config only
]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "if TYPE_CHECKING:",
    "raise NotImplementedError",
    "if __name__ == .__main__.:",
]
```

Run:

```bash
uv run pytest                        # shows coverage report after tests
uv run pytest --cov-report=html      # open htmlcov/index.html for line-level view
```

---

## Fixtures

The full `conftest.py` — async client, test DB session, auth token helpers,
and a settings override that applies to every test:

```python
# tests/conftest.py
import pytest
import pytest_asyncio
import jwt
from datetime import datetime, timedelta, timezone
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker

from app.main import app
from app.core.database import get_db
from app.core.config import get_settings, Settings
from app.core.enums import Role
from app.models import Base, User

TEST_SETTINGS = Settings(
    database_url="postgresql+asyncpg://user:pass@localhost/test_db",
    secret_key="test-secret-key-do-not-use-in-production",
    environment="test",
)

# --- settings override (applied to all tests) ---
@pytest.fixture(autouse=True)
def override_settings():
    app.dependency_overrides[get_settings] = lambda: TEST_SETTINGS
    yield
    app.dependency_overrides.pop(get_settings, None)

# --- database ---
@pytest_asyncio.fixture(scope="session")
async def engine():
    eng = create_async_engine(TEST_SETTINGS.database_url)
    async with eng.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield eng
    async with eng.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await eng.dispose()

@pytest_asyncio.fixture
async def db(engine):
    async with async_sessionmaker(engine)() as session:
        yield session
        await session.rollback()   # isolate each test — no state leaks between tests

# --- http client ---
@pytest_asyncio.fixture
async def client(db):
    app.dependency_overrides[get_db] = lambda: db
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as ac:
        yield ac
    app.dependency_overrides.pop(get_db, None)

# --- auth token helpers ---
def _make_token(user_id: int, role: str) -> str:
    return jwt.encode(
        {"sub": str(user_id), "role": role, "exp": datetime.now(timezone.utc) + timedelta(minutes=15)},
        TEST_SETTINGS.secret_key,
        algorithm="HS256",
    )

@pytest.fixture
def admin_token() -> str:
    return _make_token(user_id=1, role=Role.ADMIN)

@pytest.fixture
def viewer_token() -> str:
    return _make_token(user_id=2, role=Role.VIEWER)

# --- seeded user fixture ---
@pytest_asyncio.fixture
async def existing_user(db) -> User:
    # let the DB assign the id to avoid sequence collisions on Postgres
    user = User(email="existing@example.com", role=Role.VIEWER, hashed_password="hashed")
    db.add(user)
    await db.commit()
    await db.refresh(user)
    return user
```

---

## Happy path tests

```python
# tests/routers/test_users.py
async def test_create_user_returns_201(client, mocker):
    mocker.patch("app.tasks.email.send_welcome_email")
    response = await client.post("/users", json={"email": "new@example.com", "password": "secret123"})
    assert response.status_code == 201
    body = response.json()
    assert body["email"] == "new@example.com"
    assert "password" not in body              # never leak password in response
    assert "hashed_password" not in body       # never leak hashed password either

async def test_get_user_returns_200(client, existing_user):
    response = await client.get(f"/users/{existing_user.id}")
    assert response.status_code == 200
    assert response.json()["email"] == existing_user.email
```

---

## Error path tests

Validate the `ErrorResponse` shape on every failure mode:

```python
async def test_get_user_not_found(client):
    response = await client.get("/users/99999")
    assert response.status_code == 404
    body = response.json()
    assert body == {
        "errors": [{"field": None, "code": "not_found", "message": "User with id '99999' not found."}]
    }

async def test_create_user_duplicate_email_returns_409(client, existing_user):
    response = await client.post("/users", json={"email": existing_user.email, "password": "x"})
    assert response.status_code == 409
    assert response.json()["errors"][0]["code"] == "conflict"

async def test_delete_item_forbidden_for_viewer(client, viewer_token):
    response = await client.delete("/items/1", headers={"Authorization": f"Bearer {viewer_token}"})
    assert response.status_code == 403
    assert response.json()["errors"][0]["code"] == "permission_denied"

async def test_delete_item_succeeds_for_admin(client, admin_token):
    response = await client.delete("/items/1", headers={"Authorization": f"Bearer {admin_token}"})
    assert response.status_code == 204
```

---

## 422 override verification

Confirm FastAPI's default shape is replaced by your `ErrorResponse`:

```python
async def test_invalid_payload_returns_error_response_shape(client):
    response = await client.post("/users", json={"email": "not-an-email", "password": ""})
    assert response.status_code == 422
    body = response.json()
    # must be ErrorResponse, not FastAPI's default {"detail": [...]}
    assert "errors" in body
    assert "detail" not in body
    assert body["errors"][0]["code"] == "validation_error"
    assert body["errors"][0]["field"] == "email"

async def test_missing_required_field_returns_validation_error(client):
    response = await client.post("/users", json={})
    assert response.status_code == 422
    codes = [e["code"] for e in response.json()["errors"]]
    assert all(c == "validation_error" for c in codes)
```

---

## Auth path tests

```python
async def test_missing_token_returns_401(client):
    response = await client.get("/users/me")
    assert response.status_code == 401
    assert response.json()["errors"][0]["code"] == "unauthenticated"

async def test_invalid_token_returns_401(client):
    response = await client.get("/users/me", headers={"Authorization": "Bearer bad.token.here"})
    assert response.status_code == 401
    assert response.json()["errors"][0]["code"] == "unauthenticated"
```

---

## Background task tests

For `BackgroundTasks` — patch the function and assert it was called:

```python
async def test_background_task_called_on_create(client, mocker):
    mock_fn = mocker.patch("app.tasks.email.send_welcome_email")
    await client.post("/users", json={"email": "bg@example.com", "password": "pass"})
    mock_fn.assert_called_once()
```

For ARQ — override the pool dependency with an `AsyncMock` and assert
`enqueue_job` was called:

```python
from app.core.arq import get_arq

async def test_arq_job_enqueued_on_create(client, mocker):
    mock_arq = mocker.AsyncMock()
    app.dependency_overrides[get_arq] = lambda: mock_arq
    await client.post("/users", json={"email": "arq@example.com", "password": "pass"})
    mock_arq.enqueue_job.assert_called_once_with("send_welcome_email", mocker.ANY)
    app.dependency_overrides.pop(get_arq, None)
```

---

## Pagination tests

```python
async def test_list_items_respects_limit(client):
    response = await client.get("/items?limit=5")
    assert response.status_code == 200
    assert len(response.json()["items"]) <= 5

async def test_list_items_rejects_overlimit(client):
    response = await client.get("/items?limit=999")
    assert response.status_code == 422          # MAX_PAGE_SIZE enforced by Query(le=100)

async def test_list_items_cursor_advances(client):
    first = await client.get("/items?limit=2")
    cursor = first.json()["next_cursor"]
    second = await client.get(f"/items?after_id={cursor}&limit=2")
    first_ids = {i["id"] for i in first.json()["items"]}
    second_ids = {i["id"] for i in second.json()["items"]}
    assert first_ids.isdisjoint(second_ids)     # no overlap between pages
```

---

## Settings override in individual tests

Override one setting for a single test without affecting the others:

```python
async def test_endpoint_uses_injected_settings(client):
    custom = TEST_SETTINGS.model_copy(update={"environment": "staging"})
    app.dependency_overrides[get_settings] = lambda: custom
    response = await client.get("/health")
    assert response.json()["env"] == "staging"
    app.dependency_overrides.pop(get_settings, None)  # restore
```

---

## Test isolation rules

- **Never share mutable state between tests.** The `db` fixture rolls back after
  every test — don't commit data in a test and expect it in the next.
- **Seed data in fixtures, not in test bodies.** Use `existing_user`-style
  fixtures so setup is reusable and failures are attributable.
- **Never use production `.env`** — `override_settings` (autouse) replaces
  `get_settings` for every test, so real secrets never load.
- **Mock at the boundary.** Mock the enqueue call or the task function, not
  internal helpers — test that the dispatch happens, not how it's implemented.
- **One assertion per behaviour, not per line.** Group assertions about the
  same response together; split different behaviours into separate tests.
