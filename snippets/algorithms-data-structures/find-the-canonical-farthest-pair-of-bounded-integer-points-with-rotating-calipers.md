---
title: "Find the Canonical Farthest Pair of Bounded Integer Points with Rotating Calipers"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-a-canonical-convex-hull-for-bounded-integer-points.md
  - find-a-closest-pair-of-bounded-integer-points-with-earliest-index-pair-ties.md
  - compute-the-signed-shoelace-double-area-of-a-bounded-integer-vertex-walk.md
---

# Find the Canonical Farthest Pair of Bounded Integer Points with Rotating Calipers

## Idea and Problem

Find a canonical pair of coordinates with the greatest exact squared distance in one bounded integer point collection.

Only convex-hull vertices can form a farthest pair. A monotone-chain pass first
coalesces duplicates and removes interior and collinear non-extreme points. For
each directed hull edge, rotating calipers then advances an opposite support
vertex while the exact triangle double area increases.

Every diameter occurs among those antipodal candidates. The result normalizes
each endpoint pair lexicographically and selects the smallest pair when several
diameters have the same squared length.

## When to Use

Use this function for a static bounded set of exact two-dimensional integer
coordinates when an all-pairs scan would be unnecessarily quadratic. Squared
distance is useful for choosing maximally separated locations, measuring a
point-set diameter, or building deterministic geometry fixtures without a
floating-point square root.

Input order and duplicate multiplicity deliberately carry no identity. Use an
index-preserving algorithm when separate observations at equal coordinates or
an earliest input-position tie rule matter. Use direct pair enumeration as a
small reference oracle or when the collection is tiny.

## Implementation

```python
_MIN_FARTHEST_COORDINATE = -(1 << 63)
_MAX_FARTHEST_COORDINATE = (1 << 63) - 1
_MAX_FARTHEST_POINT_COUNT = 4_096

_Point = tuple[int, int]
_FarthestPair = tuple[_Point, _Point, int]


def _farthest_cross(origin: _Point, first: _Point, second: _Point) -> int:
    return (
        (first[0] - origin[0]) * (second[1] - origin[1])
        - (first[1] - origin[1]) * (second[0] - origin[0])
    )


def _canonical_farthest_hull(points: tuple[_Point, ...]) -> tuple[_Point, ...]:
    ordered = sorted(set(points))
    if len(ordered) <= 2:
        return tuple(ordered)

    lower: list[_Point] = []
    for point in ordered:
        while len(lower) >= 2 and _farthest_cross(lower[-2], lower[-1], point) <= 0:
            lower.pop()
        lower.append(point)

    upper: list[_Point] = []
    for point in reversed(ordered):
        while len(upper) >= 2 and _farthest_cross(upper[-2], upper[-1], point) <= 0:
            upper.pop()
        upper.append(point)

    return tuple(lower[:-1] + upper[:-1])


def canonical_farthest_point_pair(points: tuple[_Point, ...]) -> _FarthestPair:
    """Return canonical endpoint coordinates and their exact squared distance."""
    if type(points) is not tuple:
        raise TypeError("points must be an exact tuple")
    if not 1 <= len(points) <= _MAX_FARTHEST_POINT_COUNT:
        raise ValueError("point count is outside 1..4096")

    for point_index, point in enumerate(points):
        if type(point) is not tuple:
            raise TypeError(f"points[{point_index}] must be an exact tuple")
        if len(point) != 2:
            raise ValueError(f"points[{point_index}] must contain two coordinates")
        for coordinate_index, coordinate in enumerate(point):
            if type(coordinate) is not int:
                raise TypeError(
                    f"points[{point_index}][{coordinate_index}] must be an exact integer"
                )
            if not _MIN_FARTHEST_COORDINATE <= coordinate <= _MAX_FARTHEST_COORDINATE:
                raise ValueError(
                    f"points[{point_index}][{coordinate_index}] is outside signed 64-bit"
                )

    hull = _canonical_farthest_hull(points)
    if len(hull) == 1:
        return hull[0], hull[0], 0

    if len(hull) == 2:
        delta_x = hull[0][0] - hull[1][0]
        delta_y = hull[0][1] - hull[1][1]
        return hull[0], hull[1], delta_x * delta_x + delta_y * delta_y

    best_distance = -1
    best_endpoints = (hull[0], hull[1])
    opposite = 1
    hull_size = len(hull)

    for first in range(hull_size):
        next_first = (first + 1) % hull_size
        while True:
            next_opposite = (opposite + 1) % hull_size
            current_area = _farthest_cross(
                hull[first], hull[next_first], hull[opposite]
            )
            next_area = _farthest_cross(
                hull[first], hull[next_first], hull[next_opposite]
            )
            if next_area <= current_area:
                break
            opposite = next_opposite

        candidates = [(first, opposite), (next_first, opposite)]
        if next_area == current_area:
            candidates.extend(
                ((first, next_opposite), (next_first, next_opposite))
            )

        for left, right in candidates:
            if left == right:
                continue
            endpoints = tuple(sorted((hull[left], hull[right])))
            delta_x = endpoints[0][0] - endpoints[1][0]
            delta_y = endpoints[0][1] - endpoints[1][1]
            squared_distance = delta_x * delta_x + delta_y * delta_y
            if squared_distance > best_distance or (
                squared_distance == best_distance and endpoints < best_endpoints
            ):
                best_distance = squared_distance
                best_endpoints = endpoints

    return best_endpoints[0], best_endpoints[1], best_distance
```

## Example

```python
def farthest_point_pair_by_search(points: tuple[_Point, ...]) -> _FarthestPair:
    unique_points = tuple(sorted(set(points)))
    if len(unique_points) == 1:
        return unique_points[0], unique_points[0], 0

    candidates = []
    for first in range(len(unique_points)):
        for second in range(first + 1, len(unique_points)):
            delta_x = unique_points[first][0] - unique_points[second][0]
            delta_y = unique_points[first][1] - unique_points[second][1]
            candidates.append(
                (
                    delta_x * delta_x + delta_y * delta_y,
                    unique_points[first],
                    unique_points[second],
                )
            )
    distance, first, second = min(
        candidates,
        key=lambda candidate: (-candidate[0], candidate[1], candidate[2]),
    )
    return first, second, distance


def exercise_small_grid_collections() -> int:
    from itertools import combinations, product

    grid = tuple(product(range(-1, 3), repeat=2))
    checked = 0
    for point_count in range(1, 7):
        for selected in combinations(grid, point_count):
            permuted_with_duplicate = (*reversed(selected), selected[0])
            assert canonical_farthest_point_pair(
                permuted_with_duplicate
            ) == farthest_point_pair_by_search(permuted_with_duplicate)
            checked += 1
    return checked


checked_collections = exercise_small_grid_collections()
rectangle = ((4, 2), (0, 2), (4, 0), (0, 0), (2, 1), (0, 0))
collinear = ((2, 2), (-1, -1), (0, 0), (2, 2), (1, 1))

coordinate_width = _MAX_FARTHEST_COORDINATE - _MIN_FARTHEST_COORDINATE
boundary_square = (
    (_MAX_FARTHEST_COORDINATE, _MAX_FARTHEST_COORDINATE),
    (_MIN_FARTHEST_COORDINATE, _MIN_FARTHEST_COORDINATE),
    (_MAX_FARTHEST_COORDINATE, _MIN_FARTHEST_COORDINATE),
    (_MIN_FARTHEST_COORDINATE, _MAX_FARTHEST_COORDINATE),
)

rejected = 0
invalid_inputs = (
    (),
    ((0, 0), (True, 1)),
    ((0, 0, 1),),
    ((_MIN_FARTHEST_COORDINATE - 1, 0),),
)
for invalid_input in invalid_inputs:
    try:
        canonical_farthest_point_pair(invalid_input)
    except (TypeError, ValueError):
        rejected += 1

assert (
    checked_collections == 14_892
    and canonical_farthest_point_pair((((3, -2)),)) == ((3, -2), (3, -2), 0)
    and canonical_farthest_point_pair(rectangle) == ((0, 0), (4, 2), 20)
    and canonical_farthest_point_pair(collinear) == ((-1, -1), (2, 2), 18)
    and canonical_farthest_point_pair(boundary_square)
    == (
        (_MIN_FARTHEST_COORDINATE, _MIN_FARTHEST_COORDINATE),
        (_MAX_FARTHEST_COORDINATE, _MAX_FARTHEST_COORDINATE),
        2 * coordinate_width**2,
    )
    and rejected == len(invalid_inputs)
)
```

## Trade-offs and Limitations

For `N` declared points and `H` hull vertices, validation, deduplication,
sorting, and monotone-chain construction take `O(N * log(N))` time. Rotating
calipers takes `O(H)` exact orientation and squared-distance operations. The
set, sorted list, partial hulls, and final hull use `O(N)` references.

Python integers preserve exact coordinate differences, cross products, and
distances even when signed-64-bit input coordinates produce wider intermediate
values. Their arithmetic cost still grows with operand bit length.

Duplicate coordinates are coalesced, input order is ignored, and collinear
non-extreme points are removed. The result therefore identifies coordinate
values rather than original input positions. A single unique point is paired
with itself; two unique or wholly collinear extremes need no caliper pass.

The function does not return a square root, retain duplicate identities,
accept floating-point tolerances, compute a closest pair, maintain a dynamic
point set, interpret coordinate reference systems, or handle three dimensions.

## Related Snippets

<!-- catalog:related:start -->
- [Compute a Canonical Convex Hull for Bounded Integer Points](compute-a-canonical-convex-hull-for-bounded-integer-points.md)
- [Find a Closest Pair of Bounded Integer Points with Earliest Index-Pair Ties](find-a-closest-pair-of-bounded-integer-points-with-earliest-index-pair-ties.md)
- [Compute the Signed Shoelace Double Area of a Bounded Integer Vertex Walk](compute-the-signed-shoelace-double-area-of-a-bounded-integer-vertex-walk.md)
<!-- catalog:related:end -->
