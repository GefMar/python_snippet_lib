---
title: "Resolve Bounded Overlapping Integer Intervals by Priority into Canonical Winner Spans"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - audit-bounded-keyed-half-open-intervals-with-gap-and-overlap-evidence.md
  - ../algorithms-data-structures/coalesce-bounded-half-open-integer-intervals-under-an-explicit-adjacency-policy.md
  - ../algorithms-data-structures/find-a-point-in-disjoint-half-open-intervals.md
---

# Resolve Bounded Overlapping Integer Intervals by Priority into Canonical Winner Spans

## Idea and Problem

Resolve competing half-open integer intervals over one bounded domain by selecting exactly one winner at every covered position.

Lower numeric priority wins, and the lower original input index breaks equal
priority ties. A boundary sweep keeps that index as the stable identity of the
winner, emits explicit `None` gaps, and joins adjacent spans only when their
winner identities are equal. The result therefore covers the complete domain
without overlaps or avoidable boundaries.

## When to Use

Use this algorithm to turn a small in-memory set of precedence rules,
overrides, or scheduled assignments into a deterministic piecewise plan. It
is especially useful when downstream code needs direct winner indices and
must distinguish uncovered portions from portions owned by an interval.

Choose a different representation when equal-priority intervals must combine
their payloads, all contenders must be retained, or updates and point queries
arrive incrementally. This function resolves one immutable snapshot.

## Implementation

```python
from dataclasses import dataclass
from heapq import heappop, heappush
from itertools import pairwise

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_INTERVALS = 4_096


@dataclass(frozen=True, slots=True)
class Interval:
    start: int
    stop: int
    priority: int


@dataclass(frozen=True, slots=True)
class WinnerSpan:
    start: int
    stop: int
    winner_index: int | None


def _require_int64(value: object, *, name: str) -> int:
    if type(value) is not int:
        raise TypeError(f"{name} must be an exact integer")
    if not _MIN_INT64 <= value <= _MAX_INT64:
        raise ValueError(f"{name} is outside the signed 64-bit range")
    return value


def resolve_priority_intervals(
    intervals: tuple[Interval, ...],
    *,
    domain_start: int,
    domain_stop: int,
) -> tuple[WinnerSpan, ...]:
    domain_start = _require_int64(domain_start, name="domain_start")
    domain_stop = _require_int64(domain_stop, name="domain_stop")
    if domain_start >= domain_stop:
        raise ValueError("the domain must be non-empty")
    if type(intervals) is not tuple:
        raise TypeError("intervals must be an exact tuple")
    if len(intervals) > _MAX_INTERVALS:
        raise ValueError("intervals exceeds the supported limit")

    boundaries = {domain_start, domain_stop}
    for interval in intervals:
        if type(interval) is not Interval:
            raise TypeError("each item must be an exact Interval")
        start = _require_int64(interval.start, name="interval start")
        stop = _require_int64(interval.stop, name="interval stop")
        _require_int64(interval.priority, name="interval priority")
        if start >= stop:
            raise ValueError("each interval must be non-empty")
        if start < domain_start or stop > domain_stop:
            raise ValueError("each interval must be contained in the domain")
        boundaries.add(start)
        boundaries.add(stop)

    ordered = sorted(
        enumerate(intervals),
        key=lambda indexed: (indexed[1].start, indexed[0]),
    )
    ordered_boundaries = sorted(boundaries)
    active: list[tuple[int, int, int]] = []
    next_interval = 0
    spans: list[WinnerSpan] = []

    for start, stop in pairwise(ordered_boundaries):
        while next_interval < len(ordered) and ordered[next_interval][1].start == start:
            index, interval = ordered[next_interval]
            heappush(active, (interval.priority, index, interval.stop))
            next_interval += 1

        while active and active[0][2] <= start:
            heappop(active)

        winner_index = active[0][1] if active else None
        if spans and spans[-1].winner_index == winner_index:
            previous = spans[-1]
            spans[-1] = WinnerSpan(previous.start, stop, winner_index)
        else:
            spans.append(WinnerSpan(start, stop, winner_index))

    return tuple(spans)


```

## Example

```python
provided = (
    Interval(2, 8, 5),
    Interval(4, 6, 1),
    Interval(6, 10, 1),
    Interval(3, 7, 1),
    Interval(7, 9, 9),
)

resolved = resolve_priority_intervals(
    provided,
    domain_start=0,
    domain_stop=12,
)
empty = resolve_priority_intervals((), domain_start=-2, domain_stop=3)


def is_rejected(candidate: object, *, start: object = 0, stop: object = 12) -> bool:
    try:
        resolve_priority_intervals(
            candidate,  # type: ignore[arg-type]
            domain_start=start,  # type: ignore[arg-type]
            domain_stop=stop,  # type: ignore[arg-type]
        )
    except TypeError:
        return True
    except ValueError:
        return True
    return False


assert resolved == (
    WinnerSpan(0, 2, None),
    WinnerSpan(2, 3, 0),
    WinnerSpan(3, 4, 3),
    WinnerSpan(4, 6, 1),
    WinnerSpan(6, 10, 2),
    WinnerSpan(10, 12, None),
)
assert empty == (WinnerSpan(-2, 3, None),)
assert (
    len(resolved) <= 2 * len(provided) + 1,
    is_rejected([Interval(1, 2, 0)]),
    is_rejected((Interval(1, 1, 0),)),
    is_rejected((Interval(-1, 2, 0),)),
    is_rejected((Interval(1, 2, False),)),
    is_rejected((), start=False),
) == (True, True, True, True, True, True)
```

## Trade-offs and Limitations

For `n` intervals, sorting boundaries and starts plus heap maintenance takes
`O(n log n)` time and `O(n)` auxiliary memory. At most `2n` interval
endpoints split the containing domain, so the unmerged result has no more than
`2n + 1` spans; joining equal adjacent winners can only reduce that count.
Lazy heap removal may retain ended non-winning entries until they reach the
heap root, but the fixed input limit bounds this storage.

Inputs are exact immutable records with signed 64-bit integer endpoints and
priorities. Every interval must be non-empty and completely contained in the
domain. The output identifies a winner by its original zero-based input index;
it does not copy payloads, retain losing contenders, aggregate equal
priorities, or infer whether touching input intervals represent continuity.

## Related Snippets

<!-- catalog:related:start -->
- [Audit Bounded Keyed Half-Open Intervals with Gap and Overlap Evidence](audit-bounded-keyed-half-open-intervals-with-gap-and-overlap-evidence.md)
- [Coalesce Bounded Half-Open Integer Intervals Under an Explicit Adjacency Policy](../algorithms-data-structures/coalesce-bounded-half-open-integer-intervals-under-an-explicit-adjacency-policy.md)
- [Find a Point in Disjoint Half-Open Intervals](../algorithms-data-structures/find-a-point-in-disjoint-half-open-intervals.md)
<!-- catalog:related:end -->
