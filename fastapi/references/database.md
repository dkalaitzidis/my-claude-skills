# Database & ORM (SQLAlchemy 2.0 async)

Read this when designing models, writing queries, structuring the data-access
layer, or auditing for N+1 / lazy-load bugs. The engine, session factory,
and `get_db` dependency are in the main `SKILL.md` (they're the spine of
the async patterns). This file covers the model + query + repository layer
on top of that foundation.

## Model definition

```python
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship
from sqlalchemy import ForeignKey, String, func
import datetime

from app.core.enums import Role

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    # Role is a StrEnum — string equality works at the DB level, but storing
    # as String keeps migrations simple. Compare with the enum directly:
    # `user.role == Role.ADMIN` works because StrEnum == str.
    role: Mapped[str] = mapped_column(String(50), default=Role.VIEWER)
    hashed_password: Mapped[str] = mapped_column(String(255))
    created_at: Mapped[datetime.datetime] = mapped_column(server_default=func.now())
    posts: Mapped[list["Post"]] = relationship(back_populates="author", lazy="selectin")
```

## Async queries — avoid N+1 with `selectinload` / `joinedload`

```python
from sqlalchemy import select
from sqlalchemy.orm import selectinload

async def get_user_with_posts(db: AsyncSession, user_id: int) -> User | None:
    result = await db.execute(
        select(User)
        .where(User.id == user_id)
        .options(selectinload(User.posts))   # ✅ not lazy-loaded on access
    )
    return result.scalar_one_or_none()
```

Without eager loading, accessing `user.posts` during serialization triggers a
lazy-load — but the session is async, so it raises `MissingGreenlet` instead.
Always eagerly load relationships that appear in the response schema.

## Repository pattern

Keep DB access out of route handlers — routes orchestrate, repositories query:

```python
class UserRepository:
    def __init__(self, db: AsyncSession):
        self.db = db

    async def get_by_email(self, email: str) -> User | None:
        result = await self.db.execute(select(User).where(User.email == email))
        return result.scalar_one_or_none()

    async def create(self, email: str, hashed_password: str) -> User:
        user = User(email=email, hashed_password=hashed_password)
        self.db.add(user)
        await self.db.commit()
        await self.db.refresh(user)
        return user
```

Repositories raise domain exceptions (`NotFoundError`, `ConflictError`),
never `HTTPException` — see `error-handling.md`.

## Spot-checks for DB code

- Every relationship that's serialized has `lazy="selectin"` or is loaded with
  `selectinload` / `joinedload` at query time
- No raw SQL strings without `text()`
- `expire_on_commit=False` on the session factory (set in SKILL.md) — otherwise
  attributes are invalidated after `commit()` and require an extra round-trip
- Every list query has an `order_by` (required for stable pagination)
- Every commit is followed by `refresh()` if you need server-generated columns
  (autoincrement ids, `server_default` timestamps) in the response
