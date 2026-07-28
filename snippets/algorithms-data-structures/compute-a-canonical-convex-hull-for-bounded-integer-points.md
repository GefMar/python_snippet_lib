---
title: "Compute a Canonical Convex Hull for Bounded Integer Points"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - map-points-between-rectangular-coordinate-spaces.md
  - match-strict-mutual-nearest-neighbors-with-a-comparison-budget.md
  - ../data-processing/extract-a-finite-2d-bounding-box-from-bounded-wkb.md
---

# Compute a Canonical Convex Hull for Bounded Integer Points

## Idea and Problem

Reduce a bounded collection of integer points to its outermost vertices in one canonical order.

Andrew's monotone chain sorts unique points lexicographically and builds the
lower and upper hulls with orientation tests. Removing the previous vertex
whenever a new point would make a clockwise or collinear turn discards both
interior points and non-extreme points along a straight hull edge. Exact integer
cross products avoid tolerance decisions.

## When to Use

Use this algorithm for a complete, bounded two-dimensional integer point set
when downstream work needs only the extreme boundary vertices. The canonical
counterclockwise order is useful for deterministic comparison, rendering, or
further exact geometric calculations.

Use a geometry library when coordinates are floating point, a coordinate
reference system matters, or the work also needs polygon repair, predicates,
area, buffering, or triangulation. Choose a different hull structure for
incremental point updates or when every collinear boundary point must remain.

## Implementation

```python
from dataclasses import dataclass

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_POINT_COUNT = 4_096


@dataclass(frozen=True, slots=True)
class Point:
    x: int
    y: int


def _validated_points(points: object) -> tuple[Point, ...]:
    if type(points) is not tuple:
        raise TypeError("points must be an exact tuple")
    if not 1 <= len(points) <= _MAX_POINT_COUNT:
        raise ValueError("point count is outside the supported range")

    for index, point in enumerate(points):
        if type(point) is not Point:
            raise TypeError(f"points[{index}] must be an exact Point")
        if type(point.x) is not int or type(point.y) is not int:
            raise TypeError(f"points[{index}] coordinates must be exact integers")
        if not _MIN_INT64 <= point.x <= _MAX_INT64:
            raise ValueError(f"points[{index}].x is outside the signed 64-bit range")
        if not _MIN_INT64 <= point.y <= _MAX_INT64:
            raise ValueError(f"points[{index}].y is outside the signed 64-bit range")
    return points


def _cross(origin: Point, first: Point, second: Point) -> int:
    return (first.x - origin.x) * (second.y - origin.y) - (first.y - origin.y) * (
        second.x - origin.x
    )


def canonical_convex_hull(points: tuple[Point, ...]) -> tuple[Point, ...]:
    """Return extreme vertices counterclockwise from the smallest point."""
    validated = _validated_points(points)
    ordered = sorted(set(validated), key=lambda point: (point.x, point.y))
    if len(ordered) <= 2:
        return tuple(ordered)

    lower: list[Point] = []
    for point in ordered:
        while len(lower) >= 2 and _cross(lower[-2], lower[-1], point) <= 0:
            lower.pop()
        lower.append(point)

    upper: list[Point] = []
    for point in reversed(ordered):
        while len(upper) >= 2 and _cross(upper[-2], upper[-1], point) <= 0:
            upper.pop()
        upper.append(point)

    return tuple(lower[:-1] + upper[:-1])
```

## Example

```python
points = (
    Point(0, 0),
    Point(1, 0),
    Point(2, 0),
    Point(2, 2),
    Point(1, 1),
    Point(0, 2),
    Point(0, 0),
)
expected = (Point(0, 0), Point(2, 0), Point(2, 2), Point(0, 2))

collinear = canonical_convex_hull((Point(2, 2), Point(0, 0), Point(1, 1), Point(0, 0)))
extreme = canonical_convex_hull(
    (
        Point(_MIN_INT64, _MIN_INT64),
        Point(_MAX_INT64, _MIN_INT64),
        Point(_MAX_INT64, _MAX_INT64),
        Point(_MIN_INT64, _MAX_INT64),
        Point(0, 0),
    )
)

assert (
    canonical_convex_hull(points),
    canonical_convex_hull(tuple(reversed(points))),
    collinear,
    canonical_convex_hull((Point(4, -1), Point(4, -1))),
    extreme,
) == (
    expected,
    expected,
    (Point(0, 0), Point(2, 2)),
    (Point(4, -1),),
    (
        Point(_MIN_INT64, _MIN_INT64),
        Point(_MAX_INT64, _MIN_INT64),
        Point(_MAX_INT64, _MAX_INT64),
        Point(_MIN_INT64, _MAX_INT64),
    ),
)
```

## Trade-offs and Limitations

Validation, deduplication, and sorting take `O(n log n)` time, while the two
hull passes take `O(n)` time. The unique-point set, sorted list, partial hulls,
and returned tuple use `O(n)` memory. Coordinate subtraction and cross products
can exceed signed 64-bit range, but Python integers preserve the exact result.

Duplicate coordinates are coalesced, and collinear points inside a hull edge
are intentionally removed. A one-point hull contains that point; a two-point
or wholly collinear hull contains only its sorted endpoints. The result does
not repeat its first point. The function excludes floating-point tolerance,
coordinate reference systems, polygon validation, area, triangulation,
three-dimensional points, incremental updates, and retention of every
collinear boundary point.

## Related Snippets

<!-- catalog:related:start -->
- [Map Points Between Rectangular Coordinate Spaces](map-points-between-rectangular-coordinate-spaces.md)
- [Match Strict Mutual Nearest Neighbors with a Comparison Budget](match-strict-mutual-nearest-neighbors-with-a-comparison-budget.md)
- [Extract a Finite 2D Bounding Box from Bounded WKB](../data-processing/extract-a-finite-2d-bounding-box-from-bounded-wkb.md)
<!-- catalog:related:end -->
