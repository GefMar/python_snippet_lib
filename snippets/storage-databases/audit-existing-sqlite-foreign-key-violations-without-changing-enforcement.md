---
title: "Audit Existing SQLite Foreign Key Violations Without Changing Enforcement"
snippet_type: testing-technique
use_cases:
  - persistence
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - enable-sqlite-foreign-keys-before-starting-managed-transactions.md
  - audit-one-sqlite-table-against-an-exact-ordered-column-contract.md
---

# Audit Existing SQLite Foreign Key Violations Without Changing Enforcement

## Idea and Problem

Find existing foreign key violations in SQLite while preserving the connection's enforcement setting.

Old data can contain invalid references even when the current connection has
foreign key enforcement enabled. `PRAGMA main.foreign_key_check` reports one
record for each failed reference. Keep the child table, its row ID when
available, the parent table, and the foreign key ID so the caller can
investigate the constraint.

## When to Use

Use this to inspect a small trusted database before a migration or after an
import. Pass a caller-owned connection with default text conversion. The
caller controls which database snapshot the audit sees and what action to
take on the results.

## Implementation

```python
import sqlite3
from contextlib import closing
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class ForeignKeyViolation:
    child_table: str
    child_rowid: int | None
    parent_table: str
    foreign_key_id: int


def audit_foreign_keys(connection: sqlite3.Connection) -> tuple[ForeignKeyViolation, ...]:
    with closing(connection.cursor()) as cursor:
        cursor.row_factory = None
        if cursor.execute("PRAGMA foreign_keys").fetchone() not in ((0,), (1,)):
            raise RuntimeError("SQLite foreign key support is unavailable")

        cursor.execute("PRAGMA main.foreign_key_check")
        return tuple(ForeignKeyViolation(*row) for row in cursor)
```

## Example

```python
with closing(sqlite3.connect(":memory:", autocommit=True)) as connection:
    with closing(connection.cursor()) as cursor:
        cursor.executescript("""
            PRAGMA foreign_keys = OFF;
            CREATE TABLE shelves (id INTEGER PRIMARY KEY);
            CREATE TABLE books (id INTEGER PRIMARY KEY, shelf_id INTEGER REFERENCES shelves(id));
            CREATE TABLE labels (
                code TEXT PRIMARY KEY,
                shelf_id INTEGER REFERENCES shelves(id)
            ) WITHOUT ROWID;
            INSERT INTO shelves VALUES (1);
            INSERT INTO books VALUES (7, 1), (8, 2), (9, NULL);
            INSERT INTO labels VALUES ('legacy', 2);
            PRAGMA foreign_keys = ON;
        """)
        before = (cursor.execute("PRAGMA foreign_keys").fetchone(), connection.total_changes)
        violations = sorted(audit_foreign_keys(connection), key=lambda row: row.child_table)
        after = (cursor.execute("PRAGMA foreign_keys").fetchone(), connection.total_changes)

assert (violations, before[0], after == before) == (
    [
        ForeignKeyViolation("books", 8, "shelves", 0),
        ForeignKeyViolation("labels", None, "shelves", 0),
    ],
    (1,),
    True,
)
```

## Trade-offs and Limitations

The foreign key ID matches the `id` column from `PRAGMA foreign_key_list` for
the child table. For a `WITHOUT ROWID` child, SQLite returns `NULL` in the row
ID field. Finding the particular child key then requires a query based on the
table's schema. Several violations can have identical evidence, so the
result keeps every record returned by SQLite.

The audit covers declared foreign keys in `main` and materializes all
reported violations. Large databases may need a table-scoped check or
streaming diagnostic. SQL errors propagate to the caller. Treat the report
as evidence for a separate repair decision.

## Related Snippets

<!-- catalog:related:start -->
- [Enable SQLite Foreign Keys Before Starting Managed Transactions](enable-sqlite-foreign-keys-before-starting-managed-transactions.md)
- [Audit One SQLite Table Against an Exact Ordered Column Contract](audit-one-sqlite-table-against-an-exact-ordered-column-contract.md)
<!-- catalog:related:end -->
