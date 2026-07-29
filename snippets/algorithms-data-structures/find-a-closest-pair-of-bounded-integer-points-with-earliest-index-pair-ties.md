---
title: "Find a Closest Pair of Bounded Integer Points with Earliest Index-Pair Ties"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-a-canonical-convex-hull-for-bounded-integer-points.md
  - match-strict-mutual-nearest-neighbors-with-a-comparison-budget.md
  - classify-the-exact-intersection-of-two-closed-integer-line-segments.md
---

# Find a Closest Pair of Bounded Integer Points with Earliest Index-Pair Ties

## Idea and Problem

Find two input positions whose integer points have the smallest squared Euclidean distance, resolving every equal-distance choice by the earliest index pair.

The public comparison key is `(squared_distance, first_index, second_index)`,
where `first_index < second_index`. Squared distances keep the calculation
exact and avoid a square root that cannot change the ordering.

After complete validation, an x-order scan detects equal-coordinate groups.
Any duplicate pair has the absolute minimum distance zero, so the two earliest
indexes in each duplicate group are sufficient to select the global answer.
For unique coordinates, divide and conquer finds each half's best pair and
checks only a narrow y-ordered strip across their boundary.

## When to Use

Use this algorithm for a static, bounded sequence of exact two-dimensional
integer points when original positions identify observations and quadratic
all-pairs comparison would be too expensive. It handles repeated coordinates
without losing their separate input identities.

Use direct pair enumeration for tiny inputs or as a test oracle. Choose a
geometry library when coordinates are floating point, tolerance or coordinate
reference system rules matter, dimensions exceed two, or the point set changes
incrementally. Use a spatial index for repeated nearest-neighbor queries rather
than one global closest-pair calculation.

## Implementation

```python
from dataclasses import dataclass

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_CLOSEST_POINT_COUNT = 20_000


@dataclass(frozen=True, slots=True)
class IntegerPoint:
    x: int
    y: int


@dataclass(frozen=True, slots=True)
class ClosestPointPair:
    first_index: int
    second_index: int
    squared_distance: int


_IndexedPoint = tuple[int, int, int]
_Candidate = tuple[int, int, int]


def _pair_candidate(first: _IndexedPoint, second: _IndexedPoint) -> _Candidate:
    first_index, second_index = sorted((first[2], second[2]))
    delta_x = first[0] - second[0]
    delta_y = first[1] - second[1]
    return delta_x * delta_x + delta_y * delta_y, first_index, second_index


def _closest_unique_pair(
    ordered_by_x: tuple[_IndexedPoint, ...],
    ordered_by_y: tuple[_IndexedPoint, ...],
) -> _Candidate:
    point_count = len(ordered_by_x)
    if point_count <= 3:
        return min(
            _pair_candidate(ordered_by_x[first], ordered_by_x[second])
            for first in range(point_count)
            for second in range(first + 1, point_count)
        )

    midpoint = point_count // 2
    left_by_x = ordered_by_x[:midpoint]
    right_by_x = ordered_by_x[midpoint:]
    left_indexes = {point[2] for point in left_by_x}
    left_by_y = tuple(point for point in ordered_by_y if point[2] in left_indexes)
    right_by_y = tuple(point for point in ordered_by_y if point[2] not in left_indexes)

    best = min(
        _closest_unique_pair(left_by_x, left_by_y),
        _closest_unique_pair(right_by_x, right_by_y),
    )
    dividing_x = right_by_x[0][0]
    strip = tuple(
        point
        for point in ordered_by_y
        if (point[0] - dividing_x) * (point[0] - dividing_x) <= best[0]
    )

    for first_position, first in enumerate(strip):
        for second_position in range(first_position + 1, len(strip)):
            second = strip[second_position]
            delta_y = second[1] - first[1]
            if delta_y * delta_y > best[0]:
                break
            best = min(best, _pair_candidate(first, second))
    return best


def find_closest_integer_point_pair(
    points: tuple[IntegerPoint, ...],
) -> ClosestPointPair:
    """Return the closest pair under squared-distance and input-index ties."""
    if type(points) is not tuple:
        raise TypeError("points must be an exact tuple")
    if not 2 <= len(points) <= _MAX_CLOSEST_POINT_COUNT:
        raise ValueError("point count is outside the supported range")

    for index, point in enumerate(points):
        if type(point) is not IntegerPoint:
            raise TypeError(f"points[{index}] must be an exact IntegerPoint")
        if type(point.x) is not int or type(point.y) is not int:
            raise TypeError(f"points[{index}] coordinates must be exact integers")
        if not _MIN_INT64 <= point.x <= _MAX_INT64:
            raise ValueError(f"points[{index}].x is outside the signed 64-bit range")
        if not _MIN_INT64 <= point.y <= _MAX_INT64:
            raise ValueError(f"points[{index}].y is outside the signed 64-bit range")

    indexed = tuple((point.x, point.y, index) for index, point in enumerate(points))
    ordered_by_x = tuple(sorted(indexed))

    duplicate_best: tuple[int, int] | None = None
    group_start = 0
    while group_start < len(ordered_by_x):
        group_stop = group_start + 1
        coordinates = ordered_by_x[group_start][:2]
        while group_stop < len(ordered_by_x) and ordered_by_x[group_stop][:2] == coordinates:
            group_stop += 1
        if group_stop - group_start >= 2:
            candidate = (
                ordered_by_x[group_start][2],
                ordered_by_x[group_start + 1][2],
            )
            duplicate_best = candidate if duplicate_best is None else min(duplicate_best, candidate)
        group_start = group_stop

    if duplicate_best is not None:
        return ClosestPointPair(*duplicate_best, squared_distance=0)

    ordered_by_y = tuple(sorted(indexed, key=lambda point: (point[1], point[0], point[2])))
    squared_distance, first_index, second_index = _closest_unique_pair(
        ordered_by_x,
        ordered_by_y,
    )
    return ClosestPointPair(first_index, second_index, squared_distance)
```

## Example

```python
def enumerate_closest_pair(
    points: tuple[IntegerPoint, ...],
) -> ClosestPointPair:
    best = min(
        (
            (points[first].x - points[second].x) ** 2 + (points[first].y - points[second].y) ** 2,
            first,
            second,
        )
        for first in range(len(points))
        for second in range(first + 1, len(points))
    )
    return ClosestPointPair(best[1], best[2], best[0])


def exercise_small_point_sequences() -> int:
    from itertools import product

    grid = (
        IntegerPoint(0, 0),
        IntegerPoint(0, 1),
        IntegerPoint(1, 0),
        IntegerPoint(1, 1),
    )
    checked = 0
    for point_count in range(2, 6):
        for points in product(grid, repeat=point_count):
            assert find_closest_integer_point_pair(points) == enumerate_closest_pair(points)
            checked += 1
    return checked


equal_distance = (
    IntegerPoint(10, 10),
    IntegerPoint(0, 0),
    IntegerPoint(1, 0),
    IntegerPoint(10, 11),
)
duplicate_groups = (
    IntegerPoint(0, 0),
    IntegerPoint(5, 5),
    IntegerPoint(8, 8),
    IntegerPoint(5, 5),
    IntegerPoint(0, 0),
)
maximum_count = tuple(IntegerPoint(index, 0) for index in range(_MAX_CLOSEST_POINT_COUNT))
extreme = (IntegerPoint(_MIN_INT64, _MIN_INT64), IntegerPoint(_MAX_INT64, _MAX_INT64))

type_errors = 0
for invalid in ([IntegerPoint(0, 0), IntegerPoint(1, 1)], (IntegerPoint(True, 0),) * 2):
    try:
        find_closest_integer_point_pair(invalid)  # type: ignore[arg-type]
    except TypeError:
        type_errors += 1

try:
    find_closest_integer_point_pair((IntegerPoint(0, 0),) * (_MAX_CLOSEST_POINT_COUNT + 1))
except ValueError:
    oversized_rejected = True
else:
    oversized_rejected = False

assert (
    exercise_small_point_sequences(),
    find_closest_integer_point_pair(equal_distance),
    find_closest_integer_point_pair(duplicate_groups),
    find_closest_integer_point_pair(maximum_count),
    find_closest_integer_point_pair(extreme),
    type_errors,
    oversized_rejected,
) == (
    1_360,
    ClosestPointPair(0, 3, 1),
    ClosestPointPair(0, 4, 0),
    ClosestPointPair(0, 1, 1),
    ClosestPointPair(0, 1, 2 * ((1 << 64) - 1) ** 2),
    2,
    True,
)
```

## Trade-offs and Limitations

Validation and the two initial orderings take `O(n log n)` time. Each recursive
level partitions an existing y-order and scans a strip in linear time. Unique
coordinates and the current within-half distance give the strip its constant
packing bound, including candidates at exactly the current best distance. The
whole calculation therefore takes `O(n log n)` time and `O(n)` peak auxiliary
memory; the immutable result uses `O(1)` memory.

Duplicate-coordinate groups return before the y-order and recursion because
no positive distance can improve on zero. The x-order includes the original
index, so the first two members of one coordinate group are its earliest index
pair. Python integers keep coordinate differences, squares, and their sum exact
even when signed 64-bit input coordinates produce a result wider than 64 bits.

Input order is semantically significant because indexes identify observations.
The function does not return every tied pair, mutate or deduplicate the input,
accept floating-point coordinates, apply tolerance or coordinate-system rules,
support dimensions other than two, maintain a dynamic point set, or answer a
nearest-neighbor query for every point.

## Related Snippets

<!-- catalog:related:start -->
- [Compute a Canonical Convex Hull for Bounded Integer Points](compute-a-canonical-convex-hull-for-bounded-integer-points.md)
- [Match Strict Mutual Nearest Neighbors with a Comparison Budget](match-strict-mutual-nearest-neighbors-with-a-comparison-budget.md)
- [Classify the Exact Intersection of Two Closed Integer Line Segments](classify-the-exact-intersection-of-two-closed-integer-line-segments.md)
<!-- catalog:related:end -->
