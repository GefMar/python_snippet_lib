---
title: "Count Sparse Static Points in Half-Open Rectangles with an Offline Sweep"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - build-a-bounded-integer-summed-area-table-for-half-open-rectangle-sums.md
  - maintain-bounded-2d-point-additions-and-half-open-rectangle-sums-with-a-fenwick-tree.md
---

# Count Sparse Static Points in Half-Open Rectangles with an Offline Sweep

## Idea and Problem

Count points in a batch of half-open rectangles without allocating a grid across the coordinate space.

Subtract the point count to the left of a rectangle from the count to the
left of its right edge, using the same vertical band on both sides. Sweep
all such boundaries in x order. At each boundary, a Fenwick tree holds counts
by y for points with strictly smaller x. Compress only observed y values so
spatial gaps do not occupy memory.

## When to Use

Use this for many rectangle queries over one fixed set of integer coordinate
records. Rectangles use `(left, bottom, right, top)` and cover
`[left, right) × [bottom, top)`. Repeated points represent separate records.
The result contains one count for each rectangle in input order.

## Implementation

```python
from bisect import bisect_left
from collections.abc import Iterable


def _prefix_count(tree: list[int], stop: int) -> int:
    total = 0
    while stop:
        total += tree[stop]
        stop -= stop & -stop
    return total


def count_points_in_rectangles(
    points: Iterable[tuple[int, int]],
    rectangles: Iterable[tuple[int, int, int, int]],
) -> tuple[int, ...]:
    ordered_points: list[tuple[int, int]] = []
    for x, y in points:
        if type(x) is not int or type(y) is not int:
            raise TypeError("point coordinates must be integers")
        ordered_points.append((x, y))
    ordered_points.sort()

    y_values = sorted({y for _, y in ordered_points})
    tree = [0] * (len(y_values) + 1)
    counts: list[int] = []
    events: list[tuple[int, int, int, int, int]] = []
    for index, (left, bottom, right, top) in enumerate(rectangles):
        if any(type(value) is not int for value in (left, bottom, right, top)):
            raise TypeError("rectangle boundaries must be integers")
        if right < left or top < bottom:
            raise ValueError("rectangle boundaries must be in increasing order")
        counts.append(0)
        if left == right or bottom == top:
            continue
        low = bisect_left(y_values, bottom)
        high = bisect_left(y_values, top)
        events.extend(((left, index, -1, low, high), (right, index, 1, low, high)))

    consumed = 0
    for x, index, sign, low, high in sorted(events):
        while consumed < len(ordered_points) and ordered_points[consumed][0] < x:
            _, y = ordered_points[consumed]
            rank = bisect_left(y_values, y) + 1
            while rank < len(tree):
                tree[rank] += 1
                rank += rank & -rank
            consumed += 1
        counts[index] += sign * (_prefix_count(tree, high) - _prefix_count(tree, low))
    return tuple(counts)
```

## Example

```python
points = ((0, 1), (-3, -2), (0, 1), (8, 2), (2, 0), (0, 2))
regions = ((0, 0, 2, 2), (-4, -3, 0, 0), (0, 1, 9, 3), (0, 0, 0, 3))

assert count_points_in_rectangles(points, regions) == (2, 1, 4, 0)
```

## Trade-offs and Limitations

The function keeps the points, rectangle boundaries and one counter tree
entry per distinct y. Sorting and logarithmic tree operations trade some
upfront work for avoiding a full point scan per query. A direct scan is
simpler when there are only a few queries.

This sweep assumes the data and query batch are available together. For a
dense fixed grid, a summed-area table can answer a query in constant time
after preprocessing. For interleaved point changes, use an index with an
update operation.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Bounded Integer Summed-Area Table for Half-Open Rectangle Sums](build-a-bounded-integer-summed-area-table-for-half-open-rectangle-sums.md)
- [Maintain Bounded 2D Point Additions and Half-Open Rectangle Sums with a Fenwick Tree](maintain-bounded-2d-point-additions-and-half-open-rectangle-sums-with-a-fenwick-tree.md)
<!-- catalog:related:end -->
