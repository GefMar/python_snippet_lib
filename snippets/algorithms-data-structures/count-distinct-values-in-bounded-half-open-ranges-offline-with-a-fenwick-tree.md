---
title: "Count Distinct Values in Bounded Half-Open Ranges Offline with a Fenwick Tree"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - maintain-bounded-point-adds-and-half-open-range-sums-with-a-fenwick-tree.md
  - answer-static-half-open-range-minimum-queries-with-a-sparse-table.md
  - count-strict-inversions-in-a-bounded-integer-sequence.md
---

# Count Distinct Values in Bounded Half-Open Ranges Offline with a Fenwick Tree

## Idea and Problem

Answer many static distinct-value range queries by sweeping each value once and keeping one active marker at its latest occurrence.

Sort indexed queries by their exclusive stop. As the sweep reaches that stop,
remove the marker at each value's previous position and add one at its current
position. A Fenwick prefix difference then counts values whose latest seen
occurrence lies inside the query, which is exactly the number of distinct
values in that half-open range.

The queries are reordered only for computation. Their original indexes place
every count back in declaration order, including repeated and empty queries.

## When to Use

Use this algorithm for one immutable integer sequence when all half-open range
queries are known together. It is useful when repeatedly constructing a set
from each slice would revisit too much data, while the full query batch fits in
memory and answers need to follow the original query order.

Use a simple `len(set(values[start:stop]))` calculation for a few short ranges.
Choose a dynamic range-query structure when values change between queries, or
a different offline technique when the aggregate depends on frequencies
rather than only on whether a value occurs.

## Implementation

```python
from typing import TypeAlias

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_DISTINCT_VALUES = 100_000
_MAX_DISTINCT_QUERIES = 100_000

HalfOpenQuery: TypeAlias = tuple[int, int]


def count_distinct_values_offline(
    values: tuple[int, ...],
    queries: tuple[HalfOpenQuery, ...],
) -> tuple[int, ...]:
    """Return distinct counts aligned with the declared query order."""
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    if len(values) > _MAX_DISTINCT_VALUES:
        raise ValueError("value count exceeds the supported limit")
    for value_index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"values[{value_index}] must be an exact integer")
        if not _MIN_INT64 <= value <= _MAX_INT64:
            raise ValueError(f"values[{value_index}] is outside signed 64-bit range")

    if type(queries) is not tuple:
        raise TypeError("queries must be an exact tuple")
    if len(queries) > _MAX_DISTINCT_QUERIES:
        raise ValueError("query count exceeds the supported limit")
    for query_index, query in enumerate(queries):
        if type(query) is not tuple:
            raise TypeError(f"queries[{query_index}] must be an exact tuple")
        if len(query) != 2:
            raise ValueError(f"queries[{query_index}] must contain two bounds")
        start, stop = query
        if type(start) is not int or type(stop) is not int:
            raise TypeError(f"queries[{query_index}] bounds must be exact integers")
        if not 0 <= start <= stop <= len(values):
            raise ValueError(f"queries[{query_index}] is outside the value sequence")

    tree = [0] * (len(values) + 1)

    def add(position: int, delta: int) -> None:
        tree_index = position + 1
        while tree_index < len(tree):
            tree[tree_index] += delta
            tree_index += tree_index & -tree_index

    def prefix_sum(stop: int) -> int:
        total = 0
        tree_index = stop
        while tree_index:
            total += tree[tree_index]
            tree_index -= tree_index & -tree_index
        return total

    query_order = sorted(range(len(queries)), key=lambda index: (queries[index][1], index))
    answers = [0] * len(queries)
    latest_positions: dict[int, int] = {}
    scanned = 0

    for query_index in query_order:
        start, stop = queries[query_index]
        while scanned < stop:
            value = values[scanned]
            previous = latest_positions.get(value)
            if previous is not None:
                add(previous, -1)
            add(scanned, 1)
            latest_positions[value] = scanned
            scanned += 1
        answers[query_index] = prefix_sum(stop) - prefix_sum(start)

    return tuple(answers)
```

## Example

```python
values = (4, 1, 4, 2, 1, 3)
queries = ((0, 6), (0, 3), (2, 5), (3, 3), (0, 6), (5, 6))
reordered_queries = tuple(reversed(queries))
signed_limits = (_MIN_INT64, _MAX_INT64, _MIN_INT64)

assert (
    count_distinct_values_offline(values, queries),
    count_distinct_values_offline(values, reordered_queries),
    count_distinct_values_offline((), ((0, 0), (0, 0))),
    count_distinct_values_offline(signed_limits, ((0, 3), (0, 1), (1, 3))),
) == (
    (4, 2, 3, 0, 4, 1),
    (1, 4, 0, 3, 2, 4),
    (0, 0),
    (2, 1, 2),
)
```

## Trade-offs and Limitations

For `n` values and `q` queries, sorting costs `O(q log q)` and the sweep plus
Fenwick operations costs `O((n + q) log(n + 1))`. The query order, answers,
Fenwick tree, and latest-position map use `O(n + q)` peak memory. Empty input
performs no tree updates and accepts only the valid range `[0, 0)`.

The key invariant is one active marker at the latest processed occurrence of
each value. This solves static distinct counts, not a generic aggregation API.
It materializes and sorts the complete query batch, so it does not stream
answers and cannot incorporate point updates between queries.

The implementation accepts exact signed 64-bit integers as values and exact
indexes as bounds. It does not handle arbitrary hashable objects, weighted
distinctness, approximate cardinality, range modes, or frequency thresholds.

## Related Snippets

<!-- catalog:related:start -->
- [Maintain Bounded Point Adds and Half-Open Range Sums with a Fenwick Tree](maintain-bounded-point-adds-and-half-open-range-sums-with-a-fenwick-tree.md)
- [Answer Static Half-Open Range-Minimum Queries with a Sparse Table](answer-static-half-open-range-minimum-queries-with-a-sparse-table.md)
- [Count Strict Inversions in a Bounded Integer Sequence](count-strict-inversions-in-a-bounded-integer-sequence.md)
<!-- catalog:related:end -->
