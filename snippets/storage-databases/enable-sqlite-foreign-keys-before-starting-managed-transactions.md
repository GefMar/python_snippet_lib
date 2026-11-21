---
title: "Enable SQLite Foreign Keys Before Starting Managed Transactions"
snippet_type: recipe
use_cases:
  - persistence
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - open-a-verified-read-only-sqlite-connection-under-a-closed-hardening-profile.md
  - scope-caller-owned-sqlite-work-with-an-explicit-savepoint.md
---

# Enable SQLite Foreign Keys Before Starting Managed Transactions

## Idea and Problem

Enable and verify SQLite foreign key enforcement before handing a new connection to code that uses managed transactions.

SQLite ignores attempts to change `PRAGMA foreign_keys` during a transaction.
In Python, opening a connection with `autocommit=False` immediately starts
one. Open with `autocommit=True`, set and verify enforcement, then switch to
`autocommit=False` for application work.

## When to Use

Use this when opening a new `sqlite3` connection to tables with foreign key
references. Configure each connection at creation, including connections
opened by a pool. The scope owns the connection, while the caller decides
what to commit or roll back inside it.

## Implementation

```python
import sqlite3
from collections.abc import Iterator
from contextlib import closing, contextmanager
from pathlib import Path


@contextmanager
def connect_with_foreign_keys(database: str | Path) -> Iterator[sqlite3.Connection]:
    with closing(sqlite3.connect(database, autocommit=True)) as connection:
        with closing(connection.cursor()) as cursor:
            cursor.execute("PRAGMA foreign_keys = ON")
            if cursor.execute("PRAGMA foreign_keys").fetchone() != (1,):
                raise RuntimeError("SQLite foreign key enforcement is unavailable")

        connection.autocommit = False
        yield connection
```

## Example

```python
with closing(sqlite3.connect(":memory:", autocommit=True)) as control:
    with closing(control.cursor()) as cursor:
        cursor.execute("PRAGMA foreign_keys = OFF")
        control.autocommit = False
        assert control.in_transaction
        cursor.execute("PRAGMA foreign_keys = ON")
        late_setting = cursor.execute("PRAGMA foreign_keys").fetchone()

with connect_with_foreign_keys(":memory:") as connection:
    with closing(connection.cursor()) as cursor:
        with connection:
            cursor.executescript("""
                CREATE TABLE shelves (id INTEGER PRIMARY KEY);
                CREATE TABLE books (shelf_id INTEGER NOT NULL REFERENCES shelves(id));
            """)
            cursor.execute("INSERT INTO shelves VALUES (?)", (7,))
            cursor.execute("INSERT INTO books VALUES (?)", (7,))

        rejected_orphan = False
        try:
            with connection:
                cursor.execute("INSERT INTO books VALUES (?)", (99,))
        except sqlite3.IntegrityError as error:
            rejected_orphan = error.sqlite_errorcode == sqlite3.SQLITE_CONSTRAINT_FOREIGNKEY

        valid_references = cursor.execute("SELECT shelf_id FROM books").fetchall()

assert (late_setting, rejected_orphan, valid_references) == ((0,), True, [(7,)])
```

## Trade-offs and Limitations

Enforcement is a connection setting. Enabling it affects subsequent writes,
while existing inconsistent rows require a separate `PRAGMA foreign_key_check`
and a repair decision. The recipe fails at startup if it cannot verify the
setting for the linked SQLite build.

The outer scope closes the connection, leaving pending work to roll back when
it closes. Use `with connection` around each unit that should commit on
success or roll back on failure. Deferred foreign keys can fail at commit
instead of during the statement. Handle integrity errors around the entire
unit of work in that case.

## Related Snippets

<!-- catalog:related:start -->
- [Open a Verified Read-Only SQLite Connection Under a Closed Hardening Profile](open-a-verified-read-only-sqlite-connection-under-a-closed-hardening-profile.md)
- [Scope Caller-Owned SQLite Work with an Explicit Savepoint](scope-caller-owned-sqlite-work-with-an-explicit-savepoint.md)
<!-- catalog:related:end -->
