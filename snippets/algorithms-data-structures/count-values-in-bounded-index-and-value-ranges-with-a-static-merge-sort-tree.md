---
title: "Count Values in Bounded Index and Value Ranges with a Static Merge-Sort Tree"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - answer-static-k-th-smallest-queries-on-bounded-integer-ranges-with-a-persistent-segment-tree.md
  - count-distinct-values-in-bounded-half-open-ranges-offline-with-a-fenwick-tree.md
  - maintain-point-replacements-and-half-open-range-minima-with-a-segment-tree.md
---

# Count Values in Bounded Index and Value Ranges with a Static Merge-Sort Tree

## Idea and Problem

Answer many two-dimensional range-count queries over one immutable integer tuple without rescanning each requested slice.

A merge-sort tree is a segment tree whose node for each index interval stores
the values in that interval in sorted order. A query decomposes its half-open
index range into `O(log N)` disjoint nodes. Two binary searches in each stored
tuple count values inside the half-open value range, including every duplicate
occurrence.

The implementation exposes a batch function so the index is built once and
cannot outlive or diverge from the values it represents.

## When to Use

Use this approach when one in-memory sequence is fixed while many queries ask
how many entries in `values[start:stop]` fall between `lower` and `upper`. It is
useful for offline analytics, threshold counts, and validation workloads where
both the positional window and value interval vary independently.

Direct slice counting is simpler for a few queries. Use a persistent
order-statistic tree when kth-value queries are central, or a dynamic data
structure when values must change between queries. The `O(N log N)` stored
references are worthwhile only when repeated queries amortize construction.

## Implementation

```python
from bisect import bisect_left
from heapq import merge

_MIN_RANGE_VALUE = -(1 << 63)
_MAX_RANGE_VALUE = (1 << 63) - 1
_MAX_RANGE_BOUND = 1 << 63
_MAX_RANGE_VALUES = 65_536
_MAX_RANGE_QUERIES = 65_536


def count_values_in_static_ranges(
    values: tuple[int, ...],
    queries: tuple[tuple[int, int, int, int], ...],
) -> tuple[int, ...]:
    """Count multiplicities inside half-open index and value ranges."""
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    if len(values) > _MAX_RANGE_VALUES:
        raise ValueError("values contains more than 65,536 items")
    for index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"values[{index}] must be an exact integer")
        if not _MIN_RANGE_VALUE <= value <= _MAX_RANGE_VALUE:
            raise ValueError(f"values[{index}] is outside the signed 64-bit range")

    if type(queries) is not tuple:
        raise TypeError("queries must be an exact tuple")
    if len(queries) > _MAX_RANGE_QUERIES:
        raise ValueError("queries contains more than 65,536 items")

    checked_queries: list[tuple[int, int, int, int]] = []
    for query_index, query in enumerate(queries):
        if type(query) is not tuple or len(query) != 4:
            raise TypeError(f"queries[{query_index}] must be an exact four-item tuple")
        start, stop, lower, upper = query
        if any(type(value) is not int for value in query):
            raise TypeError(f"queries[{query_index}] fields must be exact integers")
        if not 0 <= start <= stop <= len(values):
            raise ValueError(f"queries[{query_index}] has an invalid index range")
        if not (
            _MIN_RANGE_VALUE <= lower <= _MAX_RANGE_BOUND
            and _MIN_RANGE_VALUE <= upper <= _MAX_RANGE_BOUND
        ):
            raise ValueError(f"queries[{query_index}] has an unsupported value endpoint")
        if lower > upper:
            raise ValueError(f"queries[{query_index}] has a reversed value range")
        checked_queries.append((start, stop, lower, upper))

    leaf_count = 1
    while leaf_count < len(values):
        leaf_count *= 2
    tree: list[tuple[int, ...]] = [()] * (2 * leaf_count)
    for index, value in enumerate(values):
        tree[leaf_count + index] = (value,)
    for node in range(leaf_count - 1, 0, -1):
        tree[node] = tuple(merge(tree[2 * node], tree[2 * node + 1]))

    answers: list[int] = []
    for start, stop, lower, upper in checked_queries:
        left = start + leaf_count
        right = stop + leaf_count
        count = 0
        while left < right:
            if left & 1:
                ordered = tree[left]
                count += bisect_left(ordered, upper) - bisect_left(ordered, lower)
                left += 1
            if right & 1:
                right -= 1
                ordered = tree[right]
                count += bisect_left(ordered, upper) - bisect_left(ordered, lower)
            left //= 2
            right //= 2
        answers.append(count)
    return tuple(answers)
```

## Example

```python
from itertools import product


def direct_range_count(
    values: tuple[int, ...],
    query: tuple[int, int, int, int],
) -> int:
    start, stop, lower, upper = query
    return sum(lower <= value < upper for value in values[start:stop])


checked = 0
value_bounds = (-2, -1, 0, 1, 2)
for size in range(6):
    for small_values in product((-1, 0, 1), repeat=size):
        queries = tuple(
            (start, stop, lower, upper)
            for start in range(size + 1)
            for stop in range(start, size + 1)
            for lower in value_bounds
            for upper in value_bounds
            if lower <= upper
        )
        expected = tuple(direct_range_count(small_values, query) for query in queries)
        assert count_values_in_static_ranges(small_values, queries) == expected
        checked += len(queries)

maximum_values = (0,) * _MAX_RANGE_VALUES
maximum_answers = count_values_in_static_ranges(
    maximum_values,
    ((0, _MAX_RANGE_VALUES, 0, 1), (123, 123, -1, 1)),
)
boundary_answers = count_values_in_static_ranges(
    (_MIN_RANGE_VALUE, _MAX_RANGE_VALUE),
    ((0, 2, _MAX_RANGE_VALUE, _MAX_RANGE_BOUND),),
)


def rejects(values: object, queries: object) -> bool:
    try:
        count_values_in_static_ranges(values, queries)  # type: ignore[arg-type]
    except (TypeError, ValueError):
        return True
    return False


assert (
    checked,
    count_values_in_static_ranges((), ((0, 0, -1, 1),)),
    maximum_answers,
    boundary_answers,
    rejects([1], ()),
    rejects((1, True), ()),
    rejects((1,), ((0, 2, 0, 1),)),
    rejects((1,), ((0, 1, 2, 1),)),
    rejects((1,), ((0, 1, 0, True),)),
) == (99_780, (0,), (65_536, 0), (1,), True, True, True, True, True)
```

## Trade-offs and Limitations

Construction takes `O(N log N)` comparisons and stores `O(N log N)` integer
references across sorted tuples. Each query visits `O(log N)` nodes and makes
two binary searches per node, for `O(log**2 N)` time. The result itself takes
`O(Q)` space. Python tuples and integer references make the real memory use
substantially larger than a packed native implementation.

Counts include multiplicity: repeated equal values are not collapsed. Both
index and value intervals are half-open, so an empty interval returns zero.
Every value must be a signed 64-bit integer. Value-range endpoints additionally
admit `2**63` so a half-open interval can include the maximum signed value;
counts and internal indexes use ordinary exact Python integers.

The tree is rebuilt for every function call and supports no updates, kth-value
selection, sums, distinct counts, persistence, serialization, or sparse
storage. A direct scan can be faster when the batch is small.

## Related Snippets

<!-- catalog:related:start -->
- [Answer Static K-th-Smallest Queries on Bounded Integer Ranges with a Persistent Segment Tree](answer-static-k-th-smallest-queries-on-bounded-integer-ranges-with-a-persistent-segment-tree.md)
- [Count Distinct Values in Bounded Half-Open Ranges Offline with a Fenwick Tree](count-distinct-values-in-bounded-half-open-ranges-offline-with-a-fenwick-tree.md)
- [Maintain Point Replacements and Half-Open Range Minima with a Segment Tree](maintain-point-replacements-and-half-open-range-minima-with-a-segment-tree.md)
<!-- catalog:related:end -->
