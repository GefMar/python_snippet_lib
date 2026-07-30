---
title: "Answer Static K-th-Smallest Queries on Bounded Integer Ranges with a Persistent Segment Tree"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - select-a-zero-based-order-statistic-from-bounded-integers-with-three-way-quickselect.md
  - answer-static-half-open-range-minimum-queries-with-a-sparse-table.md
  - count-distinct-values-in-bounded-half-open-ranges-offline-with-a-fenwick-tree.md
---

# Answer Static K-th-Smallest Queries on Bounded Integer Ranges with a Persistent Segment Tree

## Idea and Problem

Return the zero-based k-th-smallest value from each requested half-open range of one immutable integer sequence.

Coordinate compression maps every distinct value to a sorted integer index.
For each input prefix, a persistent count tree reuses the previous root while
copying only the nodes on the new value's path. Subtracting the node counts of
the `start` prefix from those of the `stop` prefix gives the frequency tree for
exactly `values[start:stop]`.

A query descends by comparing its rank with the left-child count. Duplicate
occurrences remain separate counts even though equal values share one
compressed coordinate, and answers retain query declaration order.

## When to Use

Use this structure when one integer sequence is static and many arbitrary
range order-statistic queries are known. It is useful when sorting every slice
would repeat too much work and when the persistent prefix roots fit in memory.

Use `sorted(values[start:stop])[rank]` for a few short ranges. Quickselect is a
better fit for one whole-sequence order statistic. Choose a wavelet structure
for a different space/performance balance, or a dynamic order-statistic tree
when values change between queries.

## Implementation

```python
_MIN_RANGE_VALUE = -(1 << 63)
_MAX_RANGE_VALUE = (1 << 63) - 1
_MAX_RANGE_VALUES = 20_000
_MAX_RANGE_QUERIES = 50_000


def answer_static_kth_smallest_queries(
    values: tuple[int, ...],
    queries: tuple[tuple[int, int, int], ...],
) -> tuple[int, ...]:
    """Return range order statistics in query declaration order."""
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    if not 1 <= len(values) <= _MAX_RANGE_VALUES:
        raise ValueError("value count is outside 1..20000")
    for value_index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"values[{value_index}] must be an exact integer")
        if not _MIN_RANGE_VALUE <= value <= _MAX_RANGE_VALUE:
            raise ValueError(f"values[{value_index}] is outside signed 64-bit range")

    if type(queries) is not tuple:
        raise TypeError("queries must be an exact tuple")
    if len(queries) > _MAX_RANGE_QUERIES:
        raise ValueError("query count exceeds the supported limit")
    for query_index, query in enumerate(queries):
        if type(query) is not tuple:
            raise TypeError(f"queries[{query_index}] must be an exact tuple")
        if len(query) != 3:
            raise ValueError(
                f"queries[{query_index}] must contain start, stop, and rank"
            )
        start, stop, rank = query
        if type(start) is not int or type(stop) is not int:
            raise TypeError(f"queries[{query_index}] bounds must be exact integers")
        if type(rank) is not int:
            raise TypeError(f"queries[{query_index}].rank must be an exact integer")
        if not 0 <= start < stop <= len(values):
            raise ValueError(f"queries[{query_index}] range is invalid")
        if not 0 <= rank < stop - start:
            raise ValueError(f"queries[{query_index}].rank is outside its range")

    coordinates = sorted(set(values))
    coordinate_indexes = {
        value: index for index, value in enumerate(coordinates)
    }

    left_children = [0]
    right_children = [0]
    counts = [0]

    def add(previous: int, start: int, stop: int, position: int) -> int:
        node = len(counts)
        left_children.append(left_children[previous])
        right_children.append(right_children[previous])
        counts.append(counts[previous] + 1)
        if stop - start == 1:
            return node

        middle = (start + stop) // 2
        if position < middle:
            left_children[node] = add(
                left_children[previous],
                start,
                middle,
                position,
            )
        else:
            right_children[node] = add(
                right_children[previous],
                middle,
                stop,
                position,
            )
        return node

    roots = [0]
    for value in values:
        roots.append(
            add(
                roots[-1],
                0,
                len(coordinates),
                coordinate_indexes[value],
            )
        )

    answers: list[int] = []
    for start, stop, rank in queries:
        left_root = roots[start]
        right_root = roots[stop]
        coordinate_start = 0
        coordinate_stop = len(coordinates)

        while coordinate_stop - coordinate_start > 1:
            left_count = (
                counts[left_children[right_root]]
                - counts[left_children[left_root]]
            )
            middle = (coordinate_start + coordinate_stop) // 2
            if rank < left_count:
                left_root = left_children[left_root]
                right_root = left_children[right_root]
                coordinate_stop = middle
            else:
                rank -= left_count
                left_root = right_children[left_root]
                right_root = right_children[right_root]
                coordinate_start = middle
        answers.append(coordinates[coordinate_start])

    return tuple(answers)
```

## Example

```python
def exercise_small_ranges() -> int:
    from itertools import product

    checked = 0
    for value_count in range(1, 8):
        for values in product((-1, 0, 1), repeat=value_count):
            queries = tuple(
                (start, stop, rank)
                for start in range(value_count)
                for stop in range(start + 1, value_count + 1)
                for rank in range(stop - start)
            )
            expected = tuple(
                sorted(values[start:stop])[rank]
                for start, stop, rank in queries
            )
            assert answer_static_kth_smallest_queries(values, queries) == expected
            checked += len(queries)
    return checked


checked_answers = exercise_small_ranges()
values = (7, -2, 7, 5, -2, 9, 5)
queries = ((0, 7, 0), (0, 7, 6), (1, 6, 2), (2, 5, 1), (3, 4, 0))

boundary_values = tuple(
    _MIN_RANGE_VALUE if index % 2 == 0 else _MAX_RANGE_VALUE
    for index in range(_MAX_RANGE_VALUES)
)
boundary_queries = tuple(
    (0, _MAX_RANGE_VALUES, rank % _MAX_RANGE_VALUES)
    for rank in range(_MAX_RANGE_QUERIES)
)
boundary_answers = answer_static_kth_smallest_queries(
    boundary_values,
    boundary_queries,
)

invalid_rank_rejected = False
try:
    answer_static_kth_smallest_queries((1, 2), ((0, 2, 2),))
except ValueError:
    invalid_rank_rejected = True

assert (
    checked_answers == 234_966
    and answer_static_kth_smallest_queries(values, queries)
    == (-2, 9, 5, 5, 5)
    and boundary_answers[:3]
    == (_MIN_RANGE_VALUE, _MIN_RANGE_VALUE, _MIN_RANGE_VALUE)
    and boundary_answers[_MAX_RANGE_VALUES - 1] == _MAX_RANGE_VALUE
    and boundary_answers[-1] == _MIN_RANGE_VALUE
    and invalid_rank_rejected
)
```

## Trade-offs and Limitations

Sorting and compressing `N` values costs `O(N log N)`. Building one persistent
root per prefix and answering `Q` queries costs
`O(N log(U + 1) + Q log(U + 1))`, where `U` is the number of distinct values.
The coordinates, roots, copied nodes, and answers use
`O(N log(U + 1) + U + Q)` memory.

Persistence here is an internal prefix representation, not a public versioned
update API. Duplicate occurrences contribute separate counts but return the
same scalar value; no occurrence index is selected. Every range is non-empty,
half-open, and uses a zero-based rank.

The implementation accepts exact signed 64-bit integers only. It does not
support replacements, insertions, occurrence indexes, interpolated quantiles,
floating-point values, custom comparison keys, streaming inputs, or concurrent
mutation.

## Related Snippets

<!-- catalog:related:start -->
- [Select a Zero-Based Order Statistic from Bounded Integers with Three-Way Quickselect](select-a-zero-based-order-statistic-from-bounded-integers-with-three-way-quickselect.md)
- [Answer Static Half-Open Range-Minimum Queries with a Sparse Table](answer-static-half-open-range-minimum-queries-with-a-sparse-table.md)
- [Count Distinct Values in Bounded Half-Open Ranges Offline with a Fenwick Tree](count-distinct-values-in-bounded-half-open-ranges-offline-with-a-fenwick-tree.md)
<!-- catalog:related:end -->
