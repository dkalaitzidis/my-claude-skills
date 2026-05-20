# Pagination

Read this file when writing any list endpoint. Every endpoint returning a
collection must paginate — unbounded collections are a hard rule from the
main `SKILL.md`.

## Which style to use

| Style | Use when |
|---|---|
| **Cursor (keyset)** | Default. Any table that can grow large, infinite-scroll UIs, APIs without random-page-jump requirements. Stable under inserts and deletes. |
| **Offset** | Only when the UI requires numbered pages on a small, slow-changing dataset (e.g. an archive of reports). Avoid for large or frequently updated tables — page contents shift as rows are inserted. |

Always document which style is used and why in a docstring or inline comment.

---

## Cursor-based pagination (default)

The cursor is the primary key of the last row returned. The next page is
"all rows with id greater than the cursor, ordered by id, limit N". This is
stable under concurrent inserts and avoids the offset-scan performance cliff
on large tables.

```python
from pydantic import BaseModel

class CursorPage(BaseModel):
    items: list[ItemOut]
    next_cursor: int | None       # null when there are no more pages

async def paginate_items(db: AsyncSession, after_id: int = 0, limit: int = 20) -> CursorPage:
    # fetch limit + 1 to detect whether there's another page
    result = await db.execute(
        select(Item).where(Item.id > after_id).order_by(Item.id).limit(limit + 1)
    )
    items = result.scalars().all()
    has_more = len(items) > limit
    return CursorPage(
        items=items[:limit],
        next_cursor=items[limit - 1].id if has_more else None,
    )
```

---

## Enforce a hard page-size cap on every list route

`MAX_PAGE_SIZE` defends against accidental or malicious requests for huge
pages. `Query(le=MAX_PAGE_SIZE)` makes FastAPI reject overlimit requests
with `422` automatically — no custom validation needed.

```python
from fastapi import Query
from typing import Annotated

MAX_PAGE_SIZE = 100

@router.get("/items", response_model=CursorPage)
async def list_items(
    after_id: Annotated[int, Query(ge=0)] = 0,
    limit: Annotated[int, Query(ge=1, le=MAX_PAGE_SIZE)] = 20,
    db: AsyncSession = Depends(get_db),
) -> CursorPage:
    # cursor pagination: items table can grow large, no random-page-jump UI
    return await paginate_items(db, after_id=after_id, limit=limit)
```

Client usage:

```
GET /items?limit=20                  # first page
GET /items?after_id=42&limit=20      # next page after item id 42
```

---

## Cursor on something other than `id`

If you need to paginate by `created_at` (e.g. newest-first feeds), use a
composite cursor of `(created_at, id)` to break ties when two rows share
the same timestamp:

```python
from sqlalchemy import tuple_, and_

async def paginate_by_created(
    db: AsyncSession,
    after_created: datetime | None = None,
    after_id: int | None = None,
    limit: int = 20,
) -> CursorPage:
    stmt = select(Item).order_by(Item.created_at.desc(), Item.id.desc()).limit(limit + 1)
    if after_created is not None and after_id is not None:
        stmt = stmt.where(
            tuple_(Item.created_at, Item.id) < tuple_(after_created, after_id)
        )
    result = await db.execute(stmt)
    items = result.scalars().all()
    has_more = len(items) > limit
    return CursorPage(...)
```

For client-friendliness, encode `(created_at, id)` as an opaque base64 token
in the API so callers don't need to know the internal cursor shape.

---

## Offset-based pagination (use sparingly)

Only when the UI requires numbered pages on a small, slow-changing dataset.
Always document why offset was chosen instead of cursor:

```python
from sqlalchemy import func

class OffsetPage(BaseModel):
    items: list[ReportOut]
    total: int
    page: int
    size: int

@router.get("/reports", response_model=OffsetPage)
async def list_reports(
    page: Annotated[int, Query(ge=1)] = 1,
    size: Annotated[int, Query(ge=1, le=MAX_PAGE_SIZE)] = 20,
    db: AsyncSession = Depends(get_db),
) -> OffsetPage:
    # offset pagination: report archive is append-only and UI needs page numbers
    offset = (page - 1) * size
    result = await db.execute(
        select(Report).order_by(Report.id.desc()).offset(offset).limit(size)
    )
    total = await db.scalar(select(func.count(Report.id)))
    return OffsetPage(items=result.scalars().all(), total=total, page=page, size=size)
```

**Performance note:** `OFFSET N` makes Postgres scan and discard the first N
rows. At page 1,000 with size 20 that's 20,000 rows scanned per request.
For tables over ~100k rows, switch to cursor pagination.

---

## Spot-checks for any list endpoint

- Has explicit pagination params (`after_id`/`limit` or `page`/`size`)
- Enforces a `MAX_PAGE_SIZE` cap via `Query(le=...)`
- Returns the cap-rejected case as `422` automatically (no custom handler needed)
- Has a `next_cursor` or `total` so clients can detect the last page
- Has a stable `order_by` — never paginate without an explicit ordering
- Uses cursor by default; offset is documented with a justification when used

---

## Testing pagination

See `testing.md` for the full examples. Key cases to cover:

- Respects `limit` (returns at most N items)
- Rejects `limit > MAX_PAGE_SIZE` with 422
- Cursor advances correctly (page 2 has no overlap with page 1)
- `next_cursor` is `null` on the last page
