---
title: "Interpolate Increasing Integer Time-Series Points Exactly"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - join-bounded-strictly-increasing-sequences-by-the-latest-prior-timestamp.md
  - overlay-aligned-time-series-at-the-finest-step.md
  - integrate-regular-rate-samples-across-explicit-time-buckets.md
---

# Interpolate Increasing Integer Time-Series Points Exactly

## Idea and Problem

Evaluate a bounded piecewise-linear integer time series at increasing query ticks without introducing floating-point rounding.

Every query lies in one source segment or exactly on a declared source tick.
Because source and query ticks both strictly increase, one forward cursor can
locate all segments without repeating a search. Integer-weighted endpoints and
`Fraction` keep each interpolated value exact, even when intermediate products
exceed the signed 64-bit input range.

## When to Use

Use this algorithm for one already sorted, in-memory series when ticks and
values are exact integers, query ticks are known in advance, and reproducible
linear interpolation matters. Irregular source spacing is allowed, and a query
on a source tick returns the declared value with denominator one.

Choose a binary search per query when queries are not sorted or arrive
independently. Use a numerical or time-series library for floating-point
arrays, missing-value policies, extrapolation, spline interpolation,
shape-preserving curves, calendar-aware timestamps, or implicit resampling.

## Implementation

```python
from fractions import Fraction

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_SERIES_POINTS = 100_000
_MAX_QUERY_TICKS = 100_000


def interpolate_integer_time_series(
    points: tuple[tuple[int, int], ...],
    query_ticks: tuple[int, ...],
) -> tuple[Fraction, ...]:
    """Return exact piecewise-linear values at increasing query ticks."""
    if type(points) is not tuple:
        raise TypeError("points must be an exact tuple")
    if not 2 <= len(points) <= _MAX_SERIES_POINTS:
        raise ValueError("point count is outside the supported range")

    previous_tick: int | None = None
    for index, point in enumerate(points):
        if type(point) is not tuple:
            raise TypeError(f"points[{index}] must be an exact tuple")
        if len(point) != 2:
            raise ValueError(f"points[{index}] must contain two items")
        tick, value = point
        if type(tick) is not int:
            raise TypeError(f"points[{index}].tick must be an exact non-boolean integer")
        if type(value) is not int:
            raise TypeError(f"points[{index}].value must be an exact non-boolean integer")
        if not _MIN_INT64 <= tick <= _MAX_INT64:
            raise ValueError(f"points[{index}].tick is outside the signed 64-bit range")
        if not _MIN_INT64 <= value <= _MAX_INT64:
            raise ValueError(f"points[{index}].value is outside the signed 64-bit range")
        if previous_tick is not None and tick <= previous_tick:
            raise ValueError("point ticks must strictly increase")
        previous_tick = tick

    if type(query_ticks) is not tuple:
        raise TypeError("query_ticks must be an exact tuple")
    if len(query_ticks) > _MAX_QUERY_TICKS:
        raise ValueError("query count exceeds the supported limit")

    previous_query: int | None = None
    for index, query_tick in enumerate(query_ticks):
        if type(query_tick) is not int:
            raise TypeError(f"query_ticks[{index}] must be an exact non-boolean integer")
        if not _MIN_INT64 <= query_tick <= _MAX_INT64:
            raise ValueError(f"query_ticks[{index}] is outside the signed 64-bit range")
        if previous_query is not None and query_tick <= previous_query:
            raise ValueError("query ticks must strictly increase")
        previous_query = query_tick

    if query_ticks and (query_ticks[0] < points[0][0] or query_ticks[-1] > points[-1][0]):
        raise ValueError("query ticks must stay inside the closed source range")

    result: list[Fraction] = []
    segment_index = 0
    for query_tick in query_ticks:
        while segment_index + 1 < len(points) and points[segment_index + 1][0] < query_tick:
            segment_index += 1

        left_tick, left_value = points[segment_index]
        if query_tick == left_tick:
            result.append(Fraction(left_value))
            continue

        right_tick, right_value = points[segment_index + 1]
        if query_tick == right_tick:
            result.append(Fraction(right_value))
            continue

        numerator = left_value * (right_tick - query_tick) + right_value * (query_tick - left_tick)
        result.append(Fraction(numerator, right_tick - left_tick))

    return tuple(result)
```

## Example

```python
def interpolate_with_independent_search(
    points: tuple[tuple[int, int], ...],
    query_ticks: tuple[int, ...],
) -> tuple[Fraction, ...]:
    from bisect import bisect_right

    ticks = tuple(tick for tick, _ in points)
    expected: list[Fraction] = []
    for query_tick in query_ticks:
        left_index = bisect_right(ticks, query_tick) - 1
        left_tick, left_value = points[left_index]
        if query_tick == left_tick:
            expected.append(Fraction(left_value))
            continue
        right_tick, right_value = points[left_index + 1]
        expected.append(
            Fraction(left_value)
            + Fraction(
                (query_tick - left_tick) * (right_value - left_value),
                right_tick - left_tick,
            )
        )
    return tuple(expected)


def raises(error_type: type[Exception], operation: object) -> bool:
    try:
        operation()  # type: ignore[operator]
    except error_type:
        return True
    return False


points = ((-5, 9), (0, 0), (3, 2), (10, -5))
queries = (-5, -2, 0, 1, 3, 8, 10)
interpolated = interpolate_integer_time_series(points, queries)

extreme_points = ((_MIN_INT64, _MAX_INT64), (_MAX_INT64, _MIN_INT64))
extreme_queries = (_MIN_INT64, 0, _MAX_INT64)
extreme_result = interpolate_integer_time_series(extreme_points, extreme_queries)

large_points = tuple((index, index - 50_000) for index in range(_MAX_SERIES_POINTS))
large_queries = tuple(range(_MAX_QUERY_TICKS))
large_result = interpolate_integer_time_series(large_points, large_queries)

assert (
    interpolated,
    interpolated == interpolate_with_independent_search(points, queries),
    extreme_result,
    extreme_result == interpolate_with_independent_search(extreme_points, extreme_queries),
    interpolate_integer_time_series(points, ()),
    (len(large_result), large_result[0], large_result[-1]),
    raises(ValueError, lambda: interpolate_integer_time_series(((0, 0), (0, 1)), ())),
    raises(ValueError, lambda: interpolate_integer_time_series(points, (1, 0))),
    raises(ValueError, lambda: interpolate_integer_time_series(points, (-6,))),
    raises(TypeError, lambda: interpolate_integer_time_series(points, (True,))),
) == (
    (
        Fraction(9),
        Fraction(18, 5),
        Fraction(0),
        Fraction(2, 3),
        Fraction(2),
        Fraction(-3),
        Fraction(-5),
    ),
    True,
    (Fraction(_MAX_INT64), Fraction(-1), Fraction(_MIN_INT64)),
    True,
    (),
    (_MAX_QUERY_TICKS, Fraction(-50_000), Fraction(49_999)),
    True,
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

Validation visits all `S` source points and `Q` queries before interpolation.
The forward cursor then visits every relevant segment at most once, so the
complete operation uses `O(S + Q)` exact-integer operations. The result and
the temporary result list use `O(Q)` memory and briefly coexist when the outer
tuple is created.

Python integers prevent overflow, but products can exceed 64 bits and their
arithmetic is not constant-cost. Each `Fraction` also normalizes its numerator
and denominator with bigint greatest-common-divisor work, so bit complexity
depends on the declared magnitudes and tick gaps even though item counts are
linearly bounded.

The function accepts only complete integer snapshots and exact increasing
queries inside the source range. It does not sort, deduplicate, extrapolate,
fill missing values, accept floats, infer a regular grid, preserve a requested
global monotonic shape, evaluate splines, or attach timezone or calendar
semantics to integer ticks.

## Related Snippets

<!-- catalog:related:start -->
- [Join Bounded Strictly Increasing Sequences by the Latest Prior Timestamp](join-bounded-strictly-increasing-sequences-by-the-latest-prior-timestamp.md)
- [Overlay Aligned Time Series at the Finest Step](overlay-aligned-time-series-at-the-finest-step.md)
- [Integrate Regular Rate Samples Across Explicit Time Buckets](integrate-regular-rate-samples-across-explicit-time-buckets.md)
<!-- catalog:related:end -->
