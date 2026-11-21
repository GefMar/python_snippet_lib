---
title: "Compare SQLite Row Multiplicities with a Signed Union"
snippet_type: recipe
use_cases:
  - validation
  - persistence
tested_python:
  - "3.14"
dependencies: []
related:
  - ../data-processing/stream-count-deltas-between-two-sorted-inventories.md
  - compare-direct-and-triggered-sqlite-row-changes-for-one-write.md
---

# Compare SQLite Row Multiplicities with a Signed Union

## Idea and Problem

Find which rows occur more or less often in two SQLite tables even when neither side has unique rows.

Preserve every row with a signed union, then sum by the compared columns and skip groups whose count difference is zero. SQL grouping puts null values into the same group without treating them as wildcard matches.

## When to Use

Use this for two named tables with the same text column contract, where text compares by binary collation and note may be null. A positive count means the row occurs more often in the observed table.

## Implementation

```python
import sqlite3
from contextlib import closing


def row_count_differences(
    connection: sqlite3.Connection,
) -> tuple[tuple[str, str | None, int], ...]:
    with closing(connection.cursor()) as cursor:
        cursor.row_factory = None
        return tuple(
            cursor.execute("""
                SELECT code, note, SUM(delta)
                FROM (
                    SELECT code, note, 1 AS delta FROM main.observed
                    UNION ALL
                    SELECT code, note, -1 AS delta FROM main.expected
                )
                GROUP BY code COLLATE BINARY, note COLLATE BINARY
                HAVING SUM(delta) <> 0
                ORDER BY code COLLATE BINARY, note COLLATE BINARY
                """)
        )
```

## Example

```python
with closing(sqlite3.connect(":memory:")) as connection:
    connection.executescript("""
        CREATE TABLE observed (code TEXT NOT NULL, note TEXT) STRICT;
        CREATE TABLE expected (code TEXT NOT NULL, note TEXT) STRICT;
        INSERT INTO observed VALUES ('A', NULL), ('A', NULL), ('B', 'same');
        INSERT INTO expected VALUES ('A', NULL), ('B', 'same'), ('C', '');
    """)
    delta = row_count_differences(connection)

assert delta == (("A", None, 1), ("C", "", -1))
```

## Trade-offs and Limitations

This compares row multiplicity, not order, lineage, or which physical copy changed. It scans and groups both sources and materializes the resulting differences in Python. Keep the comparison within one connection and decide separately how to handle changes between different statements.

## Related Snippets

<!-- catalog:related:start -->
- [Stream Count Deltas Between Two Sorted Inventories](../data-processing/stream-count-deltas-between-two-sorted-inventories.md)
- [Compare Direct and Triggered SQLite Row Changes for One Write](compare-direct-and-triggered-sqlite-row-changes-for-one-write.md)
<!-- catalog:related:end -->
