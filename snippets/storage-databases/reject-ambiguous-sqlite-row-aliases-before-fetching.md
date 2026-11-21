---
title: "Reject Ambiguous SQLite Row Aliases Before Fetching"
snippet_type: pattern
use_cases:
  - persistence
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - audit-one-sqlite-table-against-an-exact-ordered-column-contract.md
  - execute-a-trusted-sqlite-query-under-progress-callback-and-row-caps.md
---

# Reject Ambiguous SQLite Row Aliases Before Fetching

## Idea and Problem

Check a SQLite result's column names before reading any rows by name.

A join can expose two columns named `id`, or `id` and `ID`. Position still
distinguishes the values, but `sqlite3.Row` looks up ASCII column names without
regard to case. Check `cursor.description` immediately after execution. The
description carries the projected names even if the query matched no rows.

## When to Use

Use this when your own SQLite queries feed code that accesses `sqlite3.Row`
values by column name. Configure the row factory before fetching and choose
explicit, nonempty ASCII output names. Call the check after each `SELECT`
execution, before consuming that cursor's result.

## Implementation

```python
import sqlite3


def require_unambiguous_row_aliases(cursor: sqlite3.Cursor) -> tuple[str, ...]:
    description = cursor.description
    if description is None:
        raise ValueError("the cursor has no result columns")

    spellings: dict[str, str] = {}
    for column in description:
        name = column[0]
        if not name or not name.isascii():
            raise ValueError(f"expected a nonempty ASCII column name, got {name!r}")
        lookup_key = name.lower()
        if lookup_key in spellings:
            prior = spellings[lookup_key]
            raise ValueError(f"ambiguous column names: {prior!r} and {name!r}")
        spellings[lookup_key] = name

    return tuple(spellings.values())
```

## Example

```python
from contextlib import closing

with closing(sqlite3.connect(":memory:")) as connection:
    connection.row_factory = sqlite3.Row
    with closing(connection.cursor()) as cursor:
        cursor.executescript("""
            CREATE TABLE shelves (id INTEGER PRIMARY KEY);
            CREATE TABLE books (id INTEGER PRIMARY KEY, shelf_id INTEGER);
            INSERT INTO shelves VALUES (7);
            INSERT INTO books VALUES (42, 7);
        """)

        cursor.execute("""
            SELECT s.id AS id, b.id AS ID
            FROM shelves AS s JOIN books AS b ON b.shelf_id = s.id
            WHERE 0
        """)
        collision = None
        try:
            require_unambiguous_row_aliases(cursor)
        except ValueError as error:
            collision = str(error)
        empty_rows = cursor.fetchall()

        cursor.execute("""
            SELECT s.id AS shelf_id, b.id AS book_id
            FROM shelves AS s JOIN books AS b ON b.shelf_id = s.id
        """)
        names = require_unambiguous_row_aliases(cursor)
        row = cursor.fetchone()
        assert row is not None
        values = (row["SHELF_ID"], row["book_id"])

assert (collision, empty_rows, names, values) == (
    "ambiguous column names: 'id' and 'ID'",
    [],
    ("shelf_id", "book_id"),
    (7, 42),
)
```

## Trade-offs and Limitations

The check walks the column names once and stores one lookup key per column.
It preserves the original spellings and query order. The ASCII restriction
gives the check a narrow case rule. A custom row factory can use a different
lookup policy and needs a corresponding check.

The check reads metadata from a query the caller has already executed. The
caller still owns the SQL, cursor lifetime and transaction. For an adapter
that requires specific field names or types, check that separate contract
before interpreting the values.

## Related Snippets

<!-- catalog:related:start -->
- [Audit One SQLite Table Against an Exact Ordered Column Contract](audit-one-sqlite-table-against-an-exact-ordered-column-contract.md)
- [Execute a Trusted SQLite Query Under Progress-Callback and Row Caps](execute-a-trusted-sqlite-query-under-progress-callback-and-row-caps.md)
<!-- catalog:related:end -->
