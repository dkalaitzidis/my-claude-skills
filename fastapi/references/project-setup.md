# Project setup (greenfield)

Read this when initialising a new FastAPI project or auditing an existing
project's tooling. Use **uv** throughout — never `pip`, `pip-tools`, or
`poetry`. It's 10–100x faster, manages the venv automatically, and produces
a lockfile that guarantees reproducible installs across dev, CI, and prod.

## Init

```bash
uv init my-api
cd my-api
uv python pin 3.12
```

## Runtime dependencies

```bash
uv add "fastapi[standard]>=0.136.1"   # includes uvicorn + fastapi-cli
uv add "sqlalchemy[asyncio]>=2.0" asyncpg
uv add "pydantic-settings>=2.0"
uv add "PyJWT>=2.8" "bcrypt>=4.0"     # see auth.md
uv add httpx                          # async HTTP client
```

## Optional — add only what you need

```bash
uv add redis arq                      # caching + async task queue
uv add structlog                      # JSON logging
uv add slowapi                        # rate limiting
```

## Dev/test dependencies

Full setup in `testing.md` and `e2e-testing.md`:

```bash
uv add --dev pytest pytest-asyncio httpx anyio pytest-cov pytest-mock
uv add --dev asgi-lifespan            # E2E — actually run lifespan in tests
uv add --dev ruff mypy
```

## `pyproject.toml` tool config

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

## Run

```bash
uv run fastapi dev app/main.py        # hot-reload, OpenAPI at /docs
uv add <pkg>                          # never edit pyproject.toml by hand
uv remove <pkg>
uv sync                               # re-sync venv after pulling
```

## Auth library choice

Use `PyJWT` + `bcrypt` for rolling your own. Use `fastapi-users` if you want
registration, email verification, and password reset out of the box. Avoid
`python-jose` and `passlib` — both unmaintained. Full details in `auth.md`.
