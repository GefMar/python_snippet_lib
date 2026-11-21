---
title: "Sum Signed Integer Loads Across Sparse Half-Open Intervals"
snippet_type: algorithm
use_cases:
  - data-transformation
  - observability
tested_python:
  - "3.14"
dependencies: []
related:
  - find-peak-coverage-spans-across-bounded-half-open-integer-intervals.md
  - maintain-bounded-range-additions-and-leftmost-minima-with-a-lazy-segment-tree.md
---

# Sum Signed Integer Loads Across Sparse Half-Open Intervals

## Idea and Problem

Build a piecewise-constant total load from overlapping intervals without allocating a dense timeline.

Every interval adds its load at the start and removes it at the end. Sum
these changes at each boundary, then sweep boundaries in sorted order.
A net-zero change creates no new span. Return only maximal spans with a
nonzero total.

## When to Use

Use this for a finite batch of activity windows and signed corrections on
one integer timeline. Supply `(start, end, load)` triples. Each load applies
on `[start, end)`, and repeated intervals each contribute independently.
Empty intervals and zero loads contribute nothing.

## Implementation

```python
from collections import defaultdict
from collections.abc import Iterable
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class LoadSpan:
    start: int
    end: int
    load: int


def sum_interval_loads(intervals: Iterable[tuple[int, int, int]]) -> tuple[LoadSpan, ...]:
    changes: dict[int, int] = defaultdict(int)
    for start, end, load in intervals:
        if type(start) is not int or type(end) is not int or type(load) is not int:
            raise TypeError("boundaries and loads must be integers")
        if start > end:
            raise ValueError("end must not precede start")
        if start < end and load:
            changes[start] += load
            changes[end] -= load

    spans: list[LoadSpan] = []
    previous: int | None = None
    current = 0
    for tick, change in sorted(changes.items()):
        if not change:
            continue
        if previous is not None and current:
            spans.append(LoadSpan(previous, tick, current))
        current += change
        previous = tick
    return tuple(spans)
```

## Example

```python
intervals = ((0, 4, 3), (2, 5, -1), (4, 6, 3), (-3, -1, -2))
cancelling = ((9, 10, 7), (9, 10, -7))

assert (sum_interval_loads(intervals), sum_interval_loads(cancelling)) == (
    (LoadSpan(-3, -1, -2), LoadSpan(0, 2, 3), LoadSpan(2, 5, 2), LoadSpan(5, 6, 3)),
    (),
)
```

## Trade-offs and Limitations

The function stores one aggregated change per distinct boundary and sorts
those boundaries. Work depends on the number of intervals and boundaries,
even if adjacent boundaries are very far apart. Omitted spans have load zero.

This sweep uses a complete batch of intervals. For repeated changes or
queries against an evolving set of intervals, choose an index that supports
the required updates. Retain the original intervals separately if a report
must identify which ones produced each total.

## Related Snippets

<!-- catalog:related:start -->
- [Find Peak Coverage Spans Across Bounded Half-Open Integer Intervals](find-peak-coverage-spans-across-bounded-half-open-integer-intervals.md)
- [Maintain Bounded Range Additions and Leftmost Minima with a Lazy Segment Tree](maintain-bounded-range-additions-and-leftmost-minima-with-a-lazy-segment-tree.md)
<!-- catalog:related:end -->
