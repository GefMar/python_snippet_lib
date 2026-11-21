---
title: "Compare Direct and Triggered SQLite Row Changes for One Write"
snippet_type: testing-technique
use_cases:
  - observability
  - persistence
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - compare-and-swap-a-versioned-sqlite-setting-with-one-conditional-update.md
  - ../testing-tooling/trace-transaction-boundaries-in-a-sqlite-test-fixture.md
---

# Compare Direct and Triggered SQLite Row Changes for One Write

## Idea and Problem

Compare the rows touched by one SQLite update with the connection's broader change count to see the effects of a known trigger.

`cursor.rowcount` for this completed update counts the rows directly touched
by the statement. The increase in `connection.total_changes` also includes
the inserts made by its trigger. Sample the connection counter immediately
before and after the statement to exclude earlier writes.

## When to Use

Use this in a test or diagnostic for one known, successful table update on an
exclusively used SQLite connection. Keep `PRAGMA count_changes` disabled and
use an update without `RETURNING`, so the statement completes when `execute`
returns. The caller decides when to commit or roll back.

## Implementation

```python
import sqlite3
from contextlib import closing


def measure_state_update(
    connection: sqlite3.Connection, *, expected: str, replacement: str
) -> tuple[int, int]:
    """Return the direct row count and the connection's change delta."""
    with closing(connection.cursor()) as cursor:
        before = connection.total_changes
        cursor.execute(
            "UPDATE work_items SET state = ? WHERE state = ?",
            (replacement, expected),
        )
        return cursor.rowcount, connection.total_changes - before
```

## Example

```python
with closing(sqlite3.connect(":memory:", autocommit=False)) as connection:
    with connection, closing(connection.cursor()) as cursor:
        cursor.executescript("""
            CREATE TABLE work_items (id INTEGER PRIMARY KEY, state TEXT NOT NULL);
            CREATE TABLE state_events (item_id INTEGER NOT NULL);
            CREATE TRIGGER record_state AFTER UPDATE OF state ON work_items
            BEGIN
                INSERT INTO state_events VALUES (new.id);
            END;
            INSERT INTO work_items VALUES (1, 'queued'), (2, 'queued'), (3, 'done');
        """)

    before = connection.total_changes
    counts = measure_state_update(connection, expected="queued", replacement="ready")
    with closing(connection.cursor()) as cursor:
        events = cursor.execute("SELECT item_id FROM state_events ORDER BY item_id").fetchall()
        connection.rollback()
        count_after_rollback = connection.total_changes - before
        remaining_events = cursor.execute("SELECT item_id FROM state_events").fetchall()
        states = cursor.execute("SELECT state FROM work_items ORDER BY id").fetchall()

assert (counts, events, count_after_rollback, remaining_events, states) == (
    (2, 4),
    [(1,), (2,)],
    4,
    [],
    [("queued",), ("queued",), ("done",)],
)
```

## Trade-offs and Limitations

The counts describe completed statement work, which can still be rolled back.
Check the transaction outcome and query the resulting rows when persistence
matters. A matched row can count toward an update even when the assigned
value equals its current value.

The difference counts trigger inserts in this schema because those are the
only extra writes. Foreign key actions can also contribute to the connection
counter. Changes through views and deletions performed by `REPLACE` conflict
handling have separate counting rules. Use a schema-specific diagnostic when
you need to attribute every change to its origin.

## Related Snippets

<!-- catalog:related:start -->
- [Compare and Swap a Versioned SQLite Setting with One Conditional Update](compare-and-swap-a-versioned-sqlite-setting-with-one-conditional-update.md)
- [Trace Transaction Boundaries in a SQLite Test Fixture](../testing-tooling/trace-transaction-boundaries-in-a-sqlite-test-fixture.md)
<!-- catalog:related:end -->
