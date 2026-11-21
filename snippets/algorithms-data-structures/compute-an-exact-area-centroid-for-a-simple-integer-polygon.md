---
title: "Compute an Exact Area Centroid for a Simple Integer Polygon"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-the-signed-shoelace-double-area-of-a-bounded-integer-vertex-walk.md
  - classify-a-bounded-integer-point-under-the-even-odd-polygonal-fill-rule.md
---

# Compute an Exact Area Centroid for a Simple Integer Polygon

## Idea and Problem

Find the center of a uniformly filled simple polygon with exact rational coordinates.

The mean of the vertices gives each corner equal weight even when their
enclosed regions have different areas. Accumulate signed area and coordinate
moments over the polygon edges instead. Reversing the winding changes the
sign of each sum, leaving their ratios unchanged.

## When to Use

Use this with one already validated simple polygon ring in an integer
Cartesian coordinate system. Supply vertices in boundary order, with
either winding. The code connects the last vertex to the first. A repeated
closing vertex is harmless. The result is useful for shape summaries and
comparisons where rounding would change equality.

## Implementation

```python
from collections.abc import Iterable
from fractions import Fraction


def area_centroid(vertices: Iterable[tuple[int, int]]) -> tuple[Fraction, Fraction]:
    """Return the area centroid of one simple polygon ring."""
    ring = tuple(vertices)
    if len(ring) < 3:
        raise ValueError("a polygon needs at least three vertices")
    if any(type(x) is not int or type(y) is not int for x, y in ring):
        raise TypeError("vertex coordinates must be integers")

    twice_area = x_moment = y_moment = 0
    previous_x, previous_y = ring[-1]
    for x, y in ring:
        cross = previous_x * y - x * previous_y
        twice_area += cross
        x_moment += (previous_x + x) * cross
        y_moment += (previous_y + y) * cross
        previous_x, previous_y = x, y

    if twice_area == 0:
        raise ValueError("the polygon has zero signed area")
    denominator = 3 * twice_area
    return Fraction(x_moment, denominator), Fraction(y_moment, denominator)
```

## Example

```python
# The L shape joins a 4 by 1 rectangle and a 1 by 2 rectangle above it.
ring = ((0, 0), (4, 0), (4, 1), (1, 1), (1, 3), (0, 3))
expected = (Fraction(3, 2), Fraction(1))

try:
    area_centroid(((0, 0), (1, 1), (2, 2)))
except ValueError:
    zero_rejected = True
else:
    zero_rejected = False

assert (
    area_centroid(ring),
    area_centroid(reversed(ring)),
    area_centroid((*ring, ring[0])),
    zero_rejected,
) == (expected, expected, expected, True)
```

## Trade-offs and Limitations

The calculation examines every edge once after validating the coordinate
types. It retains the input ring and uses exact integer arithmetic until
the final division. Conversion to float for display can round the result.

The shape's simplicity is a precondition. A nonzero signed area alone cannot
establish that edges avoid self-intersection. This recipe covers a single
ring with uniform density. For holes, weighted density or geographic
coordinates, choose a geometry model that specifies those semantics.

The centroid can lie outside a concave polygon. Use an interior-point
algorithm if an annotation must stay inside the filled area.

## Related Snippets

<!-- catalog:related:start -->
- [Compute the Signed Shoelace Double Area of a Bounded Integer Vertex Walk](compute-the-signed-shoelace-double-area-of-a-bounded-integer-vertex-walk.md)
- [Classify a Bounded Integer Point Under the Even-Odd Polygonal Fill Rule](classify-a-bounded-integer-point-under-the-even-odd-polygonal-fill-rule.md)
<!-- catalog:related:end -->
