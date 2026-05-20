# Rate limiting

Read this when adding rate limits to an endpoint — login throttling,
per-user quotas, or general DoS protection. Uses `slowapi`, applied at the
router decorator level (not inside the handler body).

## Install

```bash
uv add slowapi
```

## Limiter instance

```python
# app/core/rate_limit.py
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)
```

`get_remote_address` keys off the client IP. For authenticated routes, key
off the user ID instead (see "Per-user limits" below).

## Wire it on the app

```python
# app/main.py
from slowapi import _rate_limit_exceeded_handler
from slowapi.errors import RateLimitExceeded
from app.core.rate_limit import limiter

app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)
```

> `_rate_limit_exceeded_handler` returns slowapi's own 429 shape, **not**
> `ErrorResponse`. If you want unified error shapes (recommended — see
> `error-handling.md`), write your own handler that returns
> `ErrorResponse(errors=[ErrorDetail(code="rate_limit_exceeded", ...)])`
> with a `Retry-After` header.

## Apply per route

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

`request: Request` must be in the handler signature — slowapi reads the
client address off it.

## Per-user limits (authenticated routes)

```python
from fastapi import Request

def user_id_key(request: Request) -> str:
    # the auth dep sets request.state.user_id; falls back to IP for unauth requests
    return getattr(request.state, "user_id", get_remote_address(request))

user_limiter = Limiter(key_func=user_id_key)

@router.post("/reports")
@user_limiter.limit("10/hour")
async def create_report(request: Request, current_user = Depends(get_current_user), ...):
    ...
```

## Sensible defaults

| Endpoint type | Limit |
|---|---|
| Login / password reset / signup | `5/minute` per IP |
| Authenticated write endpoints | `60/minute` per user |
| Authenticated read endpoints | `300/minute` per user |
| Public unauth endpoints | `30/minute` per IP |

These are starting points — tune from production traffic, not guesses.

## Distributed deployments

The default in-memory backend resets on each process restart and isn't
shared across instances. For a multi-replica deployment, use Redis:

```python
limiter = Limiter(
    key_func=get_remote_address,
    storage_uri="redis://localhost:6379/1",     # distinct DB from cache
)
```

## Spot-checks

- Auth endpoints (`/auth/token`, `/auth/refresh`, password reset) have a tight
  limit — `5/minute` per IP or lower
- Limits are on the router/decorator, not inside the handler body
- Authenticated routes key off user ID, not IP (otherwise NAT'd users share quota)
- Multi-replica deployments use the Redis backend, not in-memory
- The 429 response shape matches `ErrorResponse` (custom handler) — clients
  shouldn't see two error shapes
