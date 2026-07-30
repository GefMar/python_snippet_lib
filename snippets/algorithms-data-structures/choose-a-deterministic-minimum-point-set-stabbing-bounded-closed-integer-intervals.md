---
title: "Choose a Deterministic Minimum Point Set Stabbing Bounded Closed Integer Intervals"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - select-a-maximum-cardinality-set-of-non-overlapping-half-open-integer-intervals.md
  - assign-bounded-half-open-integer-intervals-to-the-minimum-number-of-reusable-rooms.md
  - coalesce-bounded-half-open-integer-intervals-under-an-explicit-adjacency-policy.md
---

# Choose a Deterministic Minimum Point Set Stabbing Bounded Closed Integer Intervals

## Idea and Problem

Select the fewest integer points so that every closed interval contains at least one selected point.

Sort the intervals by their upper endpoint. Whenever the most recently chosen
point falls before the next interval, choose that interval's upper endpoint.
That point reaches as far right as possible while still covering the earliest-
ending uncovered interval.

The standard exchange argument proves optimality: every solution needs a point
inside the earliest-ending uncovered interval, and replacing that point with
the interval's upper endpoint cannot reduce coverage of any later-ending
interval.

## When to Use

Use this greedy algorithm when bounded one-dimensional requirements are closed
integer intervals and one point can satisfy every interval containing it. It
fits minimum checkpoint placement, representative-coordinate selection, and
small coverage plans whose endpoint inclusivity is explicit.

Do not use it for arbitrary set cover. Weights, open endpoints, circular
ranges, capacity limits, multiple required hits, or higher-dimensional regions
need different algorithms.

## Implementation

```python
_MAX_INTERVALS = 4_096
_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1


def minimum_closed_interval_stabbing_points(
    intervals: tuple[tuple[int, int], ...],
) -> tuple[int, ...]:
    if type(intervals) is not tuple:
        raise TypeError("intervals must be an exact tuple")
    if len(intervals) > _MAX_INTERVALS:
        raise ValueError("interval count exceeds the supported limit")

    checked: list[tuple[int, int]] = []
    for interval in intervals:
        if type(interval) is not tuple or len(interval) != 2:
            raise TypeError("each interval must be an exact two-item tuple")
        lower, upper = interval
        if type(lower) is not int or type(upper) is not int:
            raise TypeError("interval endpoints must be exact integers")
        if not (
            _MIN_INT64 <= lower <= _MAX_INT64
            and _MIN_INT64 <= upper <= _MAX_INT64
        ):
            raise ValueError("an interval endpoint is outside signed-64")
        if lower > upper:
            raise ValueError("a closed interval must satisfy lower <= upper")
        checked.append((lower, upper))

    selected: list[int] = []
    for lower, upper in sorted(checked, key=lambda item: (item[1], item[0])):
        if not selected or selected[-1] < lower:
            selected.append(upper)
    return tuple(selected)
```

## Example

```python
intervals = (
    (1, 4),
    (2, 3),
    (3, 6),
    (7, 9),
    (8, 10),
    (10, 10),
)

points = minimum_closed_interval_stabbing_points(intervals)
assert points == (3, 9, 10)
assert all(any(lower <= point <= upper for point in points) for lower, upper in intervals)

shared_boundary = ((1, 2), (2, 3))
assert minimum_closed_interval_stabbing_points(shared_boundary) == (2,)
assert minimum_closed_interval_stabbing_points(()) == ()
```

## Trade-offs and Limitations

For `n` intervals, sorting costs `O(n log n)` time and the checked copy plus
output use `O(n)` space. The returned points are strictly increasing because a
new upper endpoint is selected only after the preceding point falls before the
new interval's lower endpoint.

The result has minimum cardinality, but the right-endpoint rule does not promise
the lexicographically smallest optimum. For example, both point 2 and point 3
alone cover `[1, 3]` and `[2, 4]`, while this policy returns 3.

Closed endpoints are part of correctness: a selected point equal to the next
interval's lower endpoint already covers it. Converting half-open or continuous
intervals mechanically can change the problem, especially at signed limits.
Duplicate and singleton intervals are accepted and do not receive special
weight.

## Related Snippets

<!-- catalog:related:start -->
- [Select a Maximum-Cardinality Set of Non-Overlapping Half-Open Integer Intervals](select-a-maximum-cardinality-set-of-non-overlapping-half-open-integer-intervals.md)
- [Assign Bounded Half-Open Integer Intervals to the Minimum Number of Reusable Rooms](assign-bounded-half-open-integer-intervals-to-the-minimum-number-of-reusable-rooms.md)
- [Coalesce Bounded Half-Open Integer Intervals Under an Explicit Adjacency Policy](coalesce-bounded-half-open-integer-intervals-under-an-explicit-adjacency-policy.md)
<!-- catalog:related:end -->
