---
title: "Simplify a Bounded Integer Polyline with Exact Ramer-Douglas-Peucker Distance Comparisons"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - classify-the-exact-intersection-of-two-closed-integer-line-segments.md
  - compute-a-canonical-convex-hull-for-bounded-integer-points.md
  - ../data-processing/downsample-fixed-width-integer-tick-buckets-by-first-minimum-maximum-and-last.md
---

# Simplify a Bounded Integer Polyline with Exact Ramer-Douglas-Peucker Distance Comparisons

## Idea and Problem

Remove redundant polyline vertices while comparing every point-to-segment distance without floating-point rounding.

Ramer-Douglas-Peucker simplification starts with the full endpoint span. If
its farthest interior point lies beyond the tolerance, that point is retained
and the two resulting spans are examined in the same way. Otherwise every
interior point of the span can be removed under this algorithm's rule.

Squared distance to a finite segment can be represented as an integer ratio.
Endpoint projections have denominator one, while an interior projection uses
the squared segment length. Cross multiplication therefore decides maxima and
threshold equality exactly, even when the perpendicular foot is fractional.

## When to Use

Use this algorithm for a bounded open 2-D polyline with exact integer
coordinates when reproducible tolerance decisions matter more than preserving
every sample. It fits review fixtures, compact plotting data, and deterministic
preprocessing where input order already describes the path.

Use a topology-aware or domain-specific simplifier for polygons, geographic
coordinates, obstacle boundaries, or curves whose self-intersections must be
preserved. Use a streaming approximation when the complete polyline cannot be
retained.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction
from itertools import product

_MAX_POLYLINE_POINTS = 2_048
_MIN_COORDINATE = -(2**31)
_MAX_COORDINATE = 2**31 - 1
_MAX_TOLERANCE_SQUARED = 2**63 - 1

Point = tuple[int, int]


@dataclass(frozen=True, slots=True)
class PolylineSimplification:
    retained_indexes: tuple[int, ...]
    retained_points: tuple[Point, ...]


def _point_segment_distance_squared_ratio(
    point: Point,
    start: Point,
    end: Point,
) -> tuple[int, int]:
    segment_x = end[0] - start[0]
    segment_y = end[1] - start[1]
    offset_x = point[0] - start[0]
    offset_y = point[1] - start[1]
    segment_squared = segment_x * segment_x + segment_y * segment_y

    if segment_squared == 0:
        return offset_x * offset_x + offset_y * offset_y, 1

    projection = offset_x * segment_x + offset_y * segment_y
    if projection <= 0:
        return offset_x * offset_x + offset_y * offset_y, 1
    if projection >= segment_squared:
        end_offset_x = point[0] - end[0]
        end_offset_y = point[1] - end[1]
        return end_offset_x * end_offset_x + end_offset_y * end_offset_y, 1

    cross = segment_x * offset_y - segment_y * offset_x
    return cross * cross, segment_squared


def simplify_integer_polyline(
    points: tuple[Point, ...],
    *,
    tolerance_squared: int,
) -> PolylineSimplification:
    """Return the vertices retained by exact Ramer-Douglas-Peucker tests."""
    if type(points) is not tuple:
        raise TypeError("points must be an exact tuple")
    if len(points) > _MAX_POLYLINE_POINTS:
        raise ValueError("point count exceeds 2048")
    for point_index, point in enumerate(points):
        if type(point) is not tuple:
            raise TypeError(f"points[{point_index}] must be an exact tuple")
        if len(point) != 2:
            raise ValueError(f"points[{point_index}] must contain two coordinates")
        for coordinate in point:
            if type(coordinate) is not int:
                raise TypeError("coordinates must be exact integers")
            if not _MIN_COORDINATE <= coordinate <= _MAX_COORDINATE:
                raise ValueError("coordinate is outside the signed 32-bit range")

    if type(tolerance_squared) is not int:
        raise TypeError("tolerance_squared must be an exact integer")
    if not 0 <= tolerance_squared <= _MAX_TOLERANCE_SQUARED:
        raise ValueError("tolerance_squared is outside the supported range")

    if len(points) < 2:
        indexes = tuple(range(len(points)))
        return PolylineSimplification(indexes, points)

    retained = [False] * len(points)
    retained[0] = True
    retained[-1] = True
    pending = [(0, len(points) - 1)]

    while pending:
        start_index, end_index = pending.pop()
        if end_index - start_index <= 1:
            continue

        farthest_index = start_index + 1
        farthest_numerator, farthest_denominator = _point_segment_distance_squared_ratio(
            points[farthest_index],
            points[start_index],
            points[end_index],
        )
        for point_index in range(start_index + 2, end_index):
            numerator, denominator = _point_segment_distance_squared_ratio(
                points[point_index],
                points[start_index],
                points[end_index],
            )
            if numerator * farthest_denominator > farthest_numerator * denominator:
                farthest_index = point_index
                farthest_numerator = numerator
                farthest_denominator = denominator

        if farthest_numerator > tolerance_squared * farthest_denominator:
            retained[farthest_index] = True
            pending.append((farthest_index, end_index))
            pending.append((start_index, farthest_index))

    retained_indexes = tuple(index for index, keep in enumerate(retained) if keep)
    return PolylineSimplification(
        retained_indexes=retained_indexes,
        retained_points=tuple(points[index] for index in retained_indexes),
    )
```

## Example

```python

def fraction_distance_squared(point: Point, start: Point, end: Point) -> Fraction:
    segment_x = end[0] - start[0]
    segment_y = end[1] - start[1]
    offset_x = point[0] - start[0]
    offset_y = point[1] - start[1]
    segment_squared = segment_x * segment_x + segment_y * segment_y
    if segment_squared == 0:
        return Fraction(offset_x * offset_x + offset_y * offset_y)

    projection = offset_x * segment_x + offset_y * segment_y
    if projection <= 0:
        return Fraction(offset_x * offset_x + offset_y * offset_y)
    if projection >= segment_squared:
        end_x = point[0] - end[0]
        end_y = point[1] - end[1]
        return Fraction(end_x * end_x + end_y * end_y)
    cross = segment_x * offset_y - segment_y * offset_x
    return Fraction(cross * cross, segment_squared)


def recursive_oracle(points: tuple[Point, ...], tolerance_squared: int) -> tuple[int, ...]:
    if len(points) < 2:
        return tuple(range(len(points)))

    def visit(start: int, end: int) -> tuple[int, ...]:
        if end - start <= 1:
            return start, end
        distances = tuple(
            fraction_distance_squared(points[index], points[start], points[end])
            for index in range(start + 1, end)
        )
        largest = max(distances)
        if largest <= tolerance_squared:
            return start, end
        split = start + 1 + distances.index(largest)
        left = visit(start, split)
        right = visit(split, end)
        return left + right[1:]

    return visit(0, len(points) - 1)


polyline = ((0, 0), (1, 1), (2, 0), (3, 0))
assert simplify_integer_polyline(polyline, tolerance_squared=0) == PolylineSimplification(
    retained_indexes=(0, 1, 2, 3),
    retained_points=polyline,
)
assert simplify_integer_polyline(polyline, tolerance_squared=1).retained_indexes == (0, 3)
assert simplify_integer_polyline(
    ((0, 0), (1, 1), (2, 0)), tolerance_squared=1
).retained_indexes == (
    0,
    2,
)

small_points = tuple(product(range(-1, 2), repeat=2))
checked = 0
for point_count in range(5):
    for candidate in product(small_points, repeat=point_count):
        for tolerance in range(4):
            actual = simplify_integer_polyline(candidate, tolerance_squared=tolerance)
            expected_indexes = recursive_oracle(candidate, tolerance)
            assert actual.retained_indexes == expected_indexes
            assert actual.retained_points == tuple(candidate[index] for index in expected_indexes)
            checked += 1

boundary = (-(2**31), 2**31 - 1)
assert simplify_integer_polyline(
    (boundary, boundary, boundary), tolerance_squared=0
).retained_indexes == (
    0,
    2,
)
assert checked == 29_524
```

## Trade-offs and Limitations

For `N` points, the iterative stack retains `O(N)` state. Repeated rescanning
can take `O(N²)` time on adversarial polylines. Python integers keep cross
products and squared distances exact beyond machine-word size, with arithmetic
cost that grows with those bounded values.

Tolerance equality removes the interior point, and equal farthest distances
select the earliest input index. These choices make the result reproducible;
they do not make Ramer-Douglas-Peucker a minimum-vertex or globally optimal
simplifier. Repeated points are valid, including a span whose endpoints
coincide.

The routine treats coordinates as a flat Cartesian plane. It does not preserve
polygon topology, prevent new intersections, account for geographic
projection, infer measurement error, or operate incrementally.

## Related Snippets

<!-- catalog:related:start -->
- [Classify the Exact Intersection of Two Closed Integer Line Segments](classify-the-exact-intersection-of-two-closed-integer-line-segments.md)
- [Compute a Canonical Convex Hull for Bounded Integer Points](compute-a-canonical-convex-hull-for-bounded-integer-points.md)
- [Downsample Fixed-Width Integer Tick Buckets by First, Minimum, Maximum, and Last](../data-processing/downsample-fixed-width-integer-tick-buckets-by-first-minimum-maximum-and-last.md)
<!-- catalog:related:end -->
