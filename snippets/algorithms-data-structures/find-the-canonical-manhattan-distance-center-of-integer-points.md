---
title: "Find the Canonical Manhattan-Distance Center of Integer Points"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../machine-learning-statistics/select-the-lower-weighted-median-of-bounded-integer-observations.md
  - compute-an-exact-area-centroid-for-a-simple-integer-polygon.md
---

# Find the Canonical Manhattan-Distance Center of Integer Points

## Idea and Problem

Choose the lexicographically first integer location that minimizes the total Manhattan distance to a collection of points.

The cost separates into the sum of horizontal distances and the sum of
vertical distances, so minimize each coordinate independently. A median
minimizes the sum of absolute differences in one dimension. For even input
size, the entire interval between the two middle coordinates minimizes that
cost. Taking the lower median on each axis resolves the tie.

## When to Use

Use this for a representative location when travel cost is the sum of axis
displacements on an integer grid. All input records have equal weight, and
repeated points count repeatedly. The chosen location can be anywhere on the
grid, including locations absent from the original points.

## Implementation

```python
from collections.abc import Iterable
from dataclasses import dataclass
from statistics import median_low


@dataclass(frozen=True, slots=True)
class ManhattanCenter:
    x: int
    y: int
    total_distance: int


def manhattan_center(points: Iterable[tuple[int, int]]) -> ManhattanCenter:
    xs: list[int] = []
    ys: list[int] = []
    for x, y in points:
        if type(x) is not int or type(y) is not int:
            raise TypeError("point coordinates must be integers")
        xs.append(x)
        ys.append(y)
    if not xs:
        raise ValueError("at least one point is required")

    center_x = median_low(xs)
    center_y = median_low(ys)
    distance = sum(abs(x - center_x) for x in xs) + sum(abs(y - center_y) for y in ys)
    return ManhattanCenter(center_x, center_y, distance)
```

## Example

```python
locations = ((0, 2), (2, 0), (4, 4), (6, 6))
repeated = ((3, 9), (3, 9), (8, 2))

assert (
    manhattan_center(iter(locations)),
    manhattan_center(reversed(locations)),
    manhattan_center(repeated),
) == (ManhattanCenter(2, 2, 16), ManhattanCenter(2, 2, 16), ManhattanCenter(3, 9, 12))
```

## Trade-offs and Limitations

The function holds all coordinates and sorts each axis through `median_low()`.
This costs `O(n log n)` comparisons for `n` points. If locations carry unequal
integer weights, choose a lower weighted median on each axis instead.

The model uses Manhattan distance on a grid with unrestricted axis-aligned
travel. Obstacles, a prescribed list of possible locations and Euclidean
distance require different optimization rules.

## Related Snippets

<!-- catalog:related:start -->
- [Select the Lower Weighted Median of Bounded Integer Observations](../machine-learning-statistics/select-the-lower-weighted-median-of-bounded-integer-observations.md)
- [Compute an Exact Area Centroid for a Simple Integer Polygon](compute-an-exact-area-centroid-for-a-simple-integer-polygon.md)
<!-- catalog:related:end -->
