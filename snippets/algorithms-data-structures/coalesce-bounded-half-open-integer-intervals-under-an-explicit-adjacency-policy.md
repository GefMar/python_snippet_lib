---
title: "Coalesce Bounded Half-Open Integer Intervals Under an Explicit Adjacency Policy"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - find-a-point-in-disjoint-half-open-intervals.md
  - cover-a-half-open-integer-range-with-dyadic-intervals.md
  - read-a-bounded-range-from-non-overlapping-byte-segments.md
---

# Coalesce Bounded Half-Open Integer Intervals Under an Explicit Adjacency Policy

## Idea and Problem

Validate, sort, and coalesce a bounded tuple of half-open integer intervals while making the treatment of touching boundaries explicit.

Overlapping intervals always merge. Intervals such as `[1, 4)` and `[4, 7)`
merge only when the selected policy treats adjacency as continuity. Sorting by
both endpoints makes the result independent of input order, while complete
validation before sorting prevents malformed inputs from producing a partial
result.

## When to Use

Use this algorithm to normalize an in-memory collection of signed 64-bit
integer ranges before coverage checks, indexing, or storage. It handles
duplicates and intervals contained inside larger intervals, accepts an empty
tuple, and never mutates the input.

Choose the adjacency policy from the meaning of the ranges. Keep touching
intervals separate when their boundary preserves a meaningful partition; join
them when only continuous coverage matters.

## Implementation

```python
from dataclasses import dataclass
from enum import StrEnum

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_INTERVALS = 10_000


@dataclass(frozen=True, slots=True)
class Interval:
    start: int
    stop: int


class AdjacencyPolicy(StrEnum):
    OVERLAPS_ONLY = "overlaps-only"
    OVERLAPS_OR_TOUCHES = "overlaps-or-touches"


def coalesce_intervals(
    intervals: tuple[Interval, ...],
    *,
    policy: AdjacencyPolicy,
) -> tuple[Interval, ...]:
    if type(intervals) is not tuple:
        raise TypeError("intervals must be an exact tuple")
    if len(intervals) > _MAX_INTERVALS:
        raise ValueError("intervals exceeds the supported limit")
    if type(policy) is not AdjacencyPolicy:
        raise TypeError("policy must be an AdjacencyPolicy")

    for interval in intervals:
        if type(interval) is not Interval:
            raise TypeError("each interval must be an exact Interval")
        if type(interval.start) is not int or type(interval.stop) is not int:
            raise TypeError("interval endpoints must be exact integers")
        if not _MIN_INT64 <= interval.start <= _MAX_INT64:
            raise ValueError("interval start is outside the signed 64-bit range")
        if not _MIN_INT64 <= interval.stop <= _MAX_INT64:
            raise ValueError("interval stop is outside the signed 64-bit range")
        if interval.start >= interval.stop:
            raise ValueError("each interval must have start < stop")

    if not intervals:
        return ()

    ordered = sorted(intervals, key=lambda interval: (interval.start, interval.stop))
    merged: list[Interval] = []
    current_start = ordered[0].start
    current_stop = ordered[0].stop
    merge_touches = policy is AdjacencyPolicy.OVERLAPS_OR_TOUCHES

    for interval in ordered[1:]:
        overlaps = interval.start < current_stop
        touches = interval.start == current_stop
        if overlaps or (merge_touches and touches):
            current_stop = max(current_stop, interval.stop)
            continue
        merged.append(Interval(current_start, current_stop))
        current_start = interval.start
        current_stop = interval.stop

    merged.append(Interval(current_start, current_stop))
    return tuple(merged)
```

## Example

```python
provided = (
    Interval(5, 8),
    Interval(1, 4),
    Interval(3, 6),
    Interval(1, 4),
    Interval(2, 3),
    Interval(8, 10),
    Interval(12, 14),
)

overlaps_only = coalesce_intervals(
    provided,
    policy=AdjacencyPolicy.OVERLAPS_ONLY,
)
overlaps_or_touches = coalesce_intervals(
    provided,
    policy=AdjacencyPolicy.OVERLAPS_OR_TOUCHES,
)

try:
    coalesce_intervals(
        (Interval(False, 3),),
        policy=AdjacencyPolicy.OVERLAPS_ONLY,
    )
except TypeError:
    bool_endpoint_rejected = True
else:
    bool_endpoint_rejected = False

try:
    coalesce_intervals(provided, policy="overlaps-only")  # type: ignore[arg-type]
except TypeError:
    raw_policy_rejected = True
else:
    raw_policy_rejected = False

try:
    coalesce_intervals(
        tuple(Interval(index, index + 1) for index in range(_MAX_INTERVALS + 1)),
        policy=AdjacencyPolicy.OVERLAPS_ONLY,
    )
except ValueError:
    oversized_input_rejected = True
else:
    oversized_input_rejected = False

assert (
    overlaps_only,
    overlaps_or_touches,
    provided,
    coalesce_intervals((), policy=AdjacencyPolicy.OVERLAPS_ONLY),
    bool_endpoint_rejected,
    raw_policy_rejected,
    oversized_input_rejected,
) == (
    (Interval(1, 8), Interval(8, 10), Interval(12, 14)),
    (Interval(1, 10), Interval(12, 14)),
    (
        Interval(5, 8),
        Interval(1, 4),
        Interval(3, 6),
        Interval(1, 4),
        Interval(2, 3),
        Interval(8, 10),
        Interval(12, 14),
    ),
    (),
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

Validation and sorting take `O(n log n)` time, and the ordered input plus
output require `O(n)` memory. The fixed input limit keeps both costs bounded.
Coalescing discards source order, multiplicity, and provenance; add payload
aggregation explicitly when those details matter.

The algorithm supports only non-empty half-open intervals with signed 64-bit
integer endpoints. Its adjacency rule applies only to exactly equal
boundaries. It provides neither interval-tree queries nor incremental updates,
and converting closed or floating-point ranges requires a separate domain
policy.

## Related Snippets

<!-- catalog:related:start -->
- [Find a Point in Disjoint Half-Open Intervals](find-a-point-in-disjoint-half-open-intervals.md)
- [Cover a Half-Open Integer Range with Dyadic Intervals](cover-a-half-open-integer-range-with-dyadic-intervals.md)
- [Read a Bounded Range from Non-Overlapping Byte Segments](read-a-bounded-range-from-non-overlapping-byte-segments.md)
<!-- catalog:related:end -->
