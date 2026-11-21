---
title: "Clip an Integer Segment to a Closed Rectangle Exactly"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - map-points-between-rectangular-coordinate-spaces.md
  - simplify-a-bounded-integer-polyline-with-exact-ramer-douglas-peucker-distance-comparisons.md
---

# Clip an Integer Segment to a Closed Rectangle Exactly

## Idea and Problem

Find the exact portion of an integer line segment that is inside a closed axis-aligned rectangle.

Each coordinate constrains the same parameter along the segment. Intersect
its allowed intervals with the segment parameter range from zero to one.
Evaluating just the two remaining endpoints preserves direction and exact
boundary hits.

## When to Use

Use this for planar integer geometry, such as clipping a line to a viewport
before later rendering or rasterization. Supply the lower and upper corners
of the rectangle. Boundaries are included, and a rectangle may collapse to
a line or point.

## Implementation

```python
from fractions import Fraction


type Point = tuple[int, int]
type ExactPoint = tuple[Fraction, Fraction]


def clip_segment(
    start: Point, end: Point, lower: Point, upper: Point
) -> tuple[ExactPoint, ExactPoint] | None:
    if any(low > high for low, high in zip(lower, upper, strict=True)):
        raise ValueError("rectangle corners are reversed")

    direction = (end[0] - start[0], end[1] - start[1])
    enter, leave = Fraction(0), Fraction(1)

    for origin, delta, low, high in zip(start, direction, lower, upper, strict=True):
        if delta == 0:
            if not low <= origin <= high:
                return None
            continue

        first, second = sorted((Fraction(low - origin, delta), Fraction(high - origin, delta)))
        enter = max(enter, first)
        leave = min(leave, second)
        if enter > leave:
            return None

    def point_at(parameter: Fraction) -> ExactPoint:
        return (
            start[0] + direction[0] * parameter,
            start[1] + direction[1] * parameter,
        )

    return point_at(enter), point_at(leave)
```

## Example

```python
lower, upper = (0, 0), (3, 4)
start, end = (-2, 0), (5, 3)
first = (0, Fraction(6, 7))
last = (3, Fraction(15, 7))

assert (
    clip_segment(start, end, lower, upper),
    clip_segment(end, start, lower, upper),
    clip_segment((-1, 1), (1, -1), lower, upper),
    clip_segment((-1, 8), (5, 8), lower, upper),
) == ((first, last), (last, first), ((0, 0), (0, 0)), None)
```

## Trade-offs and Limitations

The method uses a fixed number of rational comparisons and arithmetic steps
per segment. Large input coordinates still increase integer arithmetic costs.
A tangency or an in-bounds point segment yields two equal endpoints.

This works in planar coordinates, with closed rectangle bounds. Tiles that
share an edge can both include a segment on that edge. Define a separate
ownership rule if the shared edge must belong to only one tile.

## Related Snippets

<!-- catalog:related:start -->
- [Map Points Between Rectangular Coordinate Spaces](map-points-between-rectangular-coordinate-spaces.md)
- [Simplify a Bounded Integer Polyline with Exact Ramer-Douglas-Peucker Distance Comparisons](simplify-a-bounded-integer-polyline-with-exact-ramer-douglas-peucker-distance-comparisons.md)
<!-- catalog:related:end -->
