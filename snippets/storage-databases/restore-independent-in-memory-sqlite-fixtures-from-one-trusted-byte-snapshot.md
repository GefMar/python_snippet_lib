---
title: "Restore Independent In-Memory SQLite Fixtures from One Trusted Byte Snapshot"
snippet_type: recipe
use_cases:
  - persistence
  - resource-management
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - clone-a-bounded-in-memory-sqlite-fixture-with-connection-backup.md
  - execute-a-trusted-sqlite-query-under-progress-callback-and-row-caps.md
---

# Restore Independent In-Memory SQLite Fixtures from One Trusted Byte Snapshot

## Idea and Problem

Build a SQLite fixture once, close its source connection, and restore independent copies for tests.

`Connection.serialize()` turns the committed fixture into bytes. Calling
`Connection.deserialize()` on a fresh connection gives that connection its
own in-memory database. Keep the original bytes for later tests instead of
keeping the source connection open.

## When to Use

Use this for a small, trusted in-memory test fixture when setup is repeated
but each test must be free to write its own data. Serialize an owned
connection after committing fixture setup.

## Implementation

```python
import sqlite3
from collections.abc import Iterator
from contextlib import closing, contextmanager


@contextmanager
def restored_fixture(snapshot: bytes) -> Iterator[sqlite3.Connection]:
    with closing(sqlite3.connect(":memory:")) as connection:
        connection.deserialize(snapshot)
        yield connection
```

## Example

```python
with closing(sqlite3.connect(":memory:")) as source:
    source.execute("CREATE TABLE fixture (label TEXT NOT NULL)")
    source.execute("INSERT INTO fixture VALUES (?)", ("original",))
    source.commit()
    snapshot = source.serialize()

with restored_fixture(snapshot) as first, restored_fixture(snapshot) as second:
    first.execute("UPDATE fixture SET label = ?", ("changed",))
    first.commit()
    changed_rows = first.execute("SELECT label FROM fixture").fetchall()
    independent_rows = second.execute("SELECT label FROM fixture").fetchall()

with restored_fixture(snapshot) as later:
    restored_rows = later.execute("SELECT label FROM fixture").fetchall()

assert (changed_rows, independent_rows, restored_rows) == (
    [("changed",)],
    [("original",)],
    [("original",)],
)
```

## Trade-offs and Limitations

Only use bytes from a fixture you created and trust. Restoration is not a
validation or redaction step. The snapshot and each restored database take
memory, so keep the fixture small. The Python build must link against
SQLite with both the serialize and deserialize APIs.

This copies the main database, not connection settings or attached
databases. The context manager owns each new connection and closes it on
exit, including when restoration or the test fails. Commit explicitly if
your test needs to observe a commit; this scope only manages closure.

## Related Snippets

<!-- catalog:related:start -->
- [Clone a Bounded In-Memory SQLite Fixture with Connection.backup](clone-a-bounded-in-memory-sqlite-fixture-with-connection-backup.md)
- [Execute a Trusted SQLite Query Under Progress-Callback and Row Caps](execute-a-trusted-sqlite-query-under-progress-callback-and-row-caps.md)
<!-- catalog:related:end -->
