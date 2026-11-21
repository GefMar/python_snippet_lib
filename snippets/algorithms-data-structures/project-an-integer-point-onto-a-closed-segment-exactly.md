---
title: "Project an Integer Point onto a Closed Segment Exactly"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - clip-an-integer-segment-to-a-closed-rectangle-exactly.md
  - simplify-a-bounded-integer-polyline-with-exact-ramer-douglas-peucker-distance-comparisons.md
---

# Project an Integer Point onto a Closed Segment Exactly

## Idea and Problem

Find the nearest point on a closed segment and its squared distance from an integer query point.

The dot product of the query offset and the segment direction gives the
projection's numerator. Divide by the squared segment length to get the
parameter on the supporting line. Clamp that parameter to `[0, 1]` to
select the nearest endpoint when the line projection lies outside the
segment. Fractions preserve exact coordinates and squared distances.

## When to Use

Use this for comparisons or geometry tests in an integer Cartesian plane.
The parameter runs from the first endpoint at zero to the second at one.
Coincident endpoints represent a single point and return parameter zero.
An interior projection can have fractional coordinates even when all
inputs are integers.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction


type Point = tuple[int, int]


@dataclass(frozen=True, slots=True)
class Projection:
    parameter: Fraction
    point: tuple[Fraction, Fraction]
    squared_distance: Fraction


def _checked_point(point: Point) -> Point:
    x, y = point
    if type(x) is not int or type(y) is not int:
        raise TypeError("point coordinates must be integers")
    return x, y


def project_onto_segment(query: Point, start: Point, end: Point) -> Projection:
    query_x, query_y = _checked_point(query)
    start_x, start_y = _checked_point(start)
    end_x, end_y = _checked_point(end)
    dx, dy = end_x - start_x, end_y - start_y
    length_squared = dx * dx + dy * dy

    parameter = Fraction(0)
    if length_squared:
        dot = (query_x - start_x) * dx + (query_y - start_y) * dy
        parameter = Fraction(min(max(dot, 0), length_squared), length_squared)

    x = start_x + parameter * dx
    y = start_y + parameter * dy
    distance_squared = (query_x - x) ** 2 + (query_y - y) ** 2
    return Projection(parameter, (x, y), distance_squared)
```

## Example

```python
assert (
    project_onto_segment((1, 2), (0, 0), (3, 3)),
    project_onto_segment((5, 1), (0, 0), (3, 0)),
    project_onto_segment((-1, 2), (0, 0), (3, 0)),
    project_onto_segment((1, 5), (-1, 2), (-1, 2)),
) == (
    Projection(Fraction(1, 2), (Fraction(3, 2), Fraction(3, 2)), Fraction(1, 2)),
    Projection(Fraction(1), (Fraction(3), Fraction(0)), Fraction(5)),
    Projection(Fraction(0), (Fraction(0), Fraction(0)), Fraction(5)),
    Projection(Fraction(0), (Fraction(-1), Fraction(2)), Fraction(13)),
)
```

## Trade-offs and Limitations

The implementation uses a fixed number of arithmetic operations for a
single segment. Fraction operations have a cost that depends on the size
of the integers. Comparing squared distances avoids computing square roots.

This is Euclidean geometry on one segment. For many queries against a large
polyline, use a spatial index to narrow the candidate segments. Geographic
coordinates require a distance model appropriate to their projection.

## Related Snippets

<!-- catalog:related:start -->
- [Clip an Integer Segment to a Closed Rectangle Exactly](clip-an-integer-segment-to-a-closed-rectangle-exactly.md)
- [Simplify a Bounded Integer Polyline with Exact Ramer-Douglas-Peucker Distance Comparisons](simplify-a-bounded-integer-polyline-with-exact-ramer-douglas-peucker-distance-comparisons.md)
<!-- catalog:related:end -->
