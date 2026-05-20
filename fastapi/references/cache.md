# Redis caching

Read this when adding Redis to a FastAPI app — caching read-heavy endpoints,
storing short-lived state (rate-limit counters, idempotency keys, session
data), or fronting an expensive external call. Uses the same
lifespan-managed-resource pattern as `httpx` and `arq` (see `SKILL.md`).

## Install

```bash
uv add redis
```

## Lifespan + dependency

```python
# app/main.py
import redis.asyncio as aioredis

@asynccontextmanager
async def lifespan(app: FastAPI):
    settings = get_settings()
    app.state.redis = aioredis.from_url(settings.redis_url, decode_responses=True)
    yield
    await app.state.redis.aclose()
```

```python
# app/core/cache.py
import redis.asyncio as aioredis
from fastapi import Request

async def get_redis(request: Request) -> AsyncGenerator[aioredis.Redis, None]:
    yield request.app.state.redis
```

## Cache-aside helper

```python
import json

async def get_cached(key: str, ttl: int, fetch_fn, redis: aioredis.Redis):
    cached = await redis.get(key)
    if cached:
        return json.loads(cached)
    value = await fetch_fn()
    await redis.setex(key, ttl, json.dumps(value))
    return value
```

## Usage in a route

```python
@router.get("/popular-items", response_model=list[ItemOut])
async def popular_items(
    db: AsyncSession = Depends(get_db),
    redis: aioredis.Redis = Depends(get_redis),
):
    async def fetch():
        result = await db.execute(select(Item).order_by(Item.views.desc()).limit(10))
        return [ItemOut.model_validate(i).model_dump() for i in result.scalars()]
    return await get_cached("popular-items", ttl=60, fetch_fn=fetch, redis=redis)
```

## Cache invalidation

Two reliable patterns:

- **TTL-only.** Set a short TTL and accept some staleness. Simplest, no
  invalidation bugs.
- **Explicit invalidate on write.** On mutations to the underlying data, call
  `await redis.delete(key)` (or `redis.delete(*keys)` for a list). Use named
  key builders so you don't have stringly-typed keys scattered around.

```python
def popular_items_key() -> str:
    return "popular-items"

# in the write path:
await redis.delete(popular_items_key())
```

Avoid clever pattern-based invalidation (`SCAN` + `DEL`) — it's slow and
error-prone. If you need it, the cache key design is probably wrong.

## What not to cache

- Per-user data without a user-scoped key (data leak risk)
- Auth/session state without explicit TTL matching token expiry
- Anything containing secrets — even short-TTL Redis is a credential store
  if it holds tokens

## Spot-checks

- Pool created in lifespan, closed in shutdown — never per-request
- `decode_responses=True` on the client so values come back as `str`, not bytes
- Every `setex` has a TTL (no `set` without expiry — orphaned keys accumulate)
- Cache keys include all variables that affect the value (user ID, locale,
  feature flag) — otherwise users see each other's data
