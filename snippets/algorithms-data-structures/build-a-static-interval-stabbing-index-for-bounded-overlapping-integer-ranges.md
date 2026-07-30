---
title: "Build a Static Interval Stabbing Index for Bounded Overlapping Integer Ranges"
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
  - find-peak-coverage-spans-across-bounded-half-open-integer-intervals.md
  - assign-bounded-half-open-integer-intervals-to-the-minimum-number-of-reusable-rooms.md
---

# Build a Static Interval Stabbing Index for Bounded Overlapping Integer Ranges

## Idea and Problem

Build an immutable centered interval tree that returns the declaration indexes of every half-open integer interval containing a query point.

Each node chooses the median of the remaining intervals' integer midpoints.
Intervals wholly before or after that center descend into balanced subtrees;
intervals containing it stay at the node in both ascending-start and
descending-stop order. A query follows only one subtree and scans only the
centered prefix that can contain its point.

## When to Use

Use this index for a fixed, bounded collection of overlapping ranges that will
receive many point queries. Declaration indexes retain duplicates and let a
caller keep payloads in a separate immutable sequence without copying them
into the tree.

Use binary search over starts for disjoint intervals. Use a sweep when every
endpoint is processed together, or a different data structure for updates,
deletions, range queries, floating-point endpoints, or unbounded input.

## Implementation

```python
from dataclasses import dataclass
from typing import Self

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_INTERVALS = 4_096


@dataclass(frozen=True, slots=True)
class _DeclaredInterval:
    start: int
    stop: int
    declaration_index: int


@dataclass(frozen=True, slots=True)
class _IntervalNode:
    center: int
    by_start: tuple[_DeclaredInterval, ...]
    by_descending_stop: tuple[_DeclaredInterval, ...]
    left: Self | None
    right: Self | None


def _build_interval_node(
    intervals: tuple[_DeclaredInterval, ...],
) -> _IntervalNode | None:
    if not intervals:
        return None

    midpoints = sorted(
        interval.start + (interval.stop - interval.start) // 2 for interval in intervals
    )
    center = midpoints[len(intervals) // 2]
    left: list[_DeclaredInterval] = []
    centered: list[_DeclaredInterval] = []
    right: list[_DeclaredInterval] = []

    for interval in intervals:
        if interval.stop <= center:
            left.append(interval)
        elif interval.start > center:
            right.append(interval)
        else:
            centered.append(interval)

    return _IntervalNode(
        center=center,
        by_start=tuple(
            sorted(
                centered,
                key=lambda interval: (
                    interval.start,
                    interval.declaration_index,
                ),
            )
        ),
        by_descending_stop=tuple(
            sorted(
                centered,
                key=lambda interval: (
                    interval.stop,
                    interval.declaration_index,
                ),
                reverse=True,
            )
        ),
        left=_build_interval_node(tuple(left)),
        right=_build_interval_node(tuple(right)),
    )


@dataclass(frozen=True, slots=True, init=False)
class StaticIntervalStabbingIndex:
    _root: _IntervalNode

    def __init__(self, intervals: tuple[tuple[int, int], ...]) -> None:
        if type(intervals) is not tuple:
            raise TypeError("intervals must be an exact tuple")
        if not 1 <= len(intervals) <= _MAX_INTERVALS:
            raise ValueError("interval count is outside the supported range")

        declared: list[_DeclaredInterval] = []
        for declaration_index, interval in enumerate(intervals):
            if type(interval) is not tuple:
                raise TypeError("each interval must be an exact tuple")
            if len(interval) != 2:
                raise ValueError("each interval must contain two endpoints")
            start, stop = interval
            if type(start) is not int or type(stop) is not int:
                raise TypeError("interval endpoints must be exact integers")
            if not _MIN_INT64 <= start <= _MAX_INT64:
                raise ValueError("interval start is outside signed 64-bit range")
            if not _MIN_INT64 <= stop <= _MAX_INT64:
                raise ValueError("interval stop is outside signed 64-bit range")
            if start >= stop:
                raise ValueError("each interval must have start < stop")
            declared.append(_DeclaredInterval(start, stop, declaration_index))

        root = _build_interval_node(tuple(declared))
        assert root is not None
        object.__setattr__(self, "_root", root)

    def find_all(self, point: int) -> tuple[int, ...]:
        if type(point) is not int:
            raise TypeError("point must be an exact integer")
        if not _MIN_INT64 <= point <= _MAX_INT64:
            raise ValueError("point is outside signed 64-bit range")

        matches: list[int] = []
        node: _IntervalNode | None = self._root
        while node is not None:
            if point < node.center:
                for interval in node.by_start:
                    if interval.start > point:
                        break
                    matches.append(interval.declaration_index)
                node = node.left
            elif point > node.center:
                for interval in node.by_descending_stop:
                    if interval.stop <= point:
                        break
                    matches.append(interval.declaration_index)
                node = node.right
            else:
                matches.extend(interval.declaration_index for interval in node.by_start)
                break

        matches.sort()
        return tuple(matches)


```

## Example

```python
intervals = ((0, 5), (2, 7), (2, 7), (5, 8), (-3, 1))
index = StaticIntervalStabbingIndex(intervals)

assert tuple(index.find_all(point) for point in (-3, 0, 1, 2, 5, 7, 8)) == (
    (4,),
    (0, 4),
    (0,),
    (0, 1, 2),
    (1, 2, 3),
    (3,),
    (),
)

try:
    index.find_all(True)
except TypeError:
    boolean_rejected = True
else:
    boolean_rejected = False

assert boolean_rejected
```

## Trade-offs and Limitations

Choosing median interval midpoints keeps both descendant groups at most half
the current node size, so tree height is `O(log n)`. This deliberately simple
build sorts midpoints independently at each level and costs `O(n log² n)`
time. The frozen nodes and their two centered views retain `O(n)` references.

A query scans `O(log n + k)` tree records and then sorts its `k` declaration
indexes, for `O(log n + k log k)` time and `O(k)` output. An interval ending
at a center belongs left, while one starting at the center is centered; this
is what makes `start <= point < stop` hold at touching boundaries.

Inputs are exact built-in tuples and integers within the signed 64-bit range.
The index has no payload storage, updates, deletions, range-query API, or
serialization format.

## Related Snippets

<!-- catalog:related:start -->
- [Find a Point in Disjoint Half-Open Intervals](find-a-point-in-disjoint-half-open-intervals.md)
- [Find Peak Coverage Spans Across Bounded Half-Open Integer Intervals](find-peak-coverage-spans-across-bounded-half-open-integer-intervals.md)
- [Assign Bounded Half-Open Integer Intervals to the Minimum Number of Reusable Rooms](assign-bounded-half-open-integer-intervals-to-the-minimum-number-of-reusable-rooms.md)
<!-- catalog:related:end -->
