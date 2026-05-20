# Response model discipline

Read this file when designing request/response schemas, writing route
handlers that return data, or auditing for ORM-leak bugs (returning
SQLAlchemy models directly).

## The rule

Always declare `response_model=` explicitly. Never return ORM objects
directly — always project through a Pydantic schema. This is what stops
internal fields (`hashed_password`, `internal_notes`, `deleted_at`) from
leaking into API responses by accident.

---

## Schema definitions

Define three schemas per resource: `Out` (response), `Create` (POST body),
`Patch` (PATCH body with all fields optional). For larger resources you may
also need `Update` (PUT body with all fields required).

```python
# app/schemas/users.py
from pydantic import BaseModel, ConfigDict, EmailStr

class UserOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)  # enables ORM → Pydantic conversion

    id: int
    email: EmailStr
    role: str
    # note: no hashed_password, no internal fields

class UserCreate(BaseModel):
    email: EmailStr
    password: str        # plain text in the request, hashed before storage

class UserPatch(BaseModel):
    email: EmailStr | None = None   # all fields optional for PATCH
    role: str | None = None
```

`from_attributes=True` is what lets `UserOut.model_validate(orm_user)` work
— Pydantic reads attributes off the ORM object rather than expecting a dict.

---

## GET — always declare `response_model`

```python
@router.get("/{user_id}", response_model=UserOut)
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)) -> UserOut:
    user = await db.get(User, user_id)
    if user is None:
        raise NotFoundError("User", user_id)
    return UserOut.model_validate(user)   # explicit ORM → schema conversion
```

The explicit `model_validate` call is belt-and-braces — `response_model=`
would do the conversion anyway, but writing it out makes the intent
obvious and gives you a typed value to assert on in tests.

---

## POST — return the created resource

```python
@router.post("/users", response_model=UserOut, status_code=201)
async def create_user(
    payload: UserCreate,
    db: AsyncSession = Depends(get_db),
) -> UserOut:
    existing = await UserRepository(db).get_by_email(payload.email)
    if existing is not None:
        raise ConflictError("A user with this email already exists.")
    user = await UserRepository(db).create(
        email=payload.email,
        hashed_password=hash_password(payload.password),
    )
    return UserOut.model_validate(user)
```

Note that `payload.password` (plain text) never appears in the response —
`UserOut` doesn't have a `password` field, so it can't.

---

## PATCH — use `model_dump(exclude_unset=True)`

So only provided fields are written, preventing accidental overwrites with
`None`:

```python
@router.patch("/{user_id}", response_model=UserOut)
async def patch_user(
    user_id: int,
    payload: UserPatch,
    db: AsyncSession = Depends(get_db),
) -> UserOut:
    user = await db.get(User, user_id)
    if user is None:
        raise NotFoundError("User", user_id)
    update_data = payload.model_dump(exclude_unset=True)  # only set fields
    for field, value in update_data.items():
        setattr(user, field, value)
    await db.commit()
    await db.refresh(user)
    return UserOut.model_validate(user)
```

The distinction matters:
- `model_dump()` includes all fields, with `None` for unset → would
  overwrite the user's email with `None` if the client only sent `role`
- `model_dump(exclude_unset=True)` includes only fields the client
  explicitly sent → safe partial updates

---

## DELETE — return 204, no body

```python
@router.delete("/{user_id}", status_code=204)
async def delete_user(user_id: int, db: AsyncSession = Depends(get_db)) -> None:
    user = await db.get(User, user_id)
    if user is None:
        raise NotFoundError("User", user_id)
    await db.delete(user)
    await db.commit()
```

No `response_model` because there's no body. The return type is `None` and
the status is `204 No Content`.

---

## Annotate error responses in OpenAPI

So clients see the full contract — not just the happy-path schema:

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

For app-wide defaults (every route returns `ErrorResponse` on 422 and 500),
set them on the `FastAPI` instance — see `error-handling.md`.

---

## Lists — paginate, always

Never return a raw `list[UserOut]` from a list endpoint. Always wrap it in
a pagination response model:

```python
class UserListPage(BaseModel):
    items: list[UserOut]
    next_cursor: int | None
```

See `pagination.md` for the full cursor and offset patterns.

---

## Nested schemas and relationships

When a response includes a related resource, define a nested schema and
make sure the relationship is eagerly loaded:

```python
class PostOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    title: str

class UserWithPostsOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    email: EmailStr
    posts: list[PostOut]    # nested

@router.get("/{user_id}/with-posts", response_model=UserWithPostsOut)
async def get_user_with_posts(user_id: int, db: AsyncSession = Depends(get_db)) -> UserWithPostsOut:
    result = await db.execute(
        select(User).where(User.id == user_id).options(selectinload(User.posts))  # ⚠ avoid N+1
    )
    user = result.scalar_one_or_none()
    if user is None:
        raise NotFoundError("User", user_id)
    return UserWithPostsOut.model_validate(user)
```

Without `selectinload`, accessing `user.posts` during serialization would
trigger a lazy-load — but the session is async, so it raises a
`MissingGreenlet` error instead. Always eagerly load relationships that
appear in the response schema.

---

## Spot-checks for response models

- Every route has an explicit `response_model=` (or `status_code=204` and `None` return)
- No route returns an ORM object directly — always via `Schema.model_validate(orm_obj)`
- PATCH routes use `model_dump(exclude_unset=True)`, not `model_dump()`
- Response schemas never include sensitive fields (`hashed_password`, internal IDs, soft-delete timestamps)
- Nested relationships in response schemas are eagerly loaded with `selectinload` or `joinedload`
- List endpoints return a `Page`-style wrapper, never a bare list
- Error responses are documented in `responses=` so they appear in OpenAPI
