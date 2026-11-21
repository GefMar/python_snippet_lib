---
title: "Recognize a Stale SQLite WAL Write Upgrade by Its Extended Error Code"
snippet_type: recipe
use_cases:
  - concurrency-control
  - persistence
  - retry-recovery
tested_python:
  - "3.14"
dependencies: []
related:
  - compare-and-swap-a-versioned-sqlite-setting-with-one-conditional-update.md
  - hold-one-sqlite-wal-read-snapshot-across-multiple-queries.md
---

# Recognize a Stale SQLite WAL Write Upgrade by Its Extended Error Code

## Idea and Problem

Recognize when a SQLite transaction is too old to become a writer and needs a new snapshot.

In WAL mode, another connection can commit after a deferred transaction
reads. A subsequent write from the old snapshot can fail with
`SQLITE_BUSY_SNAPSHOT`. Compare the exact extended result code. The error
text and the primary `SQLITE_BUSY` code do not identify this case.

## When to Use

Use this for a read-then-write operation on a local WAL database when the
caller owns the entire transaction. Roll back the failed attempt, then
repeat the reads and recompute the change in a fresh transaction. The
example uses explicit SQL transaction control with `autocommit=True`.

## Implementation

```python
import sqlite3


def is_stale_wal_snapshot(error: sqlite3.Error) -> bool:
    return getattr(error, "sqlite_errorcode", None) == sqlite3.SQLITE_BUSY_SNAPSHOT
```

## Example

```python
from contextlib import closing
from pathlib import Path
from tempfile import TemporaryDirectory

read_stock = "SELECT available FROM stock WHERE item = ?"
write_stock = "UPDATE stock SET available = ? WHERE item = ?"
item = "notebook"

with TemporaryDirectory() as directory:
    database = Path(directory) / "inventory.sqlite"
    with (
        closing(sqlite3.connect(database, autocommit=True, timeout=0)) as other,
        closing(sqlite3.connect(database, autocommit=True, timeout=0)) as client,
        closing(other.cursor()) as concurrent,
        closing(client.cursor()) as attempt,
    ):
        if concurrent.execute("PRAGMA journal_mode = WAL").fetchone() != ("wal",):
            raise RuntimeError("the fixture needs WAL mode")
        concurrent.execute("CREATE TABLE stock (item TEXT PRIMARY KEY, available INTEGER)")
        concurrent.execute("INSERT INTO stock VALUES (?, ?)", (item, 5))

        attempt.execute("BEGIN")
        (old_count,) = attempt.execute(read_stock, (item,)).fetchone()
        concurrent.execute(write_stock, (8, item))
        try:
            attempt.execute(write_stock, (old_count - 1, item))
        except sqlite3.OperationalError as error:
            if not is_stale_wal_snapshot(error):
                raise
            attempt.execute("ROLLBACK")
        else:
            raise RuntimeError("the fixture did not reproduce a stale snapshot")
        released = not client.in_transaction

        attempt.execute("BEGIN")
        (new_count,) = attempt.execute(read_stock, (item,)).fetchone()
        attempt.execute(write_stock, (new_count - 1, item))
        attempt.execute("COMMIT")
        stored = concurrent.execute(read_stock, (item,)).fetchone()

assert (old_count, released, new_count, stored) == (5, True, 8, (7,))
```

## Trade-offs and Limitations

Waiting or reissuing the same write inside the failed transaction does not
make its snapshot current. The example makes one fresh attempt; a second
failure propagates, and the owned connection closes without committing.
Only replay work whose earlier effects were transactional or are safe to
repeat. With `autocommit=True`, use SQL `ROLLBACK`, not the connection's
`rollback()` method, for an explicitly started transaction.

If the operation is definitely a write, `BEGIN IMMEDIATE` can acquire the
write transaction up front and avoid this upgrade. It can itself fail with
a busy error and holds up other writers until the transaction ends.

## Related Snippets

<!-- catalog:related:start -->
- [Compare and Swap a Versioned SQLite Setting with One Conditional Update](compare-and-swap-a-versioned-sqlite-setting-with-one-conditional-update.md)
- [Hold One SQLite WAL Read Snapshot Across Multiple Queries](hold-one-sqlite-wal-read-snapshot-across-multiple-queries.md)
<!-- catalog:related:end -->
