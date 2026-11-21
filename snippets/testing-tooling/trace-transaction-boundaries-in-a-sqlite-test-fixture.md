---
title: "Trace Transaction Boundaries in a SQLite Test Fixture"
snippet_type: testing-technique
use_cases:
  - persistence
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - ../storage-databases/scope-caller-owned-sqlite-work-with-an-explicit-savepoint.md
  - ../storage-databases/commit-one-sqlite-item-mutation-and-its-checkpoint-atomically.md
---

# Trace Transaction Boundaries in a SQLite Test Fixture

## Idea and Problem

Observe the transaction statements a SQLite connection executes and verify the resulting rows in the same test.

The trace callback sees statements emitted by the driver as well as those
sent by the test. A small whitelist keeps only the transaction commands
expected from this fixture. The trace can explain why a transaction test
failed, while a query after the action checks which rows remain.

## When to Use

Use this with a dedicated, short-lived connection and synthetic data. Reserve
the connection's empty trace callback slot for one scope and keep the
connection open until the scope ends. The example selects `autocommit=False`,
so a commit or rollback starts another transaction. Tests with a different
transaction mode need different trace expectations.

## Implementation

```python
import sqlite3
from collections.abc import Iterator
from contextlib import contextmanager


@contextmanager
def trace_transaction_boundaries(connection: sqlite3.Connection) -> Iterator[list[str]]:
    events: list[str] = []

    def collect(statement: str) -> None:
        command = statement.strip().removesuffix(";").strip().upper()
        if command in {"BEGIN", "COMMIT", "ROLLBACK"}:
            events.append(command)

    connection.set_trace_callback(collect)
    try:
        yield events
    finally:
        connection.set_trace_callback(None)
```

## Example

```python
from contextlib import closing

with closing(sqlite3.connect(":memory:", autocommit=False)) as connection:
    with closing(connection.cursor()) as cursor:
        cursor.execute("CREATE TABLE notes (label TEXT PRIMARY KEY)")
        connection.commit()

        duplicate_rejected = False
        try:
            with trace_transaction_boundaries(connection) as events:
                with connection:
                    cursor.execute("INSERT INTO notes VALUES (?)", ("saved",))
                with connection:
                    cursor.execute("INSERT INTO notes VALUES (?)", ("discarded",))
                    cursor.execute("INSERT INTO notes VALUES (?)", ("saved",))
        except sqlite3.IntegrityError:
            duplicate_rejected = True

        # This commit would add more events if the callback were still installed.
        connection.commit()
        remaining = cursor.execute("SELECT label FROM notes ORDER BY label").fetchall()

assert (duplicate_rejected, events, remaining) == (
    True,
    ["COMMIT", "BEGIN", "ROLLBACK", "BEGIN"],
    [("saved",)],
)
```

## Trade-offs and Limitations

Installing a callback replaces any existing one. The scope clears the slot on
exit, so use an uninstrumented fixture and avoid nesting scopes. The whitelist
recognizes the shown spellings, with optional whitespace and a trailing
semicolon. It does not parse SQL or track savepoints. Trigger execution can
produce extra or repeated trace events, so avoid treating a general trace as
a one-to-one list of calls made by Python.

SQLite can expand bound values in trace SQL before the callback receives it.
Use synthetic data even when the collector retains only command names.
Callback exceptions do not propagate to the caller, so assert outside the
callback. Use the trace as a test diagnostic. It does not prove a commit
succeeded or provide an audit log.

## Related Snippets

<!-- catalog:related:start -->
- [Scope Caller-Owned SQLite Work with an Explicit Savepoint](../storage-databases/scope-caller-owned-sqlite-work-with-an-explicit-savepoint.md)
- [Commit One SQLite Item Mutation and Its Checkpoint Atomically](../storage-databases/commit-one-sqlite-item-mutation-and-its-checkpoint-atomically.md)
<!-- catalog:related:end -->
