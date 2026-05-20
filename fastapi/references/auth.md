# Auth & security

Read this file when implementing authentication or authorisation — JWT
issuance and verification, password hashing, current-user dependencies, or
role-based access control. The patterns here assume the settings, DB, and
project layout from the main `SKILL.md`.

> **Coupled with error handling.** `get_current_user` and `require_role` raise
> `AuthenticationError` / `PermissionDeniedError`, which only become proper
> HTTP responses because of the handlers in `error-handling.md`. If you're
> implementing auth, read both files.

## Library choice

| Library | When to use |
|---|---|
| **`PyJWT`** | Rolling your own JWT auth. Actively maintained, minimal deps. The default for this skill. |
| **`fastapi-users`** | You want registration, email verification, password reset, and OAuth2 out of the box. Skip the manual JWT code below and follow its SQLAlchemy integration guide instead. |
| **`python-jose`** / **`fastapi-jwt-auth`** | **Avoid on new projects** — unmaintained or superseded. |

---

## Domain exception classes

These are referenced throughout the auth and error-handling layers. Define
them once in `app/core/exceptions.py`:

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

These are caught and converted to `ErrorResponse` JSON by the handler
registered in `error-handling.md`. Never use raw `HTTPException` inside
business logic — it leaks HTTP concerns into your domain layer.

---

## JWT access + refresh tokens (PyJWT)

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
    return jwt.encode(
        {"sub": subject, "exp": expire},
        settings.secret_key,
        algorithm=ALGORITHM,
    )

def create_refresh_token(
    subject: str,
    settings: Settings,
    expires_delta: timedelta = timedelta(days=7),
) -> str:
    expire = datetime.now(timezone.utc) + expires_delta
    return jwt.encode(
        {"sub": subject, "exp": expire, "type": "refresh"},
        settings.secret_key,
        algorithm=ALGORITHM,
    )

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

### Token expiry

- **Access token** — 15 minutes is a reasonable default. Short enough that a
  leaked token isn't catastrophic; long enough to avoid hammering the auth
  endpoint.
- **Refresh token** — 7 days for typical web apps, 30 days for mobile. Store
  refresh tokens in `httpOnly` cookies (web) or secure storage (mobile),
  never in `localStorage`.
- **Drive expiry from settings** — `settings.access_token_expire_minutes`,
  `settings.refresh_token_expire_days` (already in the example `Settings`
  class in `SKILL.md`).

---

## Password hashing (bcrypt)

```python
# app/core/security.py (continued)
import bcrypt

def hash_password(password: str) -> str:
    return bcrypt.hashpw(password.encode("utf-8"), bcrypt.gensalt()).decode("utf-8")

def verify_password(password: str, hashed: str) -> bool:
    return bcrypt.checkpw(password.encode("utf-8"), hashed.encode("utf-8"))
```

> **Why `bcrypt` directly and not `passlib`?** `passlib` had its last release
> in 2020 and is effectively unmaintained — it produces noisy warnings against
> modern `bcrypt` versions (`AttributeError: module 'bcrypt' has no attribute '__about__'`).
> Use `bcrypt` directly, or `argon2-cffi` if you prefer Argon2.

### Cost factor

`bcrypt.gensalt()` defaults to 12 rounds. This is fine for most apps; bump
to 13 or 14 if your threat model warrants it and you've measured the latency
impact (each round doubles the time).

---

## RBAC via dependency

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

`Role` lives in `app/core/enums.py` — see the "Shared enums and types"
section in the main `SKILL.md`.

For more granular permissions (e.g. "this user can edit *this specific*
resource"), write a resource-scoped dependency:

```python
async def require_resource_owner(
    item_id: int,
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
) -> Item:
    item = await db.get(Item, item_id)
    if item is None:
        raise NotFoundError("Item", item_id)
    if item.owner_id != current_user.id and current_user.role != Role.ADMIN:
        raise PermissionDeniedError()
    return item
```

---

## Login endpoint sketch

```python
from fastapi.security import OAuth2PasswordRequestForm

@router.post("/auth/token")
async def login(
    form: OAuth2PasswordRequestForm = Depends(),
    db: AsyncSession = Depends(get_db),
    settings: Settings = Depends(get_settings),
) -> dict:
    user = await UserRepository(db).get_by_email(form.username)
    if user is None or not verify_password(form.password, user.hashed_password):
        raise AuthenticationError()
    return {
        "access_token": create_access_token(str(user.id), settings),
        "refresh_token": create_refresh_token(str(user.id), settings),
        "token_type": "bearer",
    }
```

Apply a stricter rate limit to this endpoint to prevent brute force — see
the rate-limiting section in the main `SKILL.md`.

---

## Spot-checks for auth code

- `secret_key` is loaded from settings, never hardcoded
- `get_current_user` injects `settings` via `Depends`, not a module global
- `AuthenticationError` is raised on every failure mode (missing token,
  invalid signature, expired token, missing `sub`, user not found) — never
  return different shapes for different failures
- Password hashing uses `bcrypt` directly, not `passlib`
- The login endpoint has a tight rate limit (`@limiter.limit("5/minute")`)
- Refresh tokens carry a `type: "refresh"` claim and are rejected if used as
  access tokens
