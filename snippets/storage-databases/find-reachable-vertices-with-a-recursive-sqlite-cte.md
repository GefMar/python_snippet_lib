---
title: "Find Reachable Vertices with a Recursive SQLite CTE"
snippet_type: recipe
use_cases:
  - data-transformation
  - persistence
tested_python:
  - "3.14"
dependencies: []
related:
  - ../algorithms-data-structures/traverse-a-parent-graph-with-breadth-first-search.md
  - ../algorithms-data-structures/compute-the-reflexive-transitive-closure-of-a-bounded-directed-graph-with-integer-bitsets.md
---

# Find Reachable Vertices with a Recursive SQLite CTE

## Idea and Problem

Find the vertices reachable from one start in a directed graph stored in SQLite, including when the graph contains cycles.

Seed a recursive common table expression with the requested vertex and follow
outgoing edges. Each recursive row contains only the vertex ID. With `UNION`,
SQLite discards repeats before adding them to the traversal queue. This lets
shared destinations and cycles produce one result per reachable vertex.

## When to Use

Use this for a small, trusted graph whose `main` tables hold integer vertex
IDs and edges between existing vertices. Index the edge table by source ID
to support the recursive lookup. Use a caller-owned `sqlite3` connection with
default column conversion.

The result includes a present start vertex through the path of length zero.
An isolated vertex therefore returns its own ID, while a missing start
returns an empty tuple.

## Implementation

```python
import sqlite3
from contextlib import closing

_REACHABLE = """
WITH RECURSIVE reached(id) AS (
    SELECT id FROM main.vertices WHERE id = ?
    UNION
    SELECT edge.target_id
    FROM reached JOIN main.edges AS edge ON edge.source_id = reached.id
)
SELECT id FROM reached ORDER BY id
"""


def reachable_vertices(connection: sqlite3.Connection, start_id: int) -> tuple[int, ...]:
    if type(start_id) is not int:
        raise TypeError("start_id must be an integer")

    with closing(connection.cursor()) as cursor:
        cursor.row_factory = None
        cursor.execute(_REACHABLE, (start_id,))
        return tuple(vertex_id for (vertex_id,) in cursor)
```

## Example

```python
with closing(sqlite3.connect(":memory:", autocommit=True)) as connection:
    with closing(connection.cursor()) as cursor:
        cursor.executescript("""
            PRAGMA foreign_keys = ON;
            CREATE TABLE vertices (id INTEGER PRIMARY KEY);
            CREATE TABLE edges (
                source_id INTEGER NOT NULL REFERENCES vertices(id),
                target_id INTEGER NOT NULL REFERENCES vertices(id),
                PRIMARY KEY (source_id, target_id)
            ) WITHOUT ROWID;
            INSERT INTO vertices VALUES (1), (2), (3), (4), (5), (6), (9);
        """)
        cursor.executemany(
            "INSERT INTO edges VALUES (?, ?)",
            [(1, 2), (1, 3), (2, 4), (3, 4), (4, 1), (4, 5), (5, 5), (9, 1)],
        )

    observed = tuple(reachable_vertices(connection, start_id) for start_id in (1, 5, 6, 9, 8))

assert observed == ((1, 2, 3, 4, 5), (5,), (6,), (1, 2, 3, 4, 5, 9), ())
```

## Trade-offs and Limitations

SQLite needs to remember seen IDs for `UNION`, and the outer query sorts the
complete result. Keep the reachable subgraph small. The ID order is for
deterministic presentation. Distances and traversal order require a different
approach. Adding a changing depth or path to the recursive row can make a
revisited vertex appear new and defeat cycle protection.

The query follows edges only from source to target and assumes valid graph
references. Vertex IDs must fit SQLite's signed integer range. A large graph
or a query with a runtime budget needs a separate resource policy.

## Related Snippets

<!-- catalog:related:start -->
- [Traverse a Parent Graph with Breadth-First Search](../algorithms-data-structures/traverse-a-parent-graph-with-breadth-first-search.md)
- [Compute the Reflexive Transitive Closure of a Bounded Directed Graph with Integer Bitsets](../algorithms-data-structures/compute-the-reflexive-transitive-closure-of-a-bounded-directed-graph-with-integer-bitsets.md)
<!-- catalog:related:end -->
