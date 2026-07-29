---
title: "Find Peak Coverage Spans Across Bounded Half-Open Integer Intervals"
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
  - select-a-maximum-cardinality-set-of-non-overlapping-half-open-integer-intervals.md
  - select-a-maximum-weight-set-of-non-overlapping-half-open-integer-intervals.md
---

# Find Peak Coverage Spans Across Bounded Half-Open Integer Intervals

## Idea and Problem

Find the greatest number of simultaneously active half-open intervals and every maximal span on which that coverage count holds.

A sweep applies the aggregate endpoint delta at one coordinate before examining
the elementary span to the next coordinate. Aggregating first preserves
half-open semantics when some intervals stop exactly where others start, while
merging adjacent peak spans removes artificial boundaries between equal counts.

## When to Use

Use this algorithm for a bounded, in-memory snapshot of intervals whose signed
integer endpoints share one scale. It is useful for finding concurrency hot
spots or checking an observed peak against a separate capacity limit while
preserving every disjoint location of that peak.

Use interval union when only covered extent matters, or interval scheduling
when the goal is to select compatible work. This routine counts duplicate
intervals separately and deliberately does not preserve interval identities.

## Implementation

```python
from dataclasses import dataclass

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_INTERVALS = 65_536


@dataclass(frozen=True, slots=True)
class HalfOpenInterval:
    start: int
    stop: int


@dataclass(frozen=True, slots=True)
class PeakCoverage:
    peak_count: int
    spans: tuple[HalfOpenInterval, ...]


def find_peak_coverage(
    intervals: tuple[HalfOpenInterval, ...],
) -> PeakCoverage:
    if type(intervals) is not tuple:
        raise TypeError("intervals must be an exact tuple")
    if not 1 <= len(intervals) <= _MAX_INTERVALS:
        raise ValueError("interval count is outside the supported range")

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

    events: list[tuple[int, int]] = []
    for interval in intervals:
        events.append((interval.start, 1))
        events.append((interval.stop, -1))
    events.sort()

    active_count = 0
    peak_count = 0
    peak_spans: list[HalfOpenInterval] = []
    event_index = 0
    while event_index < len(events):
        coordinate = events[event_index][0]
        aggregate_delta = 0
        while event_index < len(events) and events[event_index][0] == coordinate:
            aggregate_delta += events[event_index][1]
            event_index += 1

        active_count += aggregate_delta
        if event_index == len(events):
            break
        next_coordinate = events[event_index][0]

        if active_count > peak_count:
            peak_count = active_count
            peak_spans = [HalfOpenInterval(coordinate, next_coordinate)]
        elif active_count == peak_count:
            if peak_spans and peak_spans[-1].stop == coordinate:
                peak_spans[-1] = HalfOpenInterval(
                    peak_spans[-1].start,
                    next_coordinate,
                )
            else:
                peak_spans.append(HalfOpenInterval(coordinate, next_coordinate))

    return PeakCoverage(peak_count, tuple(peak_spans))
```

## Example

```python
intervals = (
    HalfOpenInterval(0, 2),
    HalfOpenInterval(0, 1),
    HalfOpenInterval(1, 2),
    HalfOpenInterval(4, 6),
    HalfOpenInterval(4, 6),
)

coverage = find_peak_coverage(intervals)
permuted = find_peak_coverage(tuple(reversed(intervals)))

assert coverage == PeakCoverage(
    peak_count=2,
    spans=(HalfOpenInterval(0, 2), HalfOpenInterval(4, 6)),
)
assert permuted == coverage
```

## Trade-offs and Limitations

Validation is complete before any endpoint events are created. Sorting at most
`2n` events takes `O(n log n)` time, and the event list plus output use `O(n)`
memory. The result is frozen, ascending, disjoint, non-empty, and has a positive
peak because the contract requires at least one non-empty interval.

The sweep measures coverage depth, not the identities responsible for it.
Weighted coverage, point-only endpoints, dynamic updates, calendars, capacity
assignment, resource selection, and conversions from closed or floating-point
intervals are outside this contract.

## Related Snippets

<!-- catalog:related:start -->
- [Coalesce Bounded Half-Open Integer Intervals Under an Explicit Adjacency Policy](coalesce-bounded-half-open-integer-intervals-under-an-explicit-adjacency-policy.md)
- [Select a Maximum-Cardinality Set of Non-Overlapping Half-Open Integer Intervals](select-a-maximum-cardinality-set-of-non-overlapping-half-open-integer-intervals.md)
- [Select a Maximum-Weight Set of Non-Overlapping Half-Open Integer Intervals](select-a-maximum-weight-set-of-non-overlapping-half-open-integer-intervals.md)
<!-- catalog:related:end -->
