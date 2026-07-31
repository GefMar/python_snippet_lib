---
title: "Compute the Exact Minimum Enclosing Circle of Bounded Integer Points"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - compute-a-canonical-convex-hull-for-bounded-integer-points.md
  - find-a-closest-pair-of-bounded-integer-points-with-earliest-index-pair-ties.md
  - find-the-canonical-farthest-pair-of-bounded-integer-points-with-rotating-calipers.md
---

# Compute the Exact Minimum Enclosing Circle of Bounded Integer Points

## Idea and Problem

Find the smallest circle containing every bounded integer point while keeping its center and squared radius as exact rational values.

A minimum enclosing circle is determined by one boundary point, two points at
opposite ends of a diameter, or three noncollinear boundary points. For a
deliberately bounded input, enumerating all such supports and testing every
candidate avoids floating-point tolerances and the control-flow subtleties of
randomized incremental algorithms.

The returned squared radius stays exact even when the radius itself is
irrational. Duplicate input points do not change the geometry.

## When to Use

Use this implementation for small integer point sets when a reproducible exact
covering disk matters more than asymptotic performance. It is useful as a
geometry oracle, for bounded fixture generation, or when downstream code can
compare squared distances without taking square roots.

Use a proven near-linear floating-point implementation for large point clouds.
Use a different optimization model when points have weights, positive radii,
uncertainty regions, or live in more than two dimensions.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction
from itertools import combinations, permutations

_MAX_CIRCLE_POINTS = 64
_MIN_SIGNED_32 = -(1 << 31)
_MAX_SIGNED_32 = (1 << 31) - 1

IntegerPoint = tuple[int, int]


@dataclass(frozen=True, slots=True)
class ExactCircle:
    center_x: Fraction
    center_y: Fraction
    radius_squared: Fraction


def _circle_from_support(
    support: tuple[IntegerPoint, ...],
) -> ExactCircle | None:
    if len(support) == 1:
        x, y = support[0]
        return ExactCircle(Fraction(x), Fraction(y), Fraction(0))

    if len(support) == 2:
        (first_x, first_y), (second_x, second_y) = support
        center_x = Fraction(first_x + second_x, 2)
        center_y = Fraction(first_y + second_y, 2)
        radius_squared = (center_x - first_x) ** 2 + (center_y - first_y) ** 2
        return ExactCircle(center_x, center_y, radius_squared)

    (ax, ay), (bx, by), (cx, cy) = support
    determinant = 2 * (ax * (by - cy) + bx * (cy - ay) + cx * (ay - by))
    if determinant == 0:
        return None

    squared_a = ax * ax + ay * ay
    squared_b = bx * bx + by * by
    squared_c = cx * cx + cy * cy
    center_x = Fraction(
        squared_a * (by - cy)
        + squared_b * (cy - ay)
        + squared_c * (ay - by),
        determinant,
    )
    center_y = Fraction(
        squared_a * (cx - bx)
        + squared_b * (ax - cx)
        + squared_c * (bx - ax),
        determinant,
    )
    radius_squared = (center_x - ax) ** 2 + (center_y - ay) ** 2
    return ExactCircle(center_x, center_y, radius_squared)


def _circle_contains(circle: ExactCircle, point: IntegerPoint) -> bool:
    offset_x = Fraction(point[0]) - circle.center_x
    offset_y = Fraction(point[1]) - circle.center_y
    return offset_x * offset_x + offset_y * offset_y <= circle.radius_squared


def compute_exact_minimum_enclosing_circle(
    points: tuple[IntegerPoint, ...],
) -> ExactCircle:
    """Return the exact smallest circle containing every declared point."""
    if type(points) is not tuple:
        raise TypeError("points must be an exact tuple")
    if not 1 <= len(points) <= _MAX_CIRCLE_POINTS:
        raise ValueError("point count is outside 1..64")

    for index, point in enumerate(points):
        if type(point) is not tuple or len(point) != 2:
            raise TypeError(f"points[{index}] must be an exact coordinate pair")
        x, y = point
        if type(x) is not int or type(y) is not int:
            raise TypeError(f"points[{index}] coordinates must be exact integers")
        if not _MIN_SIGNED_32 <= x <= _MAX_SIGNED_32:
            raise ValueError(f"points[{index}].x is outside signed 32-bit range")
        if not _MIN_SIGNED_32 <= y <= _MAX_SIGNED_32:
            raise ValueError(f"points[{index}].y is outside signed 32-bit range")

    unique_points = tuple(sorted(set(points)))
    best: ExactCircle | None = None
    for support_size in (1, 2, 3):
        for support in combinations(unique_points, support_size):
            candidate = _circle_from_support(support)
            if candidate is None:
                continue
            if not all(_circle_contains(candidate, point) for point in unique_points):
                continue
            if best is None or (
                candidate.radius_squared,
                candidate.center_x,
                candidate.center_y,
            ) < (best.radius_squared, best.center_x, best.center_y):
                best = candidate

    if best is None:
        raise AssertionError("a finite point set must have an enclosing circle")
    return best
```

## Example

```python
single = compute_exact_minimum_enclosing_circle(((3, -2), (3, -2)))
segment = compute_exact_minimum_enclosing_circle(((-2, 0), (0, 0), (2, 0)))
triangle = compute_exact_minimum_enclosing_circle(((0, 0), (4, 0), (2, 3)))
square_points = ((0, 0), (0, 2), (2, 0), (2, 2))
square = compute_exact_minimum_enclosing_circle(square_points)

assert single == ExactCircle(Fraction(3), Fraction(-2), Fraction(0))
assert segment == ExactCircle(Fraction(0), Fraction(0), Fraction(4))
assert triangle == ExactCircle(Fraction(2), Fraction(5, 6), Fraction(169, 36))
assert square == ExactCircle(Fraction(1), Fraction(1), Fraction(2))
assert all(
    compute_exact_minimum_enclosing_circle(tuple(order)) == square
    for order in permutations(square_points)
)

grid = tuple((x, y) for x in range(-1, 2) for y in range(-1, 2))
checked = 0
for mask in range(1, 1 << len(grid)):
    points = tuple(point for index, point in enumerate(grid) if mask >> index & 1)
    circle = compute_exact_minimum_enclosing_circle(points)
    assert all(_circle_contains(circle, point) for point in points)

    shifted = tuple((x + 5, y - 7) for x, y in points)
    shifted_circle = compute_exact_minimum_enclosing_circle(shifted)
    assert shifted_circle == ExactCircle(
        circle.center_x + 5,
        circle.center_y - 7,
        circle.radius_squared,
    )

    reflected = tuple((-x, y) for x, y in points)
    reflected_circle = compute_exact_minimum_enclosing_circle(reflected)
    assert reflected_circle == ExactCircle(
        -circle.center_x,
        circle.center_y,
        circle.radius_squared,
    )
    checked += 1

assert checked == 511 and square.radius_squared < segment.radius_squared
```

## Trade-offs and Limitations

With `U` unique points, there are `O(U³)` one-, two-, and three-point supports,
and each candidate performs an `O(U)` containment scan. The resulting
`O(U⁴)` time is intentionally traded for a compact exact algorithm; deduplication
and candidate storage use `O(U)` auxiliary memory.

Rational arithmetic can produce large numerators and denominators near the
coordinate bounds. The function returns squared radius because an exact radius
would often require an irrational representation. It does not expose boundary
support points, approximate tolerances, a streaming update, randomized
expected-linear performance, weighted disks, three-dimensional spheres, or a
large-input spatial optimization.

## Related Snippets

<!-- catalog:related:start -->
- [Compute a Canonical Convex Hull for Bounded Integer Points](compute-a-canonical-convex-hull-for-bounded-integer-points.md)
- [Find a Closest Pair of Bounded Integer Points with Earliest Index-Pair Ties](find-a-closest-pair-of-bounded-integer-points-with-earliest-index-pair-ties.md)
- [Find the Canonical Farthest Pair of Bounded Integer Points with Rotating Calipers](find-the-canonical-farthest-pair-of-bounded-integer-points-with-rotating-calipers.md)
<!-- catalog:related:end -->
