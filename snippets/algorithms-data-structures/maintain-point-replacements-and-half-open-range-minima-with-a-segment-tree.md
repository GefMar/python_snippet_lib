---
title: "Maintain Point Replacements and Half-Open Range Minima with a Segment Tree"
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
  - build-a-bounded-integer-summed-area-table-for-half-open-rectangle-sums.md
  - compute-full-window-trailing-maxima-with-a-monotonic-index-deque.md
---

# Maintain Point Replacements and Half-Open Range Minima with a Segment Tree

## Idea and Problem

Own a bounded integer sequence in a segment tree so point replacements and leftmost half-open range minima both take logarithmic time.

Every leaf stores one value and its original position. Each parent stores the
smaller child, comparing positions when values are equal. A query visits only
the canonical tree nodes covering `[start, stop)`, while a replacement rebuilds
the single path from one leaf to the root.

## When to Use

Use this structure when one bounded in-memory sequence receives many point
replacements interleaved with minimum queries over arbitrary non-empty ranges.
It is especially useful when equal minima need a stable leftmost position and
rescanning each requested range would be too expensive.

Use a plain list when updates or queries are rare. Prefer a Fenwick tree for
point additions and range sums, an immutable prefix structure for static data,
or a specialized array library when compact numeric storage and vectorized
operations matter more than a standard-library-only object.

## Implementation

```python
from dataclasses import dataclass

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_VALUE_COUNT = 65_536


@dataclass(frozen=True, slots=True)
class RangeMinimum:
    value: int
    index: int


def _validated_int64(value: object, *, field: str) -> int:
    if type(value) is not int:
        raise TypeError(f"{field} must be an exact integer")
    if not _MIN_INT64 <= value <= _MAX_INT64:
        raise ValueError(f"{field} must be in the signed 64-bit range")
    return value


def _smaller(
    left: RangeMinimum | None,
    right: RangeMinimum | None,
) -> RangeMinimum | None:
    if left is None:
        return right
    if right is None:
        return left
    if (right.value, right.index) < (left.value, left.index):
        return right
    return left


class SegmentTreeRangeMinima:
    """Maintain leftmost range minima for one owned tree state."""

    __slots__ = ("_leaf_count", "_length", "_tree")

    def __init__(self, values: tuple[int, ...]) -> None:
        if type(values) is not tuple:
            raise TypeError("values must be an exact tuple")
        if not 1 <= len(values) <= _MAX_VALUE_COUNT:
            raise ValueError("value count is outside the supported range")

        for index, value in enumerate(values):
            _validated_int64(value, field=f"values[{index}]")

        leaf_count = 1 << (len(values) - 1).bit_length()
        tree: list[RangeMinimum | None] = [None] * (2 * leaf_count)
        for index, value in enumerate(values):
            tree[leaf_count + index] = RangeMinimum(value, index)
        for position in range(leaf_count - 1, 0, -1):
            tree[position] = _smaller(tree[2 * position], tree[2 * position + 1])

        self._leaf_count = leaf_count
        self._length = len(values)
        self._tree = tree

    def replace(self, index: int, value: int) -> None:
        """Replace one point and rebuild its path to the root."""
        if type(index) is not int:
            raise TypeError("index must be an exact integer")
        if not 0 <= index < self._length:
            raise IndexError("index is outside the sequence")
        checked_value = _validated_int64(value, field="value")

        position = self._leaf_count + index
        self._tree[position] = RangeMinimum(checked_value, index)
        while position > 1:
            position //= 2
            self._tree[position] = _smaller(
                self._tree[2 * position],
                self._tree[2 * position + 1],
            )

    def range_minimum(self, start: int, stop: int) -> RangeMinimum:
        """Return the leftmost minimum over the non-empty range [start, stop)."""
        if type(start) is not int or type(stop) is not int:
            raise TypeError("range bounds must be exact integers")
        if not 0 <= start < stop <= self._length:
            raise ValueError("range must satisfy 0 <= start < stop <= length")

        left = self._leaf_count + start
        right = self._leaf_count + stop
        best: RangeMinimum | None = None
        while left < right:
            if left & 1:
                best = _smaller(best, self._tree[left])
                left += 1
            if right & 1:
                right -= 1
                best = _smaller(best, self._tree[right])
            left //= 2
            right //= 2

        if best is None:
            raise RuntimeError("a validated non-empty range produced no minimum")
        return best
```

## Example

```python
minima = SegmentTreeRangeMinima((7, 3, 9, 3, 5))
before = (
    minima.range_minimum(0, 5),
    minima.range_minimum(2, 5),
)

minima.replace(1, 8)
after_first_replacement = minima.range_minimum(0, 5)
minima.replace(4, 3)
after_new_tie = minima.range_minimum(0, 5)
minima.replace(3, 10)
after_tie_removed = minima.range_minimum(0, 5)
minima.replace(4, -2)
minima.replace(4, -2)

assert (
    before,
    after_first_replacement,
    after_new_tie,
    after_tie_removed,
    minima.range_minimum(0, 5),
    minima.range_minimum(2, 4),
) == (
    (RangeMinimum(3, 1), RangeMinimum(3, 3)),
    RangeMinimum(3, 3),
    RangeMinimum(3, 3),
    RangeMinimum(3, 4),
    RangeMinimum(-2, 4),
    RangeMinimum(9, 2),
)
```

## Trade-offs and Limitations

Validation and construction take `O(n)` time. The power-of-two leaf capacity
is less than twice the source length, so the complete backing array has fewer
than `4n` slots and uses `O(n)` memory.
Each point replacement and non-empty half-open query takes `O(log n)` time and
`O(1)` additional working space. Stored values remain signed 64-bit integers;
comparisons do not perform arithmetic that could overflow.

The tree supports assignments, not point additions. It deliberately excludes
empty ranges, range updates, lazy propagation, insertion, deletion, persistent
versions, snapshots, generic comparison keys, and synchronization. The frozen
result is safe to retain, but the tree itself is mutable; callers must
serialize concurrent reads and replacements.

## Related Snippets

<!-- catalog:related:start -->
- [Maintain Bounded Point Adds and Half-Open Range Sums with a Fenwick Tree](maintain-bounded-point-adds-and-half-open-range-sums-with-a-fenwick-tree.md)
- [Build a Bounded Integer Summed-Area Table for Half-Open Rectangle Sums](build-a-bounded-integer-summed-area-table-for-half-open-rectangle-sums.md)
- [Compute Full-Window Trailing Maxima with a Monotonic Index Deque](compute-full-window-trailing-maxima-with-a-monotonic-index-deque.md)
<!-- catalog:related:end -->
