---
title: "Commit One SQLite Item Mutation and Its Checkpoint Atomically"
snippet_type: pattern
use_cases:
  - persistence
  - retry-recovery
tested_python:
  - "3.14"
dependencies: []
related:
  - ../reliability-resilience/commit-a-source-checkpoint-only-after-the-sink-accepts-a-batch.md
  - compare-and-swap-a-versioned-sqlite-setting-with-one-conditional-update.md
  - ../reliability-resilience/reduce-bounded-acknowledgements-into-exactly-once-completions.md
---

# Commit One SQLite Item Mutation and Its Checkpoint Atomically

## Idea and Problem

Put one item's local database mutation and its processed checkpoint in the same SQLite transaction so neither can commit alone.

A marker written after a separate commit leaves a crash window in which a
successful item is processed again. A marker written first can suppress work
that never committed. Acquire SQLite's write reservation, check the marker,
run the trusted local mutation on that same connection, and insert the marker
before one commit. Retrying an equal item becomes a no-op, while a changed
operation version is an explicit conflict.

## When to Use

Use this pattern for resumable bounded work when both the output and checkpoint
live in one SQLite database. The caller must own an idle connection and provide
a short trusted callback whose only effects are SQL statements on that exact
connection. A stable operation version should change whenever replaying an
item would use meaningfully different mutation semantics.

Use an outbox, a remote idempotency key, or reconciliation when processing
also sends a request, writes another database, publishes a message, or changes
a file. SQLite cannot make those external effects part of this transaction.

## Implementation

```python
import re
import sqlite3
from collections.abc import Callable

_TOKEN = re.compile(r"[a-z][a-z0-9._:-]{0,63}", re.ASCII).fullmatch


class CheckpointConflictError(RuntimeError):
    pass


def _token(value: object, *, name: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{name} must be an exact string")
    if _TOKEN(value) is None:
        raise ValueError(f"{name} must be a 1-64 byte conservative ASCII token")
    return value


def initialize_checkpoint_table(connection: sqlite3.Connection) -> None:
    if not isinstance(connection, sqlite3.Connection):
        raise TypeError("connection must be a sqlite3.Connection")
    if connection.in_transaction:
        raise RuntimeError("connection must be idle")
    connection.execute(
        """
        CREATE TABLE IF NOT EXISTS item_checkpoints (
            item_id TEXT PRIMARY KEY,
            operation_version TEXT NOT NULL
        ) STRICT
        """
    )


def apply_sqlite_item_once(
    connection: sqlite3.Connection,
    *,
    item_id: str,
    operation_version: str,
    mutate: Callable[[sqlite3.Connection, str], None],
) -> bool:
    if not isinstance(connection, sqlite3.Connection):
        raise TypeError("connection must be a sqlite3.Connection")
    if connection.in_transaction:
        raise RuntimeError("connection must be idle")
    if not callable(mutate):
        raise TypeError("mutate must be callable")
    item_id = _token(item_id, name="item_id")
    operation_version = _token(
        operation_version,
        name="operation_version",
    )

    connection.execute("BEGIN IMMEDIATE")
    try:
        row = connection.execute(
            "SELECT operation_version FROM item_checkpoints WHERE item_id = ?",
            (item_id,),
        ).fetchone()
        if row is not None:
            if row[0] != operation_version:
                raise CheckpointConflictError("item was processed by a different operation version")
            connection.execute("COMMIT")
            return False

        mutate(connection, item_id)
        if not connection.in_transaction:
            raise RuntimeError("mutate ended the owned SQLite transaction")
        connection.execute(
            "INSERT INTO item_checkpoints(item_id, operation_version) VALUES (?, ?)",
            (item_id, operation_version),
        )
        connection.execute("COMMIT")
        return True
    except BaseException:
        if connection.in_transaction:
            connection.execute("ROLLBACK")
        raise
```

## Example

```python
connection = sqlite3.connect(":memory:", isolation_level=None)
initialize_checkpoint_table(connection)
connection.execute(
    "CREATE TABLE item_results (item_id TEXT PRIMARY KEY, value INTEGER NOT NULL) STRICT"
)


def store_square(database: sqlite3.Connection, item_id: str) -> None:
    number = int(item_id.removeprefix("unit-"))
    database.execute(
        "INSERT INTO item_results(item_id, value) VALUES (?, ?)",
        (item_id, number * number),
    )


first_commit = apply_sqlite_item_once(
    connection,
    item_id="unit-7",
    operation_version="square-v1",
    mutate=store_square,
)
replay_commit = apply_sqlite_item_once(
    connection,
    item_id="unit-7",
    operation_version="square-v1",
    mutate=store_square,
)

try:
    apply_sqlite_item_once(
        connection,
        item_id="unit-7",
        operation_version="square-v2",
        mutate=store_square,
    )
except CheckpointConflictError:
    version_conflict = True
else:
    version_conflict = False

connection.execute(
    """
    CREATE TRIGGER reject_one_checkpoint
    BEFORE INSERT ON item_checkpoints
    WHEN NEW.item_id = 'unit-9'
    BEGIN
        SELECT RAISE(ABORT, 'synthetic checkpoint failure');
    END
    """
)
try:
    apply_sqlite_item_once(
        connection,
        item_id="unit-9",
        operation_version="square-v1",
        mutate=store_square,
    )
except sqlite3.IntegrityError:
    rollback_observed = connection.execute(
        "SELECT value FROM item_results WHERE item_id = 'unit-9'"
    ).fetchone() is None
else:
    rollback_observed = False

stored = connection.execute(
    "SELECT item_id, value FROM item_results ORDER BY item_id"
).fetchall()
connection.close()

assert (first_commit, replay_commit, version_conflict, rollback_observed, stored) == (
    True,
    False,
    True,
    True,
    [("unit-7", 49)],
)
```

## Trade-offs and Limitations

`BEGIN IMMEDIATE` serializes writers early. That removes the marker race but
can reduce concurrency, and a busy database still follows the connection's
configured timeout. Keep the callback small, deterministic, and free of slow
queries. The function deliberately takes control of an idle connection; it
cannot be nested inside a caller transaction or savepoint.

The callback is trusted not to commit, roll back, attach another database, or
perform external effects. The post-callback transaction check catches an ended
transaction but cannot prove that every operation stayed inside SQLite. An
equal marker means this local operation version committed, not that an
external workflow ran exactly once or that its inputs can never change.

## Related Snippets

<!-- catalog:related:start -->
- [Commit a Source Checkpoint Only After the Sink Accepts a Batch](../reliability-resilience/commit-a-source-checkpoint-only-after-the-sink-accepts-a-batch.md)
- [Compare and Swap a Versioned SQLite Setting with One Conditional Update](compare-and-swap-a-versioned-sqlite-setting-with-one-conditional-update.md)
- [Reduce Bounded Acknowledgements into Exactly-Once Completions](../reliability-resilience/reduce-bounded-acknowledgements-into-exactly-once-completions.md)
<!-- catalog:related:end -->
