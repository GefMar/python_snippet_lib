---
title: "Compute the Exact Union Perimeter of Bounded Axis-Aligned Integer Rectangles"
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
  - compute-the-exact-union-area-of-bounded-axis-aligned-integer-rectangles.md
  - coalesce-bounded-half-open-integer-intervals-under-an-explicit-adjacency-policy.md
  - find-peak-coverage-spans-across-bounded-half-open-integer-intervals.md
---

# Compute the Exact Union Perimeter of Bounded Axis-Aligned Integer Rectangles

## Idea and Problem

Measure the total boundary length of the union of bounded half-open integer rectangles without counting internal edges.

Compress every distinct x- and y-coordinate, then mark covered compressed
cells with a two-dimensional difference grid. A covered cell contributes an
edge length only when the neighboring cell across that edge is uncovered or
does not exist. Coordinate differences, rather than cell counts, preserve the
true geometric length.

This counts every boundary component, including the edges around holes.
Covered-to-covered interfaces contribute nothing, coincident exterior
boundaries count once, and duplicate rectangles do not cancel a boundary.

## When to Use

Use this algorithm for a small bounded set of exact rectangular regions when
overlap and touching must be resolved before measuring their complete union
boundary. It is useful for reference geometry, layout checks, grid-derived
regions, and test oracles where integer coordinates make exact arithmetic
preferable to floating-point clipping.

Use a sweep-line geometry library for a large sparse collection. Use polygon
Boolean operations when the boundary shape or connected components are needed,
or when regions are rotated, curved, or represented by non-integer
coordinates.

## Implementation

```python
from dataclasses import dataclass

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_RECTANGLE_COUNT = 128
_MAX_COMPRESSED_CELL_COUNT = 255 * 255


@dataclass(frozen=True, slots=True)
class HalfOpenRectangle:
    left: int
    bottom: int
    right: int
    top: int


def exact_rectangle_union_perimeter(
    rectangles: tuple[HalfOpenRectangle, ...],
) -> int:
    """Return total exterior and hole-boundary length of the rectangle union."""
    if type(rectangles) is not tuple:
        raise TypeError("rectangles must be an exact tuple")
    if len(rectangles) > _MAX_RECTANGLE_COUNT:
        raise ValueError("rectangle count exceeds the supported limit")

    x_coordinates: list[int] = []
    y_coordinates: list[int] = []
    for rectangle_index, rectangle in enumerate(rectangles):
        if type(rectangle) is not HalfOpenRectangle:
            raise TypeError(f"rectangles[{rectangle_index}] must be an exact HalfOpenRectangle")
        for field_name, coordinate in (
            ("left", rectangle.left),
            ("bottom", rectangle.bottom),
            ("right", rectangle.right),
            ("top", rectangle.top),
        ):
            if type(coordinate) is not int:
                raise TypeError(
                    f"rectangles[{rectangle_index}].{field_name} must be an exact integer"
                )
            if not _MIN_INT64 <= coordinate <= _MAX_INT64:
                raise ValueError(
                    f"rectangles[{rectangle_index}].{field_name} is outside the signed 64-bit range"
                )
        if rectangle.left >= rectangle.right:
            raise ValueError("every rectangle must have positive width")
        if rectangle.bottom >= rectangle.top:
            raise ValueError("every rectangle must have positive height")
        x_coordinates.extend((rectangle.left, rectangle.right))
        y_coordinates.extend((rectangle.bottom, rectangle.top))

    if not rectangles:
        return 0

    ordered_x = sorted(set(x_coordinates))
    ordered_y = sorted(set(y_coordinates))
    cell_count = (len(ordered_x) - 1) * (len(ordered_y) - 1)
    if cell_count > _MAX_COMPRESSED_CELL_COUNT:
        raise ValueError("compressed cell count exceeds the supported limit")

    x_index = {coordinate: index for index, coordinate in enumerate(ordered_x)}
    y_index = {coordinate: index for index, coordinate in enumerate(ordered_y)}
    difference = [[0] * len(ordered_y) for _ in ordered_x]

    for rectangle in rectangles:
        left = x_index[rectangle.left]
        right = x_index[rectangle.right]
        bottom = y_index[rectangle.bottom]
        top = y_index[rectangle.top]
        difference[left][bottom] += 1
        difference[right][bottom] -= 1
        difference[left][top] -= 1
        difference[right][top] += 1

    for x_position in range(len(ordered_x)):
        row_prefix = 0
        for y_position in range(len(ordered_y)):
            row_prefix += difference[x_position][y_position]
            if x_position > 0:
                difference[x_position][y_position] = (
                    row_prefix + difference[x_position - 1][y_position]
                )
            else:
                difference[x_position][y_position] = row_prefix

    perimeter = 0
    x_span_count = len(ordered_x) - 1
    y_span_count = len(ordered_y) - 1
    for x_position in range(x_span_count):
        width = ordered_x[x_position + 1] - ordered_x[x_position]
        for y_position in range(y_span_count):
            if difference[x_position][y_position] <= 0:
                continue
            height = ordered_y[y_position + 1] - ordered_y[y_position]
            if x_position == 0 or difference[x_position - 1][y_position] <= 0:
                perimeter += height
            if x_position + 1 == x_span_count or difference[x_position + 1][y_position] <= 0:
                perimeter += height
            if y_position == 0 or difference[x_position][y_position - 1] <= 0:
                perimeter += width
            if y_position + 1 == y_span_count or difference[x_position][y_position + 1] <= 0:
                perimeter += width

    return perimeter
```

## Example

```python
def unit_cell_perimeter(rectangles: tuple[HalfOpenRectangle, ...]) -> int:
    covered = {
        (x, y)
        for rectangle in rectangles
        for x in range(rectangle.left, rectangle.right)
        for y in range(rectangle.bottom, rectangle.top)
    }
    directions = ((-1, 0), (1, 0), (0, -1), (0, 1))
    return sum((x + dx, y + dy) not in covered for x, y in covered for dx, dy in directions)


overlapping = (
    HalfOpenRectangle(0, 0, 2, 2),
    HalfOpenRectangle(1, 0, 3, 2),
)
edge_touching = (
    HalfOpenRectangle(0, 0, 1, 2),
    HalfOpenRectangle(1, 0, 3, 2),
)
corner_touching = (
    HalfOpenRectangle(0, 0, 1, 1),
    HalfOpenRectangle(1, 1, 2, 2),
)
frame_with_hole = (
    HalfOpenRectangle(0, 0, 3, 1),
    HalfOpenRectangle(0, 2, 3, 3),
    HalfOpenRectangle(0, 1, 1, 2),
    HalfOpenRectangle(2, 1, 3, 2),
)
duplicate = (
    HalfOpenRectangle(0, 0, 1, 1),
    HalfOpenRectangle(0, 0, 1, 1),
)

fixtures = (
    (overlapping, 10),
    (edge_touching, 10),
    (corner_touching, 8),
    (frame_with_hole, 16),
    (duplicate, 4),
)
for fixture, expected in fixtures:
    assert exact_rectangle_union_perimeter(fixture) == expected
    assert exact_rectangle_union_perimeter(fixture) == unit_cell_perimeter(fixture)
    assert exact_rectangle_union_perimeter(tuple(reversed(fixture))) == expected

coordinate_width = _MAX_INT64 - _MIN_INT64
assert exact_rectangle_union_perimeter(()) == 0
assert (
    exact_rectangle_union_perimeter(
        (HalfOpenRectangle(_MIN_INT64, _MIN_INT64, _MAX_INT64, _MAX_INT64),)
    )
    == 4 * coordinate_width
)
```

## Trade-offs and Limitations

With `u_x` distinct x-coordinates and `u_y` distinct y-coordinates, grid
construction and boundary inspection use `O(n + u_x * u_y)` exact-integer
operations and `O(u_x * u_y)` memory. At most 128 rectangles imply at most 256
coordinates on each axis and 65,025 compressed cells. This explicit cap favors
clarity and predictable memory over sparse-geometry scalability.

Compressed cells retain their actual coordinate widths and heights. The
returned Python integer can therefore exceed the signed-64 input range.
Half-open edges make overlap unambiguous: edge-touching covered cells share an
internal interface, while corner touching removes no positive boundary length.

Only total union perimeter is returned. The function does not return area,
boundary segments, polygon rings, holes as objects, connected components, or
source-rectangle attribution. Degenerate, floating-point, rotated, curved, and
large sparse geometries are outside this bounded representation.

## Related Snippets

<!-- catalog:related:start -->
- [Compute the Exact Union Area of Bounded Axis-Aligned Integer Rectangles](compute-the-exact-union-area-of-bounded-axis-aligned-integer-rectangles.md)
- [Coalesce Bounded Half-Open Integer Intervals Under an Explicit Adjacency Policy](coalesce-bounded-half-open-integer-intervals-under-an-explicit-adjacency-policy.md)
- [Find Peak Coverage Spans Across Bounded Half-Open Integer Intervals](find-peak-coverage-spans-across-bounded-half-open-integer-intervals.md)
<!-- catalog:related:end -->
