---
title: "Build a Bounded Balanced 2D k-d Tree for Exact Nearest-Point Queries"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - find-a-closest-pair-of-bounded-integer-points-with-earliest-index-pair-ties.md
  - match-strict-mutual-nearest-neighbors-with-a-comparison-budget.md
  - map-points-between-rectangular-coordinate-spaces.md
---

# Build a Bounded Balanced 2D k-d Tree for Exact Nearest-Point Queries

## Idea and Problem

Index a static set of two-dimensional integer points so repeated exact nearest-point queries can prune regions that cannot improve the current answer.

Construction alternates the x and y axes, sorting each active subset and taking
its lower median. Every immutable node records the bounding box of its whole
subtree. A query compares exact squared distances and skips a box only when its
minimum possible distance is strictly greater than the best distance already
seen.

Input positions remain identities. Equal coordinates are not deduplicated,
and equal-distance answers choose the smallest declaration index.

## When to Use

Use this index for a bounded, static collection of exact 2D integer points when
the same collection receives multiple nearest-point queries. It is especially
useful when an inspectable standard-library implementation and deterministic
duplicate handling matter more than the strongest possible construction cost.

Use a direct scan for a few queries or tiny inputs. Choose a maintained spatial
library for dynamic updates, higher dimensions, bulk queries, approximate
search, floating-point tolerance, or geographic coordinate systems.

## Implementation

```python
from dataclasses import dataclass
from itertools import product
from typing import Self

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_POINT_COUNT = 4_096

_Point = tuple[int, int]
_IndexedPoint = tuple[_Point, int]


@dataclass(frozen=True, slots=True)
class _KdNode:
    index: int
    point: _Point
    axis: int
    min_x: int
    max_x: int
    min_y: int
    max_y: int
    left: Self | None
    right: Self | None


@dataclass(frozen=True, slots=True)
class KdTree2D:
    root: _KdNode | None
    point_count: int


@dataclass(frozen=True, slots=True)
class NearestPoint:
    index: int
    point: _Point
    squared_distance: int


def _validate_point(value: object, *, name: str) -> _Point:
    if type(value) is not tuple:
        raise TypeError(f"{name} must be an exact tuple")
    if len(value) != 2:
        raise ValueError(f"{name} must contain exactly two coordinates")
    x, y = value
    if type(x) is not int or type(y) is not int:
        raise TypeError(f"{name} coordinates must be exact integers")
    if not _MIN_INT64 <= x <= _MAX_INT64:
        raise ValueError(f"{name}[0] is outside the signed 64-bit range")
    if not _MIN_INT64 <= y <= _MAX_INT64:
        raise ValueError(f"{name}[1] is outside the signed 64-bit range")
    return x, y


def _build_node(records: tuple[_IndexedPoint, ...], depth: int) -> _KdNode | None:
    if not records:
        return None

    axis = depth % 2
    other_axis = 1 - axis
    ordered = tuple(
        sorted(
            records,
            key=lambda record: (
                record[0][axis],
                record[0][other_axis],
                record[1],
            ),
        )
    )
    median = (len(ordered) - 1) // 2
    point, index = ordered[median]
    left = _build_node(ordered[:median], depth + 1)
    right = _build_node(ordered[median + 1 :], depth + 1)

    min_x = max_x = point[0]
    min_y = max_y = point[1]
    for child in (left, right):
        if child is not None:
            min_x = min(min_x, child.min_x)
            max_x = max(max_x, child.max_x)
            min_y = min(min_y, child.min_y)
            max_y = max(max_y, child.max_y)

    return _KdNode(
        index=index,
        point=point,
        axis=axis,
        min_x=min_x,
        max_x=max_x,
        min_y=min_y,
        max_y=max_y,
        left=left,
        right=right,
    )


def build_kd_tree_2d(points: tuple[_Point, ...]) -> KdTree2D:
    """Build a deterministic lower-median 2D k-d tree."""
    if type(points) is not tuple:
        raise TypeError("points must be an exact tuple")
    if len(points) > _MAX_POINT_COUNT:
        raise ValueError("point count exceeds the supported maximum")

    indexed: list[_IndexedPoint] = []
    for index, point in enumerate(points):
        indexed.append((_validate_point(point, name=f"points[{index}]"), index))
    root = _build_node(tuple(indexed), 0)
    return KdTree2D(root=root, point_count=len(indexed))


def _squared_distance(first: _Point, second: _Point) -> int:
    delta_x = first[0] - second[0]
    delta_y = first[1] - second[1]
    return delta_x * delta_x + delta_y * delta_y


def _box_distance(point: _Point, node: _KdNode) -> int:
    if point[0] < node.min_x:
        delta_x = node.min_x - point[0]
    elif point[0] > node.max_x:
        delta_x = point[0] - node.max_x
    else:
        delta_x = 0

    if point[1] < node.min_y:
        delta_y = node.min_y - point[1]
    elif point[1] > node.max_y:
        delta_y = point[1] - node.max_y
    else:
        delta_y = 0
    return delta_x * delta_x + delta_y * delta_y


def find_nearest_point(tree: KdTree2D, query: _Point) -> NearestPoint | None:
    """Return the nearest point under squared-distance and index ties."""
    if type(tree) is not KdTree2D:
        raise TypeError("tree must be an exact KdTree2D")
    query = _validate_point(query, name="query")
    if tree.root is None:
        return None

    best: tuple[int, int, _Point] | None = None

    def visit(node: _KdNode) -> None:
        nonlocal best
        candidate = (_squared_distance(query, node.point), node.index, node.point)
        if best is None or candidate[:2] < best[:2]:
            best = candidate

        if query[node.axis] <= node.point[node.axis]:
            children = node.left, node.right
        else:
            children = node.right, node.left

        for child in children:
            if child is not None and _box_distance(query, child) <= best[0]:
                visit(child)

    visit(tree.root)
    assert best is not None
    return NearestPoint(index=best[1], point=best[2], squared_distance=best[0])


```

## Example

```python
def brute_nearest(points: tuple[_Point, ...], query: _Point) -> NearestPoint | None:
    if not points:
        return None
    squared_distance, index = min(
        (_squared_distance(point, query), index) for index, point in enumerate(points)
    )
    return NearestPoint(index, points[index], squared_distance)


grid = ((0, 0), (0, 1), (1, 0), (1, 1))
queries = ((-1, 0), (0, 0), (1, 1), (2, 2))
checked = 0
for point_count in range(5):
    for points in product(grid, repeat=point_count):
        tree = build_kd_tree_2d(points)
        for query in queries:
            assert find_nearest_point(tree, query) == brute_nearest(points, query)
            checked += 1

duplicates = ((5, 5), (0, 0), (5, 5), (9, 9))
extremes = ((_MIN_INT64, _MIN_INT64), (_MAX_INT64, _MAX_INT64))

assert (
    checked,
    find_nearest_point(build_kd_tree_2d(duplicates), (5, 5)),
    find_nearest_point(build_kd_tree_2d(extremes), (0, 0)),
) == (
    1_364,
    NearestPoint(0, (5, 5), 0),
    NearestPoint(1, (_MAX_INT64, _MAX_INT64), 2 * _MAX_INT64**2),
)
```

## Trade-offs and Limitations

Repeated sorting makes construction `O(n log^2 n)` time. The tree, bounding
boxes and recursive build use `O(n)` retained and peak auxiliary memory. A
well-distributed static set often lets a query prune most nodes, but adversarial
geometry can force `O(n)` visits; this implementation makes no unconditional
logarithmic-query claim.

Squared Euclidean distance is exact even when the result exceeds signed 64-bit
range because Python integers grow as needed. Bounding-box equality deliberately
does not prune: a point at the same distance but a smaller declaration index
can still replace the current answer.

Only `KdTree2D` values returned by `build_kd_tree_2d` are supported query
inputs; manually forged dataclass values are not validated. The tree exposes
neither mutation nor its internal node type as a supported API. It does not
rebalance after changes, return every tie, find multiple or
radius-bounded neighbors, accept floats, normalize coordinates, or interpret a
coordinate reference system.

## Related Snippets

<!-- catalog:related:start -->
- [Find a Closest Pair of Bounded Integer Points with Earliest Index-Pair Ties](find-a-closest-pair-of-bounded-integer-points-with-earliest-index-pair-ties.md)
- [Match Strict Mutual Nearest Neighbors with a Comparison Budget](match-strict-mutual-nearest-neighbors-with-a-comparison-budget.md)
- [Map Points Between Rectangular Coordinate Spaces](map-points-between-rectangular-coordinate-spaces.md)
<!-- catalog:related:end -->
