---
title: "Page Bounded SQLite Rows with a Composite Keyset Cursor"
snippet_type: pattern
use_cases:
  - performance-optimization
  - persistence
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - plan-bounded-table-initialization-and-ordered-row-batches.md
  - plan-bounded-parameterized-backfill-statements-by-date-window.md
  - scope-caller-owned-sqlite-work-with-an-explicit-savepoint.md
---

# Page Bounded SQLite Rows with a Composite Keyset Cursor

## Idea and Problem

Read a fixed SQLite catalog in stable composite-key order without making later pages scan and discard every preceding row.

The first query starts at the beginning. Each subsequent query uses a row-value
comparison against the last returned `(sort_key, item_id)` pair. Fetching one
extra row proves that another page exists, but that lookahead row is not
consumed: the next query starts after the last row actually returned.

Non-null sort keys, a unique tie-breaker, and immutability of both ordering
components make the cursor position unambiguous. A matching composite index
lets SQLite seek near that position instead of applying a growing offset.

## When to Use

Use this pattern for forward-only traversal of a trusted, fixed-schema table
when rows are ordered by a possibly repeated signed-integer sort key and a
unique signed-integer identifier. It suits bounded API pages and background
scans that can carry an opaque application-level cursor between calls.

Keep one caller-owned read transaction open across the complete traversal when
every page must observe the same database snapshot. SQLite otherwise permits
committed inserts, deletes, or sort-key changes between calls to affect later
pages. The helper deliberately neither begins nor ends a transaction.

## Implementation

```python
import sqlite3
from dataclasses import dataclass

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_PAGE_SIZE = 100
_MAX_PAYLOAD_BYTES = 4_096

_FIRST_PAGE_SQL = """
SELECT sort_key, item_id, payload
FROM catalog_items
ORDER BY sort_key, item_id
LIMIT ?
"""

_NEXT_PAGE_SQL = """
SELECT sort_key, item_id, payload
FROM catalog_items
WHERE (sort_key, item_id) > (?, ?)
ORDER BY sort_key, item_id
LIMIT ?
"""


@dataclass(frozen=True, slots=True)
class CatalogCursor:
    sort_key: int
    item_id: int


@dataclass(frozen=True, slots=True)
class CatalogItem:
    sort_key: int
    item_id: int
    payload: str


@dataclass(frozen=True, slots=True)
class CatalogPage:
    items: tuple[CatalogItem, ...]
    next_cursor: CatalogCursor | None


def _validate_int64(name: str, value: object) -> int:
    if type(value) is not int:
        raise TypeError(f"{name} must be an exact integer")
    if not _MIN_INT64 <= value <= _MAX_INT64:
        raise ValueError(f"{name} is outside the signed 64-bit range")
    return value


def _validate_cursor(after: object) -> CatalogCursor | None:
    if after is None:
        return None
    if type(after) is not CatalogCursor:
        raise TypeError("after must be an exact CatalogCursor or None")
    return CatalogCursor(
        _validate_int64("after.sort_key", after.sort_key),
        _validate_int64("after.item_id", after.item_id),
    )


def page_catalog_items(
    connection: sqlite3.Connection,
    *,
    after: CatalogCursor | None = None,
    page_size: int,
) -> CatalogPage:
    """Return one validated forward keyset page from the fixed catalog table."""
    if type(connection) is not sqlite3.Connection:
        raise TypeError("connection must be an exact sqlite3.Connection")
    validated_after = _validate_cursor(after)
    if type(page_size) is not int:
        raise TypeError("page_size must be an exact integer")
    if not 1 <= page_size <= _MAX_PAGE_SIZE:
        raise ValueError("page_size is outside the supported range")

    cursor = connection.cursor()
    cursor.row_factory = None
    try:
        if validated_after is None:
            fetched = cursor.execute(
                _FIRST_PAGE_SQL,
                (page_size + 1,),
            ).fetchall()
        else:
            fetched = cursor.execute(
                _NEXT_PAGE_SQL,
                (
                    validated_after.sort_key,
                    validated_after.item_id,
                    page_size + 1,
                ),
            ).fetchall()
    finally:
        cursor.close()

    if len(fetched) > page_size + 1:
        raise RuntimeError("fixed page query returned more rows than requested")

    validated_rows: list[CatalogItem] = []
    previous_key = (
        (validated_after.sort_key, validated_after.item_id)
        if validated_after is not None
        else None
    )
    for raw_row in fetched:
        if type(raw_row) is not tuple or len(raw_row) != 3:
            raise TypeError("fixed page query must return exact three-value tuples")
        sort_key = _validate_int64("row sort_key", raw_row[0])
        item_id = _validate_int64("row item_id", raw_row[1])
        payload = raw_row[2]
        if type(payload) is not str:
            raise TypeError("row payload must be an exact string")
        try:
            payload_bytes = len(payload.encode("utf-8"))
        except UnicodeEncodeError as error:
            raise ValueError("row payload contains a Unicode surrogate") from error
        if payload_bytes > _MAX_PAYLOAD_BYTES:
            raise ValueError("row payload exceeds the supported UTF-8 byte limit")

        current_key = (sort_key, item_id)
        if previous_key is not None and current_key <= previous_key:
            raise RuntimeError("fixed page query returned non-increasing keys")
        previous_key = current_key
        validated_rows.append(CatalogItem(sort_key, item_id, payload))

    returned = tuple(validated_rows[:page_size])
    if len(validated_rows) > page_size:
        last_returned = returned[-1]
        next_cursor = CatalogCursor(
            last_returned.sort_key,
            last_returned.item_id,
        )
    else:
        next_cursor = None
    return CatalogPage(returned, next_cursor)
```

## Example

```python
connection = sqlite3.connect(":memory:")
connection.executescript(
    """
    CREATE TABLE catalog_items (
        item_id INTEGER PRIMARY KEY,
        sort_key INTEGER NOT NULL CHECK (typeof(sort_key) = 'integer'),
        payload TEXT NOT NULL CHECK (
            typeof(payload) = 'text'
            AND length(CAST(payload AS BLOB)) <= 4096
        )
    ) STRICT;
    CREATE INDEX catalog_items_page_order
        ON catalog_items(sort_key, item_id);
    CREATE TRIGGER catalog_items_order_key_immutable
    BEFORE UPDATE OF sort_key, item_id ON catalog_items
    WHEN NEW.sort_key IS NOT OLD.sort_key OR NEW.item_id IS NOT OLD.item_id
    BEGIN
        SELECT RAISE(ABORT, 'catalog ordering keys are immutable');
    END;
    """
)
connection.executemany(
    "INSERT INTO catalog_items(item_id, sort_key, payload) VALUES (?, ?, ?)",
    (
        (40, 3, "delta"),
        (10, 1, "alpha"),
        (50, 3, "echo"),
        (30, 2, "charlie"),
        (20, 1, "bravo"),
    ),
)

try:
    connection.execute(
        "UPDATE catalog_items SET sort_key = ? WHERE item_id = ?",
        (9, 10),
    )
except sqlite3.IntegrityError:
    ordering_key_update_rejected = True
else:
    ordering_key_update_rejected = False

first = page_catalog_items(connection, page_size=2)
second = page_catalog_items(
    connection,
    after=first.next_cursor,
    page_size=2,
)
third = page_catalog_items(
    connection,
    after=second.next_cursor,
    page_size=2,
)
connection.close()

assert ordering_key_update_rejected
assert tuple(item.item_id for item in first.items) == (10, 20)
assert first.next_cursor == CatalogCursor(1, 20)
assert tuple(item.item_id for item in second.items) == (30, 40)
assert second.next_cursor == CatalogCursor(3, 40)
assert tuple(item.item_id for item in third.items) == (50,)
assert third.next_cursor is None
```

## Trade-offs and Limitations

With an index on `(sort_key, item_id)`, SQLite is expected to seek near the
cursor and read `O(page_size)` adjacent entries, commonly giving
`O(log N + page_size)` work. The selected query plan still depends on SQLite's
statistics and planner; inspect `EXPLAIN QUERY PLAN` for the deployed schema
rather than treating that complexity as a guarantee.

All fetched rows, including the lookahead row, are validated before any page is
returned. This avoids partial output on malformed data. Cursor values are
structural positions, not proof that a row exists, so an in-range forged cursor
simply starts traversal after that key.

The table contract requires non-null signed 64-bit sort keys, unique signed
64-bit item identifiers, immutability of both ordering fields, and payloads of
at most 4,096 UTF-8 bytes. This snippet does not build or migrate that schema,
support dynamic filters,
paginate backward, handle nullable keys or custom collations, sign cursors,
manage connections or transactions, retry database errors, or coordinate
pagination across different databases.

## Related Snippets

<!-- catalog:related:start -->
- [Plan Bounded Table Initialization and Ordered Row Batches](plan-bounded-table-initialization-and-ordered-row-batches.md)
- [Plan Bounded Parameterized Backfill Statements by Date Window](plan-bounded-parameterized-backfill-statements-by-date-window.md)
- [Scope Caller-Owned SQLite Work with an Explicit Savepoint](scope-caller-owned-sqlite-work-with-an-explicit-savepoint.md)
<!-- catalog:related:end -->
