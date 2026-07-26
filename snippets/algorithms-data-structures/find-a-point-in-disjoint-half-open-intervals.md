---
title: "Find a Point in Disjoint Half-Open Intervals"
snippet_type: algorithm
use_cases:
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - cover-a-half-open-integer-range-with-dyadic-intervals.md
---

# Find a Point in Disjoint Half-Open Intervals

## Idea and Problem

Build a validated sorted index of disjoint half-open integer intervals and locate a containing entry with binary search.

Sorting and validating once moves overlap errors to construction time. A
lookup then bisects interval starts and inspects only the nearest candidate.
Returning the complete entry keeps a payload value of `None` distinct from no
matching interval.

## When to Use

Use this index for a stable collection of non-overlapping integer ranges with
many point lookups. Bounds follow `[start, stop)`, so adjacent intervals are
valid and a point equal to `stop` belongs to the next interval, if any. Use an
interval tree or a domain-specific structure when overlaps, range queries, or
frequent updates are required.

## Implementation

```python
from bisect import bisect_right
from collections.abc import Iterable
from dataclasses import dataclass
from typing import Generic, TypeVar


ValueT = TypeVar("ValueT")


@dataclass(frozen=True, slots=True)
class IntervalEntry(Generic[ValueT]):
    start: int
    stop: int
    value: ValueT


@dataclass(frozen=True, slots=True, init=False)
class HalfOpenIntervalIndex(Generic[ValueT]):
    _entries: tuple[IntervalEntry[ValueT], ...]
    _starts: tuple[int, ...]

    def __init__(self, entries: Iterable[IntervalEntry[ValueT]]) -> None:
        provided = tuple(entries)
        for entry in provided:
            if type(entry.start) is not int or type(entry.stop) is not int:
                raise TypeError("interval bounds must be integers")
            if entry.start >= entry.stop:
                raise ValueError("each interval must have start < stop")

        ordered = tuple(sorted(provided, key=lambda entry: entry.start))
        for left, right in zip(ordered, ordered[1:]):
            if right.start < left.stop:
                raise ValueError("intervals must not overlap")

        object.__setattr__(self, "_entries", ordered)
        object.__setattr__(
            self,
            "_starts",
            tuple(entry.start for entry in ordered),
        )

    def find(self, point: int) -> IntervalEntry[ValueT] | None:
        if type(point) is not int:
            raise TypeError("point must be an integer")
        position = bisect_right(self._starts, point) - 1
        if position < 0:
            return None
        candidate = self._entries[position]
        return candidate if point < candidate.stop else None
```

## Example

```python
index = HalfOpenIntervalIndex(
    (
        IntervalEntry(10, 15, "high"),
        IntervalEntry(0, 5, "low"),
        IntervalEntry(5, 8, None),
    )
)
matches = tuple(
    index.find(point)
    for point in (-1, 0, 4, 5, 7, 8, 10, 14, 15)
)

try:
    HalfOpenIntervalIndex(
        (IntervalEntry(0, 5, "left"), IntervalEntry(4, 6, "right"))
    )
except ValueError:
    overlap_rejected = True
else:
    overlap_rejected = False

try:
    HalfOpenIntervalIndex((IntervalEntry(1, 1, "empty"),))
except ValueError:
    empty_interval_rejected = True
else:
    empty_interval_rejected = False

try:
    index.find(True)
except TypeError:
    bool_point_rejected = True
else:
    bool_point_rejected = False

try:
    HalfOpenIntervalIndex((IntervalEntry(False, 1, "invalid"),))
except TypeError:
    bool_bound_rejected = True
else:
    bool_bound_rejected = False

assert (
    matches,
    HalfOpenIntervalIndex(()).find(0),
    overlap_rejected,
    empty_interval_rejected,
    bool_point_rejected,
    bool_bound_rejected,
) == (
    (
        None,
        IntervalEntry(0, 5, "low"),
        IntervalEntry(0, 5, "low"),
        IntervalEntry(5, 8, None),
        IntervalEntry(5, 8, None),
        None,
        IntervalEntry(10, 15, "high"),
        IntervalEntry(10, 15, "high"),
        None,
    ),
    None,
    True,
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

Construction sorts in `O(n log n)` time and stores two immutable tuples;
lookup is `O(log n)`. Payloads can still be mutable. The index supports only
integer point queries over disjoint half-open intervals and has no update
operation. Equality at boundaries is a contract choice, so converting from
closed ranges requires deliberate endpoint handling.

## Related Snippets

<!-- catalog:related:start -->
- [Cover a Half-Open Integer Range with Dyadic Intervals](cover-a-half-open-integer-range-with-dyadic-intervals.md)
<!-- catalog:related:end -->
