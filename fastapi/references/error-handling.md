# Error handling & error responses

Read this file when designing or reviewing how the API surfaces errors —
defining the response shape, registering exception handlers, overriding
FastAPI's default 422 format, or auditing route handlers for `HTTPException`
leakage.

> **Coupled with auth.** The domain exception classes (`AppError`,
> `NotFoundError`, `AuthenticationError`, etc.) are defined in `auth.md`
> because they're referenced by `get_current_user` and `require_role`. The
> handlers here turn those exceptions into HTTP responses.

## The contract — one shape for every error

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

Why `errors: list[ErrorDetail]` and not a single object? Because validation
failures naturally produce multiple errors (one per invalid field), and you
want the same shape for one-error and many-error responses.

---

## Exception handlers — register once on the app

Three handlers cover everything: domain (`AppError`), validation
(`RequestValidationError`), and unexpected (`Exception`). They all produce
`ErrorResponse` JSON.

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
        # log the full traceback — structlog's exc_info=exc captures the stack
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

---

## Using domain exceptions in route handlers

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

The same rule applies in service and repository layers — they should raise
domain exceptions, not `HTTPException`. The exception class encodes the HTTP
status; the handler translates it.

---

## What consistent error responses look like

```json
// 404 — NotFoundError
{
  "errors": [
    { "field": null, "code": "not_found", "message": "User with id '42' not found." }
  ]
}

// 422 — RequestValidationError (overridden handler — multiple fields, multiple errors)
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

// 401 — AuthenticationError
{
  "errors": [
    { "field": null, "code": "unauthenticated", "message": "Could not validate credentials." }
  ]
}

// 403 — PermissionDeniedError
{
  "errors": [
    { "field": null, "code": "permission_denied", "message": "You do not have permission to perform this action." }
  ]
}

// 500 — unhandled exception
{
  "errors": [
    { "field": null, "code": "internal_error", "message": "An unexpected error occurred." }
  ]
}
```

The `code` is the contract for clients — they switch on it, not on the
human-readable `message`. Treat `code` values as semver: never rename a
code without a versioned migration; add new codes freely.

---

## Annotating error responses in OpenAPI

So clients see the full contract in the generated schema:

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

For a project-wide default, set this on the `FastAPI` instance and let
individual routes add specifics:

```python
app = FastAPI(
    lifespan=lifespan,
    responses={
        422: {"model": ErrorResponse},
        500: {"model": ErrorResponse},
    },
)
```

---

## Adding a new domain exception

Three steps:

1. **Define it in `app/core/exceptions.py`** — inherit from `AppError`, set
   `status_code`, `code`, and `message`.
2. **No handler change needed** — the existing `AppError` handler catches it.
3. **Add it to the OpenAPI `responses=` of any route that raises it** — so
   clients see it in the schema.

Example:

```python
# app/core/exceptions.py
class RateLimitExceededError(AppError):
    status_code = 429
    code = "rate_limit_exceeded"
    def __init__(self, retry_after_seconds: int):
        self.message = f"Rate limit exceeded. Retry after {retry_after_seconds}s."
```

Note: `slowapi`'s built-in `RateLimitExceeded` is a separate concern — that
one has its own handler registered separately (`_rate_limit_exceeded_handler`,
see `rate-limiting.md`). The `RateLimitExceededError`
above is for cases where *your code* enforces a custom rate limit (e.g. a
per-user quota beyond what slowapi tracks).

---

## Spot-checks for error handling

- No bare `raise HTTPException(...)` inside service, repository, or domain layers
- No `{"detail": "string"}` in API responses — all errors use `ErrorResponse`
- The 422 handler is overridden — FastAPI's default shape must not reach clients
- Every new domain exception class inherits from `AppError` (no separate handler needed)
- Unhandled exceptions are caught, logged with full traceback via `logger.exception`, and return a generic 500
- Error `code` slugs are stable across versions — clients depend on them
- Sensitive data (tokens, passwords, internal paths) never appears in error messages

---

## Testing error handling

See `testing.md` for the full patterns. Every error path deserves an
`ErrorResponse`-shape assertion, not just a status code check:

```python
async def test_get_user_not_found(client):
    response = await client.get("/users/99999")
    assert response.status_code == 404
    body = response.json()
    assert body == {
        "errors": [{"field": None, "code": "not_found", "message": "User with id '99999' not found."}]
    }
```

Confirm the 422 override works — `"errors"` in body, `"detail"` not in body.
That single test catches the entire family of "FastAPI's default leaked
through" regressions.
