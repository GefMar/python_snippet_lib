---
title: "Match a Literal Prefix with SQLite LIKE and an Explicit Escape"
snippet_type: recipe
use_cases:
  - persistence
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - batch-a-sqlite-in-query-under-the-connection-bind-limit.md
  - ../algorithms-data-structures/find-the-matching-range-for-a-literal-prefix-in-sorted-strings.md
---

# Match a Literal Prefix with SQLite LIKE and an Explicit Escape

## Idea and Problem

Find labels that start with a supplied prefix without letting characters inside it act as wildcards.

Binding a parameter prevents the value from becoming SQL syntax, but SQLite
still interprets `%` and `_` as pattern operators inside `LIKE`. Escape the
escape character first, then escape both wildcards and append the one
intentional `%` for the suffix.

## When to Use

Use this for short, nonempty prefixes against non-null text labels without NUL
characters. The connection must use the built-in default `LIKE` behavior
and default text conversion. The caller owns the connection and transaction.

## Implementation

```python
import sqlite3
from contextlib import closing


def find_labels_by_prefix(connection: sqlite3.Connection, prefix: str) -> tuple[str, ...]:
    if type(prefix) is not str:
        raise TypeError("prefix must be a string")
    if not prefix or "\x00" in prefix:
        raise ValueError("prefix must be nonempty and contain no NUL characters")

    pattern = prefix.replace("!", "!!").replace("%", "!%").replace("_", "!_") + "%"
    with closing(connection.cursor()) as cursor:
        cursor.row_factory = None
        cursor.execute(
            "SELECT label FROM main.labels WHERE label LIKE ? ESCAPE '!' "
            "ORDER BY label COLLATE BINARY",
            (pattern,),
        )
        return tuple(label for (label,) in cursor)
```

## Example

```python
with closing(sqlite3.connect(":memory:", autocommit=True)) as connection:
    with closing(connection.cursor()) as cursor:
        cursor.execute("""
            CREATE TABLE labels (
                label TEXT NOT NULL CHECK (instr(label, char(0)) = 0)
            ) STRICT
        """)
        cursor.executemany(
            "INSERT INTO labels VALUES (?)",
            [
                ("A!_% draft",),
                ("a!_%",),
                ("a!_x draft",),
                ("a!z% draft",),
                ("a_% draft",),
                ("Æ_ tag",),
                ("æ_ tag",),
            ],
        )

    matched = find_labels_by_prefix(connection, "a!_%")
    non_ascii = find_labels_by_prefix(connection, "æ_")

assert (matched, non_ascii) == (("A!_% draft", "a!_%"), ("æ_ tag",))
```

## Trade-offs and Limitations

Literal here means that the input contributes no pattern operators. Built-in
default SQLite `LIKE` folds ASCII case only; it is not a byte-for-byte
prefix test or Unicode caseless search. The binary ordering above affects
output order, not matching.

The function collects every match and does not guarantee an indexed plan.
For a large table or broad search, verify the query plan and bound the result
size. Limit user-supplied prefix length as well: the escaped pattern must
fit SQLite's pattern length limit.

## Related Snippets

<!-- catalog:related:start -->
- [Batch a SQLite IN Query Under the Connection Bind Limit](batch-a-sqlite-in-query-under-the-connection-bind-limit.md)
- [Find the Matching Range for a Literal Prefix in Sorted Strings](../algorithms-data-structures/find-the-matching-range-for-a-literal-prefix-in-sorted-strings.md)
<!-- catalog:related:end -->
