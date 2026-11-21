---
title: "Find a Farthest Manhattan Pair with Coordinate Extrema"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
tested_python:
  - "3.14"
dependencies: []
related:
  - find-the-canonical-manhattan-distance-center-of-integer-points.md
  - find-the-canonical-farthest-pair-of-bounded-integer-points-with-rotating-calipers.md
---

# Find a Farthest Manhattan Pair with Coordinate Extrema

## Idea and Problem

Find the two input positions farthest apart on an integer grid under Manhattan distance without comparing every pair.

For coordinate differences `dx` and `dy`, the distance `abs(dx) + abs(dy)`
equals `max(abs(dx + dy), abs(dx - dy))`. The largest distance is therefore
the larger range of the two forms `x + y` and `x - y`. Keep the earliest
input position at each extreme. For a positive range its two extremes are
disjoint, and taking the earliest position at each gives the earliest
sorted index pair for that range.

## When to Use

Use this to find the maximum axis-aligned travel distance between two
observations, for example on an unobstructed grid. The input is a finite
iterable of two-coordinate integer points, with at least two observations.
The function returns `(first_index, second_index, distance)` with distinct,
increasing, zero-based indices. Ties choose the lexicographically first
index pair, so identical coordinates throughout yield `(0, 1, 0)`.

## Implementation

```python
from collections.abc import Iterable


def farthest_manhattan_pair(points: Iterable[tuple[int, int]]) -> tuple[int, int, int]:
    # Each slot stores the highest value of one signed coordinate form.
    extremes: list[tuple[int, int]] = []
    count = 0
    for index, (x, y) in enumerate(points):
        if type(x) is not int or type(y) is not int:
            raise TypeError("point coordinates must be integers")
        plus, minus = x + y, x - y
        for slot, value in enumerate((plus, -plus, minus, -minus)):
            if index == 0:
                extremes.append((value, index))
            elif value > extremes[slot][0]:
                extremes[slot] = (value, index)
        count += 1

    if count < 2:
        raise ValueError("at least two points are required")

    best = (0, 0, 1)  # Negative distance, then increasing indices.
    for slot in (0, 2):
        upper, i = extremes[slot]
        negative_lower, j = extremes[slot + 1]
        if i != j:
            first, second = sorted((i, j))
            best = min(best, (-(upper + negative_lower), first, second))
    negative_distance, first, second = best
    return first, second, -negative_distance
```

## Example

```python
corners = ((0, 0), (2, 2), (0, 2), (2, 0), (0, 0))
crossing = ((0, 0), (2, 0), (0, 2), (0, 2))

assert farthest_manhattan_pair(iter(corners)) == (0, 1, 4)
assert farthest_manhattan_pair(crossing) == (1, 2, 4)
assert farthest_manhattan_pair([(3, -2)] * 3) == (0, 1, 0)
```

## Trade-offs and Limitations

The function consumes the input once, uses `O(n)` integer operations and
stores a fixed number of integers. It returns indices, not points; keep the
records separately if you need to retrieve them after consuming an iterator.

This derivation is for two-dimensional Manhattan distance, not Euclidean
distance or travel with obstacles. It returns one pair even when many pairs
share the maximum distance.

## Related Snippets

<!-- catalog:related:start -->
- [Find the Canonical Manhattan-Distance Center of Integer Points](find-the-canonical-manhattan-distance-center-of-integer-points.md)
- [Find the Canonical Farthest Pair of Bounded Integer Points with Rotating Calipers](find-the-canonical-farthest-pair-of-bounded-integer-points-with-rotating-calipers.md)
<!-- catalog:related:end -->
