# Structured logging

Read this file when setting up logging for a FastAPI app. The pattern uses
`structlog` for machine-readable JSON logs paired with a request ID middleware
so every log line for a given request is traceable across services.

## Install

```bash
uv add structlog
```

## Configure structlog

```python
# app/core/logging.py
import sys
import structlog
import logging

def configure_logging() -> None:
    structlog.configure(
        processors=[
            structlog.contextvars.merge_contextvars,
            structlog.processors.add_log_level,
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.processors.JSONRenderer(),   # machine-readable in production
        ],
        wrapper_class=structlog.make_filtering_bound_logger(logging.INFO),
        logger_factory=structlog.WriteLoggerFactory(file=sys.stderr),
    )

logger = structlog.get_logger()
```

For local development, swap `JSONRenderer()` for `ConsoleRenderer()` to get
coloured, human-readable output. Drive the choice from settings:

```python
def configure_logging() -> None:
    settings = get_settings()
    renderer = (
        structlog.processors.JSONRenderer()
        if settings.environment != "development"
        else structlog.dev.ConsoleRenderer()
    )
    structlog.configure(
        processors=[
            structlog.contextvars.merge_contextvars,
            structlog.processors.add_log_level,
            structlog.processors.TimeStamper(fmt="iso"),
            renderer,
        ],
        wrapper_class=structlog.make_filtering_bound_logger(logging.INFO),
        logger_factory=structlog.WriteLoggerFactory(file=sys.stderr),
    )
```

---

## Request ID middleware

Attach a unique ID to every request so logs across services can be correlated.
If the client sends an `X-Request-ID` header (e.g. from an upstream gateway),
reuse it; otherwise generate a new UUID.

```python
# app/middleware/request_id.py
import uuid
from starlette.middleware.base import BaseHTTPMiddleware
from structlog.contextvars import bind_contextvars, clear_contextvars

class RequestIDMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        clear_contextvars()
        request_id = request.headers.get("X-Request-ID", str(uuid.uuid4()))
        bind_contextvars(request_id=request_id, path=request.url.path, method=request.method)
        response = await call_next(request)
        response.headers["X-Request-ID"] = request_id
        return response
```

`bind_contextvars` attaches `request_id`, `path`, and `method` to every
log call made during the request — no need to thread them through manually.

---

## Wire it up

```python
# app/main.py
from app.core.logging import configure_logging
from app.middleware.request_id import RequestIDMiddleware

configure_logging()
app = FastAPI(lifespan=lifespan)
app.add_middleware(RequestIDMiddleware)
```

If you're also using `CORSMiddleware`, expose `X-Request-ID` to the frontend
so it can log the same ID on its side:

```python
app.add_middleware(
    CORSMiddleware,
    ...,
    expose_headers=["X-Request-ID"],
)
```

---

## Logging from route handlers and repositories

Use bound context — no need to pass loggers around or repeat the request ID:

```python
# app/repositories/users.py
from app.core.logging import logger

async def create(self, email: str, hashed_password: str) -> User:
    user = User(email=email, hashed_password=hashed_password)
    self.db.add(user)
    await self.db.commit()
    logger.info("user.created", user_id=user.id, email=email)
    return user
```

Output (with `JSONRenderer`):

```json
{
  "event": "user.created",
  "user_id": 42,
  "email": "new@example.com",
  "request_id": "8f3a-...",
  "path": "/users",
  "method": "POST",
  "level": "info",
  "timestamp": "2026-05-20T09:00:00Z"
}
```

---

## Logging unhandled exceptions

The `unhandled_error_handler` in `app/core/exception_handlers.py` should use
`logger.exception` (or pass `exc_info=exc`) to capture the full traceback:

```python
@app.exception_handler(Exception)
async def unhandled_error_handler(request: Request, exc: Exception) -> JSONResponse:
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

`structlog` includes the traceback in the `exception` field of the JSON
output, which integrates cleanly with log aggregators (Datadog, Loki, etc.).

---

## What to log, what not to log

**Log:**
- Domain events (`user.created`, `payment.processed`, `order.shipped`)
- Auth events (`login.success`, `login.failed`, `token.refreshed`)
- External calls and their latency
- Unhandled exceptions with full traceback
- Slow queries or operations exceeding a threshold

**Never log:**
- Passwords (plain or hashed), tokens, API keys, session IDs
- Full payment card numbers, full SSNs, or other regulated PII
- Entire request/response bodies for endpoints handling sensitive data

If you need to log a sensitive field for debugging, redact it explicitly
(`email="r***@example.com"`) — never rely on log scrubbing at the aggregator.
