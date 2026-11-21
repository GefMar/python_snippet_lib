---
title: "Enforce Normalized Label Uniqueness with a Deterministic SQLite Function"
snippet_type: pattern
use_cases:
  - persistence
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - enforce-one-active-sqlite-row-per-key-with-a-partial-unique-index.md
  - ../algorithms-data-structures/build-a-canonical-unicode-caseless-comparison-key.md
---

# Enforce Normalized Label Uniqueness with a Deterministic SQLite Function

## Idea and Problem

Keep the original spelling of a label while enforcing uniqueness under an explicit text normalization rule.

Register a pure normalization function with SQLite and build a unique index
on its result. The sample rule collapses whitespace and folds case. Use the
same expression in a lookup predicate so SQLite can use the index while
returning the original label.

## When to Use

Use this in a database whose schema you control and trust, with a connection
policy that already permits application-defined functions in index
expressions. Register the same function version on every connection that
uses the index or writes to the indexed table. The fixture uses a strict
table with a non-null text label.

## Implementation

```python
import sqlite3
from contextlib import closing

LABEL_INDEX_SQL = "CREATE UNIQUE INDEX labels_by_key ON labels(label_key_v1(label))"


def label_key_v1(label: str) -> str:
    return " ".join(label.split()).casefold()


def register_label_key(connection: sqlite3.Connection) -> None:
    connection.create_function("label_key_v1", 1, label_key_v1, deterministic=True)


def lookup_label(connection: sqlite3.Connection, candidate: str) -> str | None:
    with closing(connection.cursor()) as cursor:
        cursor.row_factory = None
        row = cursor.execute(
            "SELECT label FROM main.labels WHERE label_key_v1(label) = label_key_v1(?)",
            (candidate,),
        ).fetchone()
    return None if row is None else row[0]
```

## Example

```python
original_label = "Stra\u00dfe West"
with closing(sqlite3.connect(":memory:", autocommit=True)) as connection:
    with closing(connection.cursor()) as cursor:
        policy_before = cursor.execute("PRAGMA trusted_schema").fetchone()
        cursor.execute("CREATE TABLE labels (label TEXT NOT NULL) STRICT")

        connection.create_function("label_key_v1", 1, label_key_v1, deterministic=False)
        unmarked_index_rejected = False
        try:
            cursor.execute(LABEL_INDEX_SQL)
        except sqlite3.OperationalError:
            unmarked_index_rejected = True

        register_label_key(connection)
        cursor.execute(LABEL_INDEX_SQL)
        cursor.execute("INSERT INTO labels VALUES (?)", (original_label,))

        duplicate_rejected = False
        try:
            cursor.execute("INSERT INTO labels VALUES (?)", ("STRASSE   WEST",))
        except sqlite3.IntegrityError as error:
            duplicate_rejected = error.sqlite_errorcode == sqlite3.SQLITE_CONSTRAINT_UNIQUE

        matched = lookup_label(connection, " strasse\twest ")
        absent = lookup_label(connection, "other")
        policy_after = cursor.execute("PRAGMA trusted_schema").fetchone()

assert (unmarked_index_rejected, duplicate_rejected, matched, absent, policy_after) == (
    True,
    True,
    original_label,
    None,
    policy_before,
)
```

## Trade-offs and Limitations

The indexed key is part of the stored schema contract. The Python function
must keep returning the same key for inputs that were already indexed.
Changing the rule or the Unicode behavior of the Python runtime can change
keys. Coordinate a migration that checks for new collisions and rebuilds
the index before switching functions across connections.

The recipe keeps the existing trusted-schema setting. Use an ordinary
normalized key column when your deployment policy excludes Python
functions from schema expressions. Expect index creation to fail when
existing rows already collide under the chosen rule.

## Related Snippets

<!-- catalog:related:start -->
- [Enforce One Active SQLite Row per Key with a Partial Unique Index](enforce-one-active-sqlite-row-per-key-with-a-partial-unique-index.md)
- [Build a Canonical Unicode Caseless Comparison Key](../algorithms-data-structures/build-a-canonical-unicode-caseless-comparison-key.md)
<!-- catalog:related:end -->
