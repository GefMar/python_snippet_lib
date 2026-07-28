---
title: "Select a Maximum-Cardinality Set of Non-Overlapping Half-Open Integer Intervals"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - coalesce-bounded-half-open-integer-intervals-under-an-explicit-adjacency-policy.md
  - find-a-point-in-disjoint-half-open-intervals.md
  - cover-a-half-open-integer-range-with-dyadic-intervals.md
---

# Select a Maximum-Cardinality Set of Non-Overlapping Half-Open Integer Intervals

## Idea and Problem

Select as many mutually compatible intervals as possible by always taking the next interval that finishes earliest.

Each result is the declaration index of an original interval. Sorting candidates
by stop, start, and declaration index makes one fixed input deterministic while
preserving duplicate intervals as distinct choices. With half-open ranges,
`[1, 3)` and `[3, 5)` do not overlap and may both be selected.

## When to Use

Use this algorithm for a bounded in-memory set of unweighted activities when
the objective is solely to maximize how many can fit without overlap. Signed
integer endpoints must already share one unit and timeline, and declaration
order must be an acceptable final tie-breaker.

Choose a different algorithm when intervals carry values, priorities, setup
times, resource identities, or limits on how many may run concurrently. Those
objectives are not solved by the earliest-finish greedy rule.

## Implementation

```python
from dataclasses import dataclass

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_INTERVALS = 4_096


@dataclass(frozen=True, slots=True)
class HalfOpenInterval:
    start: int
    stop: int


def select_maximum_non_overlapping_intervals(
    intervals: tuple[HalfOpenInterval, ...],
) -> tuple[int, ...]:
    """Return original indexes for one maximum-cardinality selection."""
    if type(intervals) is not tuple:
        raise TypeError("intervals must be an exact tuple")
    if len(intervals) > _MAX_INTERVALS:
        raise ValueError("interval count exceeds the supported limit")

    for interval in intervals:
        if type(interval) is not HalfOpenInterval:
            raise TypeError("each interval must be an exact HalfOpenInterval")
        if type(interval.start) is not int or type(interval.stop) is not int:
            raise TypeError("interval endpoints must be exact integers")
        if not _MIN_INT64 <= interval.start <= _MAX_INT64:
            raise ValueError("interval start is outside the signed 64-bit range")
        if not _MIN_INT64 <= interval.stop <= _MAX_INT64:
            raise ValueError("interval stop is outside the signed 64-bit range")
        if interval.start >= interval.stop:
            raise ValueError("each interval must have start < stop")

    ordered_indexes = sorted(
        range(len(intervals)),
        key=lambda index: (
            intervals[index].stop,
            intervals[index].start,
            index,
        ),
    )

    selected: list[int] = []
    previous_stop: int | None = None
    for index in ordered_indexes:
        interval = intervals[index]
        if previous_stop is None or interval.start >= previous_stop:
            selected.append(index)
            previous_stop = interval.stop
    return tuple(selected)
```

## Example

```python
intervals = (
    HalfOpenInterval(5, 7),
    HalfOpenInterval(0, 3),
    HalfOpenInterval(3, 5),
    HalfOpenInterval(1, 2),
    HalfOpenInterval(2, 3),
    HalfOpenInterval(7, 9),
    HalfOpenInterval(2, 3),
)

selected_indexes = select_maximum_non_overlapping_intervals(intervals)
selected_intervals = tuple(intervals[index] for index in selected_indexes)

try:
    select_maximum_non_overlapping_intervals((HalfOpenInterval(False, 1),))
except TypeError:
    bool_endpoint_rejected = True
else:
    bool_endpoint_rejected = False

assert (
    selected_indexes,
    selected_intervals,
    select_maximum_non_overlapping_intervals(()),
    bool_endpoint_rejected,
) == (
    (3, 4, 2, 0, 5),
    (
        HalfOpenInterval(1, 2),
        HalfOpenInterval(2, 3),
        HalfOpenInterval(3, 5),
        HalfOpenInterval(5, 7),
        HalfOpenInterval(7, 9),
    ),
    (),
    True,
)
```

## Trade-offs and Limitations

Complete validation takes `O(n)` time, sorting takes `O(n log n)` time, and the
ordered indexes plus result use `O(n)` memory. The earliest-finish rule produces
a maximum-cardinality selection for unweighted intervals; it does not enumerate
every optimal selection.

The result is deterministic for one fixed tuple, but permuting intervals can
change which equal candidates win the declaration-index tie. Duplicate
intervals remain separate inputs even though at most one overlapping duplicate
can be selected. Empty intervals, closed intervals, floating endpoints,
weighted scheduling, dynamic updates, and multi-resource scheduling are outside
the contract.

## Related Snippets

<!-- catalog:related:start -->
- [Coalesce Bounded Half-Open Integer Intervals Under an Explicit Adjacency Policy](coalesce-bounded-half-open-integer-intervals-under-an-explicit-adjacency-policy.md)
- [Find a Point in Disjoint Half-Open Intervals](find-a-point-in-disjoint-half-open-intervals.md)
- [Cover a Half-Open Integer Range with Dyadic Intervals](cover-a-half-open-integer-range-with-dyadic-intervals.md)
<!-- catalog:related:end -->
