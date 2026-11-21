---
title: "Certify a Rectangle Tiling with Exact Area and Corner Parity"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-the-exact-union-area-of-bounded-axis-aligned-integer-rectangles.md
  - compute-the-exact-union-perimeter-of-bounded-axis-aligned-integer-rectangles.md
---

# Certify a Rectangle Tiling with Exact Area and Corner Parity

## Idea and Problem

Determine whether rectangular pieces cover one target rectangle exactly once without enumerating every grid cell.

Toggle each piece corner in a set. On the grid induced by the piece edges,
the parity of a cell's coverage equals the parity of all corner toggles with
x at most the cell's left edge and y at most its bottom edge. Matching the
target's four corner toggles therefore makes every target cell's coverage
odd. If the summed piece area also matches the target area, every cell must
be covered exactly once.

## When to Use

Use this to validate an axis-aligned layout such as a partitioned canvas or
rectangular storage region. Supply the target and each tile as four integer
coordinates `(left, bottom, right, top)` with positive width and height. The
occupied region is half-open on each axis. Shared edges and corners are allowed.

## Implementation

```python
from collections.abc import Iterable


type Rectangle = tuple[int, int, int, int]


def _checked_rectangle(rectangle: Rectangle) -> Rectangle:
    left, bottom, right, top = rectangle
    if any(type(value) is not int for value in (left, bottom, right, top)):
        raise TypeError("rectangle coordinates must be integers")
    if left >= right or bottom >= top:
        raise ValueError("rectangles must have positive width and height")
    return left, bottom, right, top


def _corners(rectangle: Rectangle) -> tuple[tuple[int, int], ...]:
    left, bottom, right, top = rectangle
    return (left, bottom), (left, top), (right, bottom), (right, top)


def certifies_rectangle_tiling(target: Rectangle, tiles: Iterable[Rectangle]) -> bool:
    outer = _checked_rectangle(target)
    left, bottom, right, top = outer
    target_area = (right - left) * (top - bottom)
    area = 0
    odd_corners: set[tuple[int, int]] = set()

    for raw_tile in tiles:
        tile = _checked_rectangle(raw_tile)
        x0, y0, x1, y1 = tile
        if x0 < left or y0 < bottom or x1 > right or y1 > top:
            return False
        area += (x1 - x0) * (y1 - y0)
        odd_corners.symmetric_difference_update(_corners(tile))

    return area == target_area and odd_corners == set(_corners(outer))
```

## Example

```python
target = (0, 0, 4, 3)
tiling = ((0, 0, 2, 3), (2, 0, 4, 1), (2, 1, 4, 3))
gap_and_overlap = ((0, 0, 3, 3), (2, 0, 3, 3))  # Same total area as target.
triple_coverage = (target, target, target)  # Same corner parity as target.

assert (
    certifies_rectangle_tiling(target, iter(tiling)),
    certifies_rectangle_tiling(target, gap_and_overlap),
    certifies_rectangle_tiling(target, triple_coverage),
    certifies_rectangle_tiling(target, ((0, 0, 5, 3),)),
) == (True, False, False, False)
```

## Trade-offs and Limitations

The check takes one pass, with four set toggles per tile and storage for
corners with odd counts. It rejects a tile outside the target immediately,
so the rest of that input may remain unread. An empty tile list fails for
any positive-area target.

The answer is a Boolean certificate. Use a coverage computation when you
need the locations of gaps or overlaps. Rotated pieces and approximate
coordinates require a different geometric model.

## Related Snippets

<!-- catalog:related:start -->
- [Compute the Exact Union Area of Bounded Axis-Aligned Integer Rectangles](compute-the-exact-union-area-of-bounded-axis-aligned-integer-rectangles.md)
- [Compute the Exact Union Perimeter of Bounded Axis-Aligned Integer Rectangles](compute-the-exact-union-perimeter-of-bounded-axis-aligned-integer-rectangles.md)
<!-- catalog:related:end -->
