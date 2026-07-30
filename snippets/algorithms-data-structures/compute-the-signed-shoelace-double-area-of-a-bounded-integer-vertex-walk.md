---
title: "Compute the Signed Shoelace Double Area of a Bounded Integer Vertex Walk"
snippet_type: algorithm
use_cases:
  - data-transformation
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-a-canonical-convex-hull-for-bounded-integer-points.md
  - classify-a-bounded-integer-point-under-the-even-odd-polygonal-fill-rule.md
  - classify-the-exact-intersection-of-two-closed-integer-line-segments.md
---

# Compute the Signed Shoelace Double Area of a Bounded Integer Vertex Walk

## Idea and Problem

Compute the exact algebraic double area enclosed by an ordered integer vertex walk without floating-point arithmetic.

The shoelace sum adds the cross product of each vertex and its successor,
including the implicit edge from the last vertex back to the first. Keeping
twice the area avoids division: odd results represent exact half-unit areas,
and the sign preserves the supplied traversal direction when that direction
has an unambiguous geometric meaning.

## When to Use

Use this function for bounded geometry fixtures, lattice calculations, or a
validated polygon pipeline that needs an exact signed area primitive. For a
simple, non-degenerate polygon in ordinary Cartesian coordinates with the
positive y-axis pointing upward, a positive result means counterclockwise
order, a negative result means clockwise order, and the absolute result is
twice the geometric area.

The function deliberately accepts an algebraic vertex walk rather than proving
that it is a simple polygon. Repeated points, collinear edges, and
self-intersections can make lobes cancel. Validate topology separately when
filled-region area, containment, or boundary orientation is the actual goal.

## Implementation

```python
_MIN_COORDINATE = -(1 << 31)
_MAX_COORDINATE = (1 << 31) - 1
_MAX_VERTEX_COUNT = 4_096


def signed_shoelace_double_area(
    vertices: tuple[tuple[int, int], ...],
) -> int:
    """Return the exact signed shoelace sum for one implicitly closed walk."""
    if type(vertices) is not tuple:
        raise TypeError("vertices must be an exact tuple")
    if not 3 <= len(vertices) <= _MAX_VERTEX_COUNT:
        raise ValueError("vertex count is outside 3..4096")

    for vertex_index, vertex in enumerate(vertices):
        if type(vertex) is not tuple:
            raise TypeError(f"vertices[{vertex_index}] must be an exact tuple")
        if len(vertex) != 2:
            raise ValueError(f"vertices[{vertex_index}] must contain two coordinates")
        for coordinate_index, coordinate in enumerate(vertex):
            if type(coordinate) is not int:
                raise TypeError(
                    f"vertices[{vertex_index}][{coordinate_index}] must be an exact integer"
                )
            if not _MIN_COORDINATE <= coordinate <= _MAX_COORDINATE:
                raise ValueError(
                    f"vertices[{vertex_index}][{coordinate_index}] is outside signed 32-bit"
                )

    if vertices[0] == vertices[-1]:
        raise ValueError("the closing vertex is implicit and must not be repeated")

    signed_double_area = 0
    previous_x, previous_y = vertices[-1]
    for current_x, current_y in vertices:
        signed_double_area += previous_x * current_y - previous_y * current_x
        previous_x, previous_y = current_x, current_y
    return signed_double_area
```

## Example

```python
def triangle_determinant(
    first: tuple[int, int],
    second: tuple[int, int],
    third: tuple[int, int],
) -> int:
    return (
        (second[0] - first[0]) * (third[1] - first[1])
        - (second[1] - first[1]) * (third[0] - first[0])
    )


def exercise_triangle_grid() -> int:
    from itertools import product

    grid = tuple(product(range(-1, 2), repeat=2))
    checked = 0
    for first, second, third in product(grid, repeat=3):
        if first == third:
            continue
        assert signed_shoelace_double_area((first, second, third)) == (
            triangle_determinant(first, second, third)
        )
        checked += 1
    return checked


checked_triangles = exercise_triangle_grid()

rectangle = ((0, 0), (4, 0), (4, 3), (0, 3))
rotated = rectangle[2:] + rectangle[:2]
translated = tuple((x - 7, y + 11) for x, y in rectangle)
bow_tie = ((0, 0), (2, 2), (0, 2), (2, 0))

boundary_square = (
    (_MIN_COORDINATE, _MIN_COORDINATE),
    (_MAX_COORDINATE, _MIN_COORDINATE),
    (_MAX_COORDINATE, _MAX_COORDINATE),
    (_MIN_COORDINATE, _MAX_COORDINATE),
)
boundary_width = _MAX_COORDINATE - _MIN_COORDINATE

closing_vertex_rejected = False
try:
    signed_shoelace_double_area(((0, 0), (1, 0), (0, 1), (0, 0)))
except ValueError:
    closing_vertex_rejected = True

assert (
    checked_triangles == 648
    and signed_shoelace_double_area(rectangle) == 24
    and signed_shoelace_double_area(tuple(reversed(rectangle))) == -24
    and signed_shoelace_double_area(rotated) == 24
    and signed_shoelace_double_area(translated) == 24
    and signed_shoelace_double_area(bow_tie) == 0
    and signed_shoelace_double_area(boundary_square) == 2 * boundary_width**2
    and closing_vertex_rejected
)
```

## Trade-offs and Limitations

Validation and accumulation take `O(N)` time for `N` vertices and use `O(1)`
auxiliary space. Coordinate products and the accumulated sum use exact Python
integers, so the signed-32-bit coordinate cap bounds input shape rather than
forcing the result into a fixed-width type.

The result is algebraic and signed. The function does not take an absolute
value, divide by two, infer a screen-coordinate convention, or remove repeated
interior points. It rejects a repeated closing vertex so one walk has a single
representation, but it does not check distinctness otherwise.

No simplicity, edge-intersection, hole, containment, convexity, centroid, or
filled-union-area validation is performed. A zero sum can mean a degenerate
simple boundary or cancellation inside a self-intersecting walk; it is not by
itself a diagnosis of the input geometry.

## Related Snippets

<!-- catalog:related:start -->
- [Compute a Canonical Convex Hull for Bounded Integer Points](compute-a-canonical-convex-hull-for-bounded-integer-points.md)
- [Classify a Bounded Integer Point Under the Even-Odd Polygonal Fill Rule](classify-a-bounded-integer-point-under-the-even-odd-polygonal-fill-rule.md)
- [Classify the Exact Intersection of Two Closed Integer Line Segments](classify-the-exact-intersection-of-two-closed-integer-line-segments.md)
<!-- catalog:related:end -->
