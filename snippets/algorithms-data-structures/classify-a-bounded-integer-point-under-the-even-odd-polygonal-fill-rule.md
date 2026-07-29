---
title: "Classify a Bounded Integer Point Under the Even-Odd Polygonal Fill Rule"
snippet_type: algorithm
use_cases:
  - data-transformation
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - classify-the-exact-intersection-of-two-closed-integer-line-segments.md
  - compute-a-canonical-convex-hull-for-bounded-integer-points.md
  - find-a-closest-pair-of-bounded-integer-points-with-earliest-index-pair-ties.md
---

# Classify a Bounded Integer Point Under the Even-Odd Polygonal Fill Rule

## Idea and Problem

Classify an integer point against one closed polygonal path without floating-point tolerances or an unstated simple-polygon assumption.

First test whether the point lies on any closed edge. Otherwise cast a ray to
the right and toggle parity whenever an edge crosses the point's horizontal
line on a half-open vertical interval and continues beyond the point. Exact
cross products compare the intersection with the query without division.

Counting every declared edge gives a defined result for repeated,
backtracking, degenerate, and self-crossing paths: odd crossing parity is
inside and even parity is outside. Boundary membership always takes
precedence over parity.

## When to Use

Use this predicate when coordinates are exact integers and the desired fill
semantics are the even-odd rule used by many drawing and clipping systems. The
vertices form one implicitly closed path: consecutive vertices are connected,
and the final vertex connects back to the first.

This is suitable for validation, deterministic geometry tests, and bounded
spatial filtering where edge membership must be exact. Use a geometry library
when coordinates need a reference system or tolerance, when several rings
must represent holes, or when polygon repair, indexing, buffering, or robust
floating-point predicates are also required.

## Implementation

```python
from typing import TypeAlias

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_POLYGONAL_VERTICES = 50_000

IntegerPoint: TypeAlias = tuple[int, int]


def _validate_integer_point(value: object, *, name: str) -> None:
    if type(value) is not tuple:
        raise TypeError(f"{name} must be an exact tuple")
    if len(value) != 2:
        raise ValueError(f"{name} must contain two coordinates")
    for coordinate_index, coordinate in enumerate(value):
        if type(coordinate) is not int:
            raise TypeError(f"{name}[{coordinate_index}] must be an exact integer")
        if not _MIN_INT64 <= coordinate <= _MAX_INT64:
            raise ValueError(f"{name}[{coordinate_index}] is outside the signed 64-bit range")


def _point_is_on_closed_segment(
    point: IntegerPoint,
    start: IntegerPoint,
    stop: IntegerPoint,
) -> bool:
    point_x, point_y = point
    start_x, start_y = start
    stop_x, stop_y = stop
    cross = (stop_x - start_x) * (point_y - start_y) - (stop_y - start_y) * (point_x - start_x)
    return (
        cross == 0
        and min(start_x, stop_x) <= point_x <= max(start_x, stop_x)
        and min(start_y, stop_y) <= point_y <= max(start_y, stop_y)
    )


def classify_point_by_even_odd_fill(
    point: IntegerPoint,
    vertices: tuple[IntegerPoint, ...],
) -> str:
    """Return 'boundary', 'inside', or 'outside' for one closed path."""
    _validate_integer_point(point, name="point")
    if type(vertices) is not tuple:
        raise TypeError("vertices must be an exact tuple")
    if not 3 <= len(vertices) <= _MAX_POLYGONAL_VERTICES:
        raise ValueError("vertex count is outside the supported range")
    for vertex_index, vertex in enumerate(vertices):
        _validate_integer_point(vertex, name=f"vertices[{vertex_index}]")

    point_x, point_y = point
    inside = False
    start = vertices[-1]
    for stop in vertices:
        if _point_is_on_closed_segment(point, start, stop):
            return "boundary"

        start_x, start_y = start
        stop_x, stop_y = stop
        if (start_y > point_y) != (stop_y > point_y):
            cross = (stop_x - start_x) * (point_y - start_y) - (stop_y - start_y) * (
                point_x - start_x
            )
            if (cross > 0) == (stop_y > start_y):
                inside = not inside
        start = stop

    return "inside" if inside else "outside"
```

## Example

```python
square = ((0, 0), (6, 0), (6, 6), (0, 6))
reversed_square = tuple(reversed(square))
backtracking_path = ((0, 0), (6, 0), (0, 0), (0, 6))
extreme_square = (
    (_MIN_INT64, _MIN_INT64),
    (_MAX_INT64, _MIN_INT64),
    (_MAX_INT64, _MAX_INT64),
    (_MIN_INT64, _MAX_INT64),
)

assert (
    classify_point_by_even_odd_fill((2, 3), square),
    classify_point_by_even_odd_fill((2, 3), reversed_square),
    classify_point_by_even_odd_fill((6, 3), square),
    classify_point_by_even_odd_fill((7, 3), square),
    classify_point_by_even_odd_fill((3, 0), backtracking_path),
    classify_point_by_even_odd_fill((0, 0), ((0, 0), (0, 0), (0, 0))),
    classify_point_by_even_odd_fill((0, 0), extreme_square),
) == (
    "inside",
    "inside",
    "boundary",
    "outside",
    "boundary",
    "boundary",
    "inside",
)
```

## Trade-offs and Limitations

For `v` vertices, validation, boundary testing, and parity counting take
`O(v)` exact arithmetic operations and `O(1)` auxiliary memory. Coordinate
differences and cross products can exceed signed 64-bit range, but Python
integers preserve those intermediate results exactly.

The function implements an even-odd fill rule for one declared path. It does
not prove that the path is simple, discard duplicate edges, or reinterpret
backtracking segments; every edge contributes with its declared multiplicity.
Repeated and zero-length edges are allowed, and a point on any such closed
edge is boundary. These semantics may differ from a nonzero winding rule.

The result is only a point predicate. It does not calculate area, normalize
orientation, model multiple rings or holes, repair intersections, accelerate
many queries with a spatial index, or accept floating-point coordinates.

## Related Snippets

<!-- catalog:related:start -->
- [Classify the Exact Intersection of Two Closed Integer Line Segments](classify-the-exact-intersection-of-two-closed-integer-line-segments.md)
- [Compute a Canonical Convex Hull for Bounded Integer Points](compute-a-canonical-convex-hull-for-bounded-integer-points.md)
- [Find a Closest Pair of Bounded Integer Points with Earliest Index-Pair Ties](find-a-closest-pair-of-bounded-integer-points-with-earliest-index-pair-ties.md)
<!-- catalog:related:end -->
