---
title: "Batch a SQLite IN Query Under the Connection Bind Limit"
snippet_type: recipe
use_cases:
  - persistence
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - compile-a-bounded-t-string-into-sqlite-qmark-sql-and-parameters.md
  - hold-one-sqlite-wal-read-snapshot-across-multiple-queries.md
---

# Batch a SQLite IN Query Under the Connection Bind Limit

## Idea and Problem

Look up a finite set of IDs in SQLite without exceeding the active connection's statement parameter limit.

Every `?` in this `IN` list uses one parameter slot, and the collection
filter uses another slot. Read the live limit and subtract the fixed slot
before dividing the sorted, unique IDs into batches. Only the number of
placeholders changes between statements. All values are bound separately.

## When to Use

Use this for a small lookup over a finite iterable of SQLite integer IDs.
The example table has one row per `(collection, item_id)` key. A missing ID
contributes no row, and an empty input skips SQL entirely. The caller owns
the connection and its transaction.

## Implementation

```python
import sqlite3
from collections.abc import Iterable
from contextlib import closing
from itertools import batched


def find_items(
    connection: sqlite3.Connection,
    collection: str,
    item_ids: Iterable[int],
) -> tuple[int, ...]:
    if type(collection) is not str:
        raise TypeError("collection must be a string")

    unique_ids: set[int] = set()
    for item_id in item_ids:
        if type(item_id) is not int:
            raise TypeError("item IDs must be integers")
        unique_ids.add(item_id)
    if not unique_ids:
        return ()

    available = connection.getlimit(sqlite3.SQLITE_LIMIT_VARIABLE_NUMBER) - 1
    if available < 1:
        raise ValueError("the connection has no bind slot left for an item ID")

    found: list[int] = []
    with closing(connection.cursor()) as cursor:
        cursor.row_factory = None
        for ids in batched(sorted(unique_ids), available, strict=False):
            placeholders = ", ".join("?" for _ in ids)
            cursor.execute(
                "SELECT item_id FROM main.items WHERE collection = ? "
                f"AND item_id IN ({placeholders}) ORDER BY item_id",
                (collection, *ids),
            )
            found.extend(item_id for (item_id,) in cursor)
    return tuple(found)
```

## Example

```python
with closing(sqlite3.connect(":memory:", cached_statements=0)) as connection:
    with closing(connection.cursor()) as cursor:
        cursor.executescript("""
            CREATE TABLE items (
                collection TEXT NOT NULL,
                item_id INTEGER NOT NULL,
                PRIMARY KEY (collection, item_id)
            );
            INSERT INTO items VALUES ('shelf', 1), ('shelf', 2), ('shelf', 4), ('other', 3);
        """)
        connection.setlimit(sqlite3.SQLITE_LIMIT_VARIABLE_NUMBER, 3)

        single_query_rejected = False
        try:
            cursor.execute("SELECT ?, ?, ?, ?", (1, 2, 3, 4))
        except sqlite3.OperationalError:
            single_query_rejected = True

    matched = find_items(connection, "shelf", (4, 2, 2, 3, 1, 99))
    empty = find_items(connection, "shelf", ())

assert (single_query_rejected, matched, empty) == (True, (1, 2, 4), ())
```

## Trade-offs and Limitations

The helper holds the unique inputs and matching IDs in memory. The combined
output is ascending because each sorted batch covers a disjoint section of
the ID order and each query sorts its own results. The bind limit constrains
statement shape. Work and result size still depend on the input and table.

Use default result conversion and keep the connection exclusive during the
lookup. Several statements can observe different commits when run outside an
explicit read transaction. Hold a suitable caller-owned snapshot when the
results need to describe one database state.

## Related Snippets

<!-- catalog:related:start -->
- [Compile a Bounded T-String into SQLite Qmark SQL and Parameters](compile-a-bounded-t-string-into-sqlite-qmark-sql-and-parameters.md)
- [Hold One SQLite WAL Read Snapshot Across Multiple Queries](hold-one-sqlite-wal-read-snapshot-across-multiple-queries.md)
<!-- catalog:related:end -->
