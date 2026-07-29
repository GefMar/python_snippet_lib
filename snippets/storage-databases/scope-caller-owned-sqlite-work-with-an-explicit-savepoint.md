---
title: "Scope Caller-Owned SQLite Work with an Explicit Savepoint"
snippet_type: pattern
use_cases:
  - persistence
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - commit-one-sqlite-item-mutation-and-its-checkpoint-atomically.md
  - compare-and-swap-a-versioned-sqlite-setting-with-one-conditional-update.md
  - replace-a-complete-bounded-sqlite-cache-snapshot-with-a-generation.md
---

# Scope Caller-Owned SQLite Work with an Explicit Savepoint

## Idea and Problem

Let a helper roll back only its own SQLite work while leaving the caller in control of an already active outer transaction.

A private savepoint marks the helper's starting state. Normal exit releases
that mark into the outer transaction. If the body raises, `ROLLBACK TO` rewinds
changes made after the mark, and `RELEASE` then removes the still-active
savepoint before the same body exception is re-raised.

Requiring `connection.in_transaction` before issuing `SAVEPOINT` prevents this
scope from becoming the outermost transaction and committing when released.

## When to Use

Use this pattern for a synchronous helper that shares one exclusively owned
`sqlite3.Connection` with caller-managed transaction work. The body may execute
ordinary bounded statements, and nested uses of this same helper are supported.

Use an ordinary connection transaction when the helper owns the complete unit
of work. Use a database abstraction when transaction propagation, async access,
cross-database coordination, retry policy, or connection pooling must be
managed consistently across many operations.

## Implementation

```python
import sqlite3
from collections.abc import Iterator
from contextlib import contextmanager

_SAVEPOINT = '"snippet_scope"'


@contextmanager
def caller_owned_sqlite_savepoint(
    connection: sqlite3.Connection,
) -> Iterator[None]:
    """Scope work inside an existing caller-owned SQLite transaction."""
    if not isinstance(connection, sqlite3.Connection):
        raise TypeError("connection must be a sqlite3.Connection")
    if not connection.in_transaction:
        raise RuntimeError("an active outer transaction is required")

    connection.execute(f"SAVEPOINT {_SAVEPOINT}")
    try:
        yield None
    except BaseException:
        connection.execute(f"ROLLBACK TO SAVEPOINT {_SAVEPOINT}")
        connection.execute(f"RELEASE SAVEPOINT {_SAVEPOINT}")
        raise
    else:
        connection.execute(f"RELEASE SAVEPOINT {_SAVEPOINT}")
```

## Example

```python
connection = sqlite3.connect(":memory:", isolation_level=None)
connection.execute("CREATE TABLE items (item_id INTEGER PRIMARY KEY, label TEXT NOT NULL)")

statements_without_outer: list[str] = []
connection.set_trace_callback(statements_without_outer.append)
try:
    with caller_owned_sqlite_savepoint(connection):
        pass
except RuntimeError:
    missing_outer_rejected = True
else:
    missing_outer_rejected = False
connection.set_trace_callback(None)

connection.execute("BEGIN")
connection.execute("INSERT INTO items VALUES (1, 'outer')")
with caller_owned_sqlite_savepoint(connection) as yielded:
    connection.execute("INSERT INTO items VALUES (2, 'first scope')")
    with caller_owned_sqlite_savepoint(connection):
        connection.execute("INSERT INTO items VALUES (3, 'nested scope')")

rows_before_outer_rollback = connection.execute(
    "SELECT item_id FROM items ORDER BY item_id"
).fetchall()
active_after_success = connection.in_transaction
connection.execute("ROLLBACK")
rows_after_outer_rollback = connection.execute("SELECT item_id FROM items").fetchall()

connection.execute("BEGIN")
connection.execute("INSERT INTO items VALUES (10, 'outer survives')")
try:
    with caller_owned_sqlite_savepoint(connection):
        connection.execute("INSERT INTO items VALUES (20, 'rolled back')")
        connection.execute("INSERT INTO items VALUES (10, 'duplicate')")
except sqlite3.IntegrityError:
    constraint_re_raised = True
else:
    constraint_re_raised = False

rows_after_inner_rollback = connection.execute(
    "SELECT item_id FROM items ORDER BY item_id"
).fetchall()
active_after_failure = connection.in_transaction
connection.execute("COMMIT")
connection.close()

assert missing_outer_rejected
assert not any("SAVEPOINT" in statement for statement in statements_without_outer)
assert yielded is None
assert rows_before_outer_rollback == [(1,), (2,), (3,)]
assert active_after_success
assert rows_after_outer_rollback == []
assert constraint_re_raised
assert rows_after_inner_rollback == [(10,)]
assert active_after_failure
```

## Trade-offs and Limitations

Entering and leaving adds a constant number of SQL statements and Python state
operations. The cost of `ROLLBACK TO` depends on writes performed since the
savepoint. A successful `RELEASE` only erases the inner mark: its changes can
still be undone by an outer rollback and are not durable until the outer
transaction commits.

SQLite resolves a repeated savepoint name from the most recently created mark,
which permits properly nested calls to this helper. The body must not issue its
own transaction-control or savepoint SQL, call `executescript()`, close the
connection, or transfer it to concurrent work. Any of those actions can
invalidate the helper's cleanup assumptions.

If `ROLLBACK TO` or `RELEASE` itself fails, that database error propagates with
the body error chained and transaction ownership may require caller recovery.
The helper catches cancellation-like `BaseException` subclasses for cleanup,
but it provides no async integration, timeout, retry, commit, or connection
rollback policy.

## Related Snippets

<!-- catalog:related:start -->
- [Commit One SQLite Item Mutation and Its Checkpoint Atomically](commit-one-sqlite-item-mutation-and-its-checkpoint-atomically.md)
- [Compare and Swap a Versioned SQLite Setting with One Conditional Update](compare-and-swap-a-versioned-sqlite-setting-with-one-conditional-update.md)
- [Replace a Complete Bounded SQLite Cache Snapshot with a Generation](replace-a-complete-bounded-sqlite-cache-snapshot-with-a-generation.md)
<!-- catalog:related:end -->
