---
title: "Enforce One Active SQLite Row per Key with a Partial Unique Index"
snippet_type: pattern
use_cases:
  - persistence
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compare-and-swap-a-versioned-sqlite-setting-with-one-conditional-update.md
  - scope-caller-owned-sqlite-work-with-an-explicit-savepoint.md
---

# Enforce One Active SQLite Row per Key with a Partial Unique Index

## Idea and Problem

Keep an assignment history while letting SQLite enforce at most one active assignment per resource.

A full unique index on the resource would also reject historical rows. A
partial unique index includes only rows with `retired_at IS NULL`, so retiring
a row frees the resource for a new assignment. The index enforces the rule for
all writers, including writers that skip an application-level availability
check.

## When to Use

Use this pattern for a SQLite table where each row belongs to a non-null key
and one explicit row predicate defines whether it is active. Use a connection
with default result converters and a linked SQLite version of at least 3.35
for `RETURNING`. The caller controls commits and rollbacks.

## Implementation

```python
import sqlite3
from contextlib import closing

ASSIGNMENT_SCHEMA = """
CREATE TABLE main.assignments (
    id INTEGER PRIMARY KEY,
    resource TEXT NOT NULL,
    assignee TEXT NOT NULL CHECK (assignee <> ''),
    retired_at INTEGER
);
CREATE UNIQUE INDEX main.assignments_one_active
    ON assignments(resource) WHERE retired_at IS NULL;
"""


def try_assign(connection: sqlite3.Connection, resource: str, assignee: str) -> int | None:
    """Return the inserted row ID, or None when the resource is already active."""
    with closing(connection.cursor()) as cursor:
        cursor.row_factory = None
        cursor.execute(
            """
            INSERT INTO main.assignments (resource, assignee)
            VALUES (?, ?)
            ON CONFLICT (resource) WHERE retired_at IS NULL DO NOTHING
            RETURNING id
            """,
            (resource, assignee),
        )
        row = cursor.fetchone()
    return None if row is None else row[0]
```

## Example

```python
with closing(sqlite3.connect(":memory:", autocommit=False)) as connection:
    with closing(connection.cursor()) as cursor:
        cursor.executescript(ASSIGNMENT_SCHEMA)
    connection.commit()

    with connection, closing(connection.cursor()) as cursor:
        first_id = try_assign(connection, "room-a", "Robin")
        duplicate_id = try_assign(connection, "room-a", "Taylor")
        other_id = try_assign(connection, "room-b", "Taylor")
        assert first_id is not None and other_id is not None

        try:
            try_assign(connection, "room-a", "")
        except sqlite3.IntegrityError:
            invalid_assignee_rejected = True
        else:
            invalid_assignee_rejected = False

        cursor.execute(
            "UPDATE main.assignments SET retired_at = ? WHERE id = ?",
            (10, first_id),
        )
        second_id = try_assign(connection, "room-a", "Sam")
        assert second_id is not None
        cursor.execute(
            "UPDATE main.assignments SET retired_at = ? WHERE id = ?",
            (20, second_id),
        )
        current_id = try_assign(connection, "room-a", "Alex")
        assert current_id is not None

        history = cursor.execute(
            """
            SELECT id, assignee, retired_at FROM main.assignments
            WHERE resource = ? ORDER BY id
            """,
            ("room-a",),
        ).fetchall()

assert (duplicate_id, invalid_assignee_rejected, history) == (
    None,
    True,
    [(first_id, "Robin", 10), (second_id, "Sam", 20), (current_id, "Alex", None)],
)
```

## Trade-offs and Limitations

The conflict target must match the index, so change the indexed predicate and
the `ON CONFLICT` predicate together. Only the specified uniqueness conflict
becomes `None`. Other constraint errors and database busy errors can still
propagate. A returned ID identifies the insertion in the current transaction.
Its permanence depends on the caller's commit.

The index allows a resource to have zero active rows. It does not model future
reservations or prevent overlap between historical time ranges. Keep the
retirement and replacement in one caller-owned transaction when they must
succeed or roll back together. Use a normal unique constraint when the key
should be unique across the whole table.

## Related Snippets

<!-- catalog:related:start -->
- [Compare and Swap a Versioned SQLite Setting with One Conditional Update](compare-and-swap-a-versioned-sqlite-setting-with-one-conditional-update.md)
- [Scope Caller-Owned SQLite Work with an Explicit Savepoint](scope-caller-owned-sqlite-work-with-an-explicit-savepoint.md)
<!-- catalog:related:end -->
