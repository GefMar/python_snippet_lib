---
title: "Match Strict Mutual Nearest Neighbors with a Comparison Budget"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../data-processing/select-one-record-per-key-with-an-explicit-ranking-rule.md
---

# Match Strict Mutual Nearest Neighbors with a Comparison Budget

## Idea and Problem

Keep only point pairs that are each other's unique Euclidean nearest neighbor while enforcing a hard pair-comparison budget.

One-way nearest-neighbor selection can map many points to the same candidate.
Tracking the best result in both directions during one Cartesian scan adds a
bidirectional consistency condition without storing a full distance matrix.
Rejecting exact ties makes the result independent of arbitrary input-index
tie-breaking.

## When to Use

Use this algorithm for two finite, small point sets with the same coordinate
dimension when Euclidean distance is meaningful and unmatched points are
acceptable. Set the comparison budget from an application-level resource
limit before the scan. Use a spatial index, approximate-neighbor library, or
global assignment algorithm for large collections or different matching
semantics.

## Implementation

```python
import math
from collections.abc import Iterable
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class NeighborMatch:
    left_index: int
    right_index: int
    distance: float


def _materialize_points(
    points: Iterable[Iterable[int | float]],
    *,
    name: str,
) -> tuple[tuple[float, ...], ...]:
    materialized = []
    dimensions: int | None = None
    for point_index, point in enumerate(points):
        coordinates = []
        for coordinate in point:
            if isinstance(coordinate, bool) or not isinstance(coordinate, (int, float)):
                raise TypeError(f"{name}[{point_index}] coordinates must be numeric")
            try:
                numeric = float(coordinate)
            except OverflowError as exc:
                raise ValueError(
                    f"{name}[{point_index}] coordinates must be finite"
                ) from exc
            if not math.isfinite(numeric):
                raise ValueError(f"{name}[{point_index}] coordinates must be finite")
            coordinates.append(numeric)
        if not coordinates:
            raise ValueError(f"{name}[{point_index}] must not be empty")
        if dimensions is None:
            dimensions = len(coordinates)
        elif len(coordinates) != dimensions:
            raise ValueError(f"{name} points must have one common dimension")
        materialized.append(tuple(coordinates))
    return tuple(materialized)


def match_strict_mutual_nearest_neighbors(
    left: Iterable[Iterable[int | float]],
    right: Iterable[Iterable[int | float]],
    *,
    max_distance: int | float,
    max_comparisons: int,
) -> tuple[NeighborMatch, ...]:
    if isinstance(max_distance, bool) or not isinstance(max_distance, (int, float)):
        raise TypeError("max_distance must be numeric")
    try:
        distance_limit = float(max_distance)
    except OverflowError as exc:
        raise ValueError("max_distance must be finite") from exc
    if not math.isfinite(distance_limit) or distance_limit < 0.0:
        raise ValueError("max_distance must be finite and non-negative")
    if isinstance(max_comparisons, bool) or not isinstance(max_comparisons, int):
        raise TypeError("max_comparisons must be an integer")
    if max_comparisons <= 0:
        raise ValueError("max_comparisons must be positive")

    left_points = _materialize_points(left, name="left")
    right_points = _materialize_points(right, name="right")
    if not left_points or not right_points:
        return ()

    dimensions = len(left_points[0])
    if len(right_points[0]) != dimensions:
        raise ValueError("left and right point dimensions must match")

    comparison_count = len(left_points) * len(right_points)
    if comparison_count > max_comparisons:
        raise ValueError("point pair count exceeds max_comparisons")

    left_best = [math.inf] * len(left_points)
    left_best_right = [-1] * len(left_points)
    left_tied = [False] * len(left_points)
    right_best = [math.inf] * len(right_points)
    right_best_left = [-1] * len(right_points)
    right_tied = [False] * len(right_points)

    for left_index, left_point in enumerate(left_points):
        for right_index, right_point in enumerate(right_points):
            distance = math.dist(left_point, right_point)
            if not math.isfinite(distance):
                raise ValueError("computed distance must be finite")

            if distance < left_best[left_index]:
                left_best[left_index] = distance
                left_best_right[left_index] = right_index
                left_tied[left_index] = False
            elif distance == left_best[left_index]:
                left_tied[left_index] = True

            if distance < right_best[right_index]:
                right_best[right_index] = distance
                right_best_left[right_index] = left_index
                right_tied[right_index] = False
            elif distance == right_best[right_index]:
                right_tied[right_index] = True

    matches = []
    for left_index, right_index in enumerate(left_best_right):
        if left_tied[left_index] or right_tied[right_index]:
            continue
        if right_best_left[right_index] != left_index:
            continue
        distance = left_best[left_index]
        if distance <= distance_limit:
            matches.append(NeighborMatch(left_index, right_index, distance))
    return tuple(matches)
```

## Example

```python
matches = match_strict_mutual_nearest_neighbors(
    left=[(0, 0), (10, 0), (30, 0)],
    right=[(1, 0), (12, 0), (100, 0)],
    max_distance=3,
    max_comparisons=9,
)

tied = match_strict_mutual_nearest_neighbors(
    left=[(0,)],
    right=[(-1,), (1,)],
    max_distance=1,
    max_comparisons=2,
)

try:
    match_strict_mutual_nearest_neighbors(
        left=[(0, 0), (1, 1)],
        right=[(0, 0), (1, 1)],
        max_distance=1,
        max_comparisons=3,
    )
except ValueError:
    budget_enforced = True
else:
    budget_enforced = False

assert (matches, tied, budget_enforced) == (
    (NeighborMatch(0, 0, 1.0), NeighborMatch(1, 1, 2.0)),
    (),
    True,
)
```

## Trade-offs and Limitations

The function materializes both inputs, uses `O(n + m)` matching state, and
performs `n * m` Euclidean comparisons; coordinate work adds a factor equal to
the point dimension. The budget limits pairs, not dimensions or input
materialization, so callers must still provide finite, reasonably sized
iterables. Exact floating-point ties are rejected, while near-ties are not;
quantize inputs or use a domain-specific tolerance policy when required.
Mutual nearest neighbors are not a globally optimal one-to-one assignment and
can leave many points unmatched. The result also depends on coordinate scaling
and does not provide cosine, approximate, or learned-distance semantics.

## Related Snippets

<!-- catalog:related:start -->
- [Select One Record per Key with an Explicit Ranking Rule](../data-processing/select-one-record-per-key-with-an-explicit-ranking-rule.md)
<!-- catalog:related:end -->
