---
title: "Subtract One Canonical Half-Open Integer Interval Set from Another"
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
  - find-peak-coverage-spans-across-bounded-half-open-integer-intervals.md
---

# Subtract One Canonical Half-Open Integer Interval Set from Another

## Idea and Problem

Subtract one canonical union of half-open integer intervals from another without expanding either union into individual integers.

Because both inputs are already sorted, disjoint, and non-adjacent, one
forward pointer can discard right-hand intervals that have ended and retain
one that continues across a gap in the left-hand set. Each overlap either
emits the uncovered prefix of a left interval or advances its remaining start.

## When to Use

Use this algorithm when two bounded in-memory interval sets have already been
normalized and the exact spans covered only by the left set are required. It
fits availability windows, allocated integer ranges, byte extents, and other
domains where half-open signed-integer boundaries are already the canonical
representation.

Normalize arbitrary, overlapping, touching, or unsorted inputs before this
operation. Keeping normalization separate makes the subtraction sweep linear
and rejects ambiguous representations rather than silently repairing them.

## Implementation

```python
from dataclasses import dataclass

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_INTERVALS_PER_SET = 50_000


@dataclass(frozen=True, slots=True)
class Interval:
    start: int
    stop: int


def _validate_canonical_interval_set(
    value: object,
    *,
    name: str,
) -> tuple[Interval, ...]:
    if type(value) is not tuple:
        raise TypeError(f"{name} must be an exact tuple")
    if len(value) > _MAX_INTERVALS_PER_SET:
        raise ValueError(f"{name} exceeds the supported interval limit")

    previous_stop: int | None = None
    for index, interval in enumerate(value):
        if type(interval) is not Interval:
            raise TypeError(f"{name}[{index}] must be an exact Interval")
        if type(interval.start) is not int or type(interval.stop) is not int:
            raise TypeError(f"{name}[{index}] endpoints must be exact integers")
        if not _MIN_INT64 <= interval.start <= _MAX_INT64:
            raise ValueError(f"{name}[{index}].start is outside signed 64-bit range")
        if not _MIN_INT64 <= interval.stop <= _MAX_INT64:
            raise ValueError(f"{name}[{index}].stop is outside signed 64-bit range")
        if interval.start >= interval.stop:
            raise ValueError(f"{name}[{index}] must be non-empty")
        if previous_stop is not None and previous_stop >= interval.start:
            raise ValueError(f"{name} must be sorted, disjoint, and non-adjacent")
        previous_stop = interval.stop

    return value


def subtract_interval_sets(
    left: tuple[Interval, ...],
    right: tuple[Interval, ...],
) -> tuple[Interval, ...]:
    """Return the canonical interval union covered by left but not right."""
    left_intervals = _validate_canonical_interval_set(left, name="left")
    right_intervals = _validate_canonical_interval_set(right, name="right")

    result: list[Interval] = []
    right_index = 0

    for minuend in left_intervals:
        while (
            right_index < len(right_intervals)
            and right_intervals[right_index].stop <= minuend.start
        ):
            right_index += 1

        remaining_start = minuend.start
        while (
            right_index < len(right_intervals) and right_intervals[right_index].start < minuend.stop
        ):
            subtrahend = right_intervals[right_index]
            if subtrahend.start > remaining_start:
                result.append(Interval(remaining_start, subtrahend.start))

            if subtrahend.stop >= minuend.stop:
                remaining_start = minuend.stop
                if subtrahend.stop == minuend.stop:
                    right_index += 1
                break

            remaining_start = max(remaining_start, subtrahend.stop)
            right_index += 1

        if remaining_start < minuend.stop:
            result.append(Interval(remaining_start, minuend.stop))

    return tuple(result)
```

## Example

```python
def canonical_from_integers(values: set[int]) -> tuple[Interval, ...]:
    ordered = sorted(values)
    if not ordered:
        return ()

    result = []
    start = previous = ordered[0]
    for value in ordered[1:]:
        if value != previous + 1:
            result.append(Interval(start, previous + 1))
            start = value
        previous = value
    result.append(Interval(start, previous + 1))
    return tuple(result)


def expand(intervals: tuple[Interval, ...]) -> set[int]:
    return {value for interval in intervals for value in range(interval.start, interval.stop)}


universe = tuple(range(-2, 3))
checked = 0
for left_mask in range(1 << len(universe)):
    left_values = {value for index, value in enumerate(universe) if left_mask & (1 << index)}
    left = canonical_from_integers(left_values)
    for right_mask in range(1 << len(universe)):
        right_values = {value for index, value in enumerate(universe) if right_mask & (1 << index)}
        right = canonical_from_integers(right_values)
        expected = canonical_from_integers(left_values - right_values)
        assert subtract_interval_sets(left, right) == expected
        checked += 1

split = subtract_interval_sets(
    (Interval(0, 12), Interval(20, 24)),
    (Interval(-3, 2), Interval(4, 7), Interval(9, 22)),
)
signed_limits = subtract_interval_sets(
    (Interval(_MIN_INT64, _MAX_INT64),),
    (
        Interval(_MIN_INT64, _MIN_INT64 + 1),
        Interval(_MAX_INT64 - 1, _MAX_INT64),
    ),
)

maximum_left = tuple(Interval(4 * index, 4 * index + 3) for index in range(_MAX_INTERVALS_PER_SET))
maximum_right = tuple(
    Interval(4 * index + 1, 4 * index + 2) for index in range(_MAX_INTERVALS_PER_SET)
)
maximum_result = subtract_interval_sets(maximum_left, maximum_right)

try:
    subtract_interval_sets(
        (Interval(0, 1), Interval(1, 2)),
        (),
    )
except ValueError:
    touching_rejected = True
else:
    touching_rejected = False

try:
    subtract_interval_sets((Interval(False, 1),), ())
except TypeError:
    boolean_rejected = True
else:
    boolean_rejected = False

assert (
    checked,
    split,
    signed_limits,
    len(maximum_result),
    maximum_result[:2],
    maximum_result[-2:],
    touching_rejected,
    boolean_rejected,
) == (
    1_024,
    (
        Interval(2, 4),
        Interval(7, 9),
        Interval(22, 24),
    ),
    (Interval(_MIN_INT64 + 1, _MAX_INT64 - 1),),
    100_000,
    (Interval(0, 1), Interval(2, 3)),
    (Interval(199_996, 199_997), Interval(199_998, 199_999)),
    True,
    True,
)
```

## Trade-offs and Limitations

Validation and the sweep take `O(n + m + r)` time for `n` left intervals,
`m` right intervals, and `r` returned intervals. The sweep uses `O(1)`
auxiliary state, while the immutable result uses `O(r)` memory. A right span
that crosses several left spans remains at the shared pointer until it ends;
the number of emitted spans is at most `n + m`.

The inputs must already be canonical and use exact signed 64-bit integer
endpoints. The function retains no interval provenance and does not handle
closed ranges, floating-point boundaries, payload aggregation, incremental
updates, or interval-tree queries. Coalesce or otherwise normalize data under
an explicit domain policy before subtraction.

## Related Snippets

<!-- catalog:related:start -->
- [Coalesce Bounded Half-Open Integer Intervals Under an Explicit Adjacency Policy](coalesce-bounded-half-open-integer-intervals-under-an-explicit-adjacency-policy.md)
- [Find a Point in Disjoint Half-Open Intervals](find-a-point-in-disjoint-half-open-intervals.md)
- [Find Peak Coverage Spans Across Bounded Half-Open Integer Intervals](find-peak-coverage-spans-across-bounded-half-open-integer-intervals.md)
<!-- catalog:related:end -->
