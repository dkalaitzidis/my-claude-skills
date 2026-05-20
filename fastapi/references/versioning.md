# API versioning

Read this when introducing a new API version, deprecating an old endpoint,
or deciding whether a change needs a new version at all.

## Style

Default to **URL versioning** (`/v1/`, `/v2/`). Reasons:

- Explicit — visible in every log line, request, and OpenAPI URL
- Cacheable — CDNs and reverse proxies route on the path without surgery
- Easy to retire — drop a router, return 410 Gone on the old prefix
- Tooling support — OpenAPI clients generate separate modules per version

Alternatives (header versioning, query parameter, content negotiation) are
all viable but lose at least one of those properties. Stick with URL unless
you have a concrete reason.

## Layout

```
app/routers/
├── v1/
│   ├── users.py
│   └── items.py
└── v2/
    ├── users.py
    └── items.py
```

```python
# app/routers/v1/users.py
from fastapi import APIRouter
router = APIRouter(prefix="/users", tags=["v1 / users"])
```

```python
# app/main.py
from app.routers.v1 import users as users_v1
from app.routers.v2 import users as users_v2

app.include_router(users_v1.router, prefix="/v1")
app.include_router(users_v2.router, prefix="/v2")
```

## When to bump the version

| Change | New version? |
|---|---|
| Add a new endpoint | No |
| Add an optional field to a response | No |
| Add an optional query/body field with a sensible default | No |
| Add a required field to a request body | Yes |
| Remove or rename a field in a response | Yes |
| Change the type of a field | Yes |
| Change the semantics of a status code | Yes |
| Change error `code` slugs | Yes (clients switch on them — see `error-handling.md`) |

Additive changes never need a new version. Breaking changes always do.

## Marking deprecated endpoints

```python
@router.get("/{user_id}", deprecated=True, response_model=UserOutV1)
async def get_user_v1(...):
    """Deprecated. Use v2 — adds `display_name` and removes `legacy_id`."""
    ...
```

`deprecated=True` surfaces in OpenAPI / `/docs` so client teams notice.
Pair it with a deprecation warning header for automated clients:

```python
@router.get("/{user_id}", deprecated=True)
async def get_user_v1(response: Response, ...):
    response.headers["Deprecation"] = "true"
    response.headers["Sunset"] = "Wed, 31 Dec 2026 23:59:59 GMT"
    ...
```

## Sharing logic between versions

Put the business logic in a service or repository — the routers are just
the HTTP shell. v1 and v2 routers both call the same `UserService.create()`,
differing only in request/response schema shape.

```python
# v1 schema preserves `legacy_id`
class UserOutV1(BaseModel):
    id: int
    email: EmailStr
    legacy_id: int

# v2 drops it
class UserOutV2(BaseModel):
    id: int
    email: EmailStr
    display_name: str
```

Don't duplicate validation, persistence, or business rules across versions —
they drift and the bugs are subtle.

## Retiring an old version

1. Add `deprecated=True` and `Sunset` header — communicate the date publicly
2. Add a `Deprecation` log event every time a v1 endpoint is hit, with the
   client IP / API key so you can identify the laggards
3. After the sunset date, return `410 Gone` (not `404`) so clients know it
   used to exist but is intentionally retired
4. Drop the router once usage is zero

## Spot-checks

- New endpoints land in the current version, not pinned to an old one
- Deprecated endpoints have both `deprecated=True` and a `Sunset` header
- Business logic lives in services/repositories, not in routers — versions
  share it
- Breaking changes never go into an existing version
- Error `code` slugs are treated as part of the API contract — renaming one
  is a breaking change
