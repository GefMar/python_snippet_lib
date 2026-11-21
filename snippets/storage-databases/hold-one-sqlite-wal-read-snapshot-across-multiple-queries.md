---
title: "Hold One SQLite WAL Read Snapshot Across Multiple Queries"
snippet_type: pattern
use_cases:
  - persistence
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - refresh-a-sqlite-read-cache-after-external-commits.md
  - scope-caller-owned-sqlite-work-with-an-explicit-savepoint.md
---

# Hold One SQLite WAL Read Snapshot Across Multiple Queries

## Idea and Problem

Read several related results from a single SQLite snapshot while another connection continues to commit writes.

Two independent queries can see different committed states. Put them in one
explicit read transaction. In WAL mode, the first read of the database
establishes the snapshot. Subsequent reads in that transaction see the same
state even if a separate writer commits in the meantime.

## When to Use

Use this for a short report over the `main` database on a local SQLite file
already configured for WAL. Give the report its own idle, read-only connection
opened with `autocommit=True`. Use the yielded cursor for reads and consume
their rows inside the scope. Leave transaction control to the scope.

## Implementation

```python
import sqlite3
from collections.abc import Iterator
from contextlib import closing, contextmanager


@contextmanager
def wal_read_snapshot(reader: sqlite3.Connection) -> Iterator[sqlite3.Cursor]:
    if reader.autocommit is not True or reader.in_transaction:
        raise ValueError("use a dedicated autocommit reader outside a transaction")

    with closing(reader.cursor()) as cursor:
        cursor.execute("BEGIN")
        try:
            yield cursor
        finally:
            if reader.in_transaction:
                cursor.execute("ROLLBACK")
```

## Example

```python
from pathlib import Path
from tempfile import TemporaryDirectory

with TemporaryDirectory() as directory:
    path = Path(directory).resolve() / "samples.sqlite3"
    with closing(sqlite3.connect(path, autocommit=True)) as writer:
        with closing(writer.cursor()) as writes:
            mode = writes.execute("PRAGMA journal_mode = WAL").fetchone()
            if mode != ("wal",):
                raise RuntimeError("the fixture needs WAL mode")
            writes.execute("CREATE TABLE samples (value INTEGER NOT NULL)")
            writes.execute("INSERT INTO samples VALUES (?)", (4,))

            uri = f"{path.as_uri()}?mode=ro"
            with closing(sqlite3.connect(uri, uri=True, autocommit=True)) as reader:
                with wal_read_snapshot(reader) as reads:
                    count_before_commit = reads.execute("SELECT COUNT(*) FROM samples").fetchone()
                    writes.execute("INSERT INTO samples VALUES (?)", (6,))
                    sum_inside = reads.execute("SELECT SUM(value) FROM samples").fetchone()

                released = not reader.in_transaction
                with closing(reader.cursor()) as reads:
                    sum_outside = reads.execute("SELECT SUM(value) FROM samples").fetchone()

assert (count_before_commit, sum_inside, released, sum_outside) == (
    (1,),
    (4,),
    True,
    (10,),
)
```

## Trade-offs and Limitations

The results stay as old as the snapshot until the transaction ends. A long
read can delay WAL checkpoint progress, so finish the queries before doing
slow work with their results. Under rollback journaling, a reader can block
a writer's commit. The example verifies WAL to exercise the concurrent case.

With `autocommit=True`, use SQL `ROLLBACK` to end the explicitly started
transaction. The connection's `rollback()` method has no effect in that
mode. Keep the caller-owned connection open and exclusive to the report
throughout the scope.

## Related Snippets

<!-- catalog:related:start -->
- [Refresh a SQLite Read Cache After External Commits](refresh-a-sqlite-read-cache-after-external-commits.md)
- [Scope Caller-Owned SQLite Work with an Explicit Savepoint](scope-caller-owned-sqlite-work-with-an-explicit-savepoint.md)
<!-- catalog:related:end -->
