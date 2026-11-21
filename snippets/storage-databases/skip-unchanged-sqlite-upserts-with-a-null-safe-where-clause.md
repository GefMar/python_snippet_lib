---
title: "Skip Unchanged SQLite Upserts with a Null-Safe WHERE Clause"
snippet_type: recipe
use_cases:
  - configuration
  - persistence
tested_python:
  - "3.14"
dependencies: []
related:
  - compare-direct-and-triggered-sqlite-row-changes-for-one-write.md
  - enforce-one-active-sqlite-row-per-key-with-a-partial-unique-index.md
---

# Skip Unchanged SQLite Upserts with a Null-Safe WHERE Clause

## Idea and Problem

Insert new settings and update existing ones only when the stored value differs.

Add a `WHERE` clause to the upsert's update branch. Unlike `!=`,
`IS DISTINCT FROM` treats two SQL `NULL` values as equal and one
`NULL` paired with a non-null value as different. On this schema,
`RETURNING` yields a row for an insert or an update, but not for a skip.

## When to Use

Use this to apply the latest setting by primary key while avoiding repeated
update triggers for equal values. The value is nullable text with the default
binary collation. This spelling requires SQLite 3.39 or later. The caller
owns the connection and controls the transaction.

## Implementation

```python
import sqlite3
from contextlib import closing


def store_preference(connection: sqlite3.Connection, key: str, value: str | None) -> bool:
    with closing(connection.cursor()) as cursor:
        cursor.row_factory = None
        cursor.execute(
            """
            INSERT INTO main.preferences (key, value) VALUES (?, ?)
            ON CONFLICT (key) DO UPDATE SET value = excluded.value
            WHERE preferences.value IS DISTINCT FROM excluded.value
            RETURNING 1
            """,
            (key, value),
        )
        return cursor.fetchone() is not None
```

## Example

```python
with closing(sqlite3.connect(":memory:", autocommit=False)) as connection:
    with closing(connection.cursor()) as cursor:
        cursor.executescript("""
            CREATE TABLE preferences (key TEXT PRIMARY KEY, value TEXT) STRICT;
            CREATE TABLE updates (old_value TEXT, new_value TEXT) STRICT;
            CREATE TRIGGER record_change AFTER UPDATE OF value ON preferences
            BEGIN
                INSERT INTO updates VALUES (old.value, new.value);
            END;
        """)

        changed = [
            store_preference(connection, "theme", value)
            for value in (None, None, "dark", "dark", None, None)
        ]
        audit = cursor.execute("SELECT * FROM updates ORDER BY rowid").fetchall()
        stored = cursor.execute("SELECT value FROM preferences").fetchone()

assert (changed, audit, stored) == (
    [True, False, True, False, True, False],
    [(None, "dark"), ("dark", None)],
    (None,),
)
```

## Trade-offs and Limitations

This prevents the conflicting row's update, not all side effects of the
insert attempt. SQLite may still take write locks or run a `BEFORE INSERT`
trigger. A returned row indicates a direct statement change, not a commit
or a count of trigger effects.

The predicate compares the values using SQLite's comparison rules. It is
not an optimistic-lock check: a different incoming value overwrites the
existing value even if a caller has stale information.

## Related Snippets

<!-- catalog:related:start -->
- [Compare Direct and Triggered SQLite Row Changes for One Write](compare-direct-and-triggered-sqlite-row-changes-for-one-write.md)
- [Enforce One Active SQLite Row per Key with a Partial Unique Index](enforce-one-active-sqlite-row-per-key-with-a-partial-unique-index.md)
<!-- catalog:related:end -->
