---
title: "Maintain Bounded Point Adds and Half-Open Range Sums with a Fenwick Tree"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - find-a-point-in-disjoint-half-open-intervals.md
  - compute-full-window-trailing-maxima-with-a-monotonic-index-deque.md
  - ../observability-operations/count-values-in-fixed-upper-bound-bins.md
---

# Maintain Bounded Point Adds and Half-Open Range Sums with a Fenwick Tree

## Idea and Problem

Own a bounded integer snapshot in a Fenwick tree so point additions and arbitrary half-open range sums both take logarithmic time.

Each tree position stores the sum of one power-of-two suffix of a prefix. A
point addition updates every stored suffix that contains that position, while a
prefix query repeatedly removes the lowest set bit. Subtracting two prefix sums
produces the sum over `[start, stop)`.

## When to Use

Use this data structure when one bounded in-memory integer sequence receives
many point additions interleaved with range-sum queries. It is particularly
useful when rebuilding prefix sums after every change would cost too much and a
full segment tree would provide operations the caller does not need.

The instance owns copies of both the values and its index. Keep all access
behind the instance, and use a different structure for range updates, minimum
queries, immutable snapshots, concurrent mutation, or durable state.

## Implementation

```python
_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_VALUE_COUNT = 10_000


class FenwickRangeSums:
    """Maintain point additions and exact half-open sums for owned integers."""

    __slots__ = ("_tree", "_values")

    def __init__(self, values: tuple[int, ...]) -> None:
        if type(values) is not tuple:
            raise TypeError("values must be an exact tuple")
        if not 1 <= len(values) <= _MAX_VALUE_COUNT:
            raise ValueError("value count is outside the supported range")

        owned_values: list[int] = []
        for value in values:
            if type(value) is not int:
                raise TypeError("values must contain exact integers")
            if not _MIN_INT64 <= value <= _MAX_INT64:
                raise ValueError("values must be in the signed 64-bit range")
            owned_values.append(value)

        tree = [0, *owned_values]
        for position in range(1, len(tree)):
            parent = position + (position & -position)
            if parent < len(tree):
                tree[parent] += tree[position]

        self._values = owned_values
        self._tree = tree

    def _validated_index(self, value: object, *, allow_end: bool) -> int:
        if type(value) is not int:
            raise TypeError("range indices must be exact integers")
        upper_bound = len(self._values) if allow_end else len(self._values) - 1
        if not 0 <= value <= upper_bound:
            raise IndexError("index is outside the supported range")
        return value

    def _prefix_sum(self, stop: int) -> int:
        total = 0
        while stop > 0:
            total += self._tree[stop]
            stop -= stop & -stop
        return total

    def add(self, index: int, delta: int) -> None:
        """Add delta to one point without changing it outside int64."""
        checked_index = self._validated_index(index, allow_end=False)
        if type(delta) is not int:
            raise TypeError("delta must be an exact integer")
        if not _MIN_INT64 <= delta <= _MAX_INT64:
            raise ValueError("delta must be in the signed 64-bit range")

        updated_value = self._values[checked_index] + delta
        if not _MIN_INT64 <= updated_value <= _MAX_INT64:
            raise ValueError("the updated point would leave the signed 64-bit range")

        self._values[checked_index] = updated_value
        position = checked_index + 1
        while position < len(self._tree):
            self._tree[position] += delta
            position += position & -position

    def range_sum(self, start: int, stop: int) -> int:
        """Return the exact sum over the half-open range [start, stop)."""
        checked_start = self._validated_index(start, allow_end=True)
        checked_stop = self._validated_index(stop, allow_end=True)
        if checked_start > checked_stop:
            raise ValueError("start must not be greater than stop")
        return self._prefix_sum(checked_stop) - self._prefix_sum(checked_start)
```

## Example

```python
totals = FenwickRangeSums((3, -2, 7, 1))
before = (totals.range_sum(0, 4), totals.range_sum(1, 3))
totals.add(1, 5)
after = (totals.range_sum(0, 2), totals.range_sum(2, 2))

try:
    totals.add(2, _MAX_INT64)
except ValueError:
    overflow_rejected = True
else:
    overflow_rejected = False

assert (before, after, totals.range_sum(2, 3), overflow_rejected) == (
    (9, 5),
    (6, 0),
    7,
    True,
)
```

## Trade-offs and Limitations

Validation and linear Fenwick construction take `O(n)` time. Each point add,
prefix sum, and half-open range sum takes `O(log n)` time, and the owned value
and tree lists use `O(n)` memory. Stored points remain signed 64-bit integers,
but Python calculates aggregate tree and result sums with exact integers beyond
that range.

An update is a signed 64-bit addition, not an assignment. Empty ranges are
valid and sum to zero; reversed or out-of-bounds ranges are rejected. The
structure does not provide range updates, minimum or maximum queries,
insertion, deletion, versioned snapshots, persistence, or synchronization.
Callers must serialize concurrent reads and writes themselves.

## Related Snippets

<!-- catalog:related:start -->
- [Find a Point in Disjoint Half-Open Intervals](find-a-point-in-disjoint-half-open-intervals.md)
- [Compute Full-Window Trailing Maxima with a Monotonic Index Deque](compute-full-window-trailing-maxima-with-a-monotonic-index-deque.md)
- [Count Values in Fixed Upper-Bound Bins](../observability-operations/count-values-in-fixed-upper-bound-bins.md)
<!-- catalog:related:end -->
