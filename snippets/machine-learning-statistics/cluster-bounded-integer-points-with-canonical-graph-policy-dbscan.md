---
title: "Cluster Bounded Integer Points with Canonical Graph-Policy DBSCAN"
snippet_type: algorithm
use_cases:
  - data-transformation
  - testing
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - compute-exact-silhouette-coefficients-from-a-bounded-integer-dissimilarity-matrix.md
  - fit-exact-one-dimensional-k-means-with-contiguous-cluster-dp.md
  - compute-an-exact-adjusted-rand-index-from-two-bounded-integer-partitions.md
---

# Cluster Bounded Integer Points with Canonical Graph-Policy DBSCAN

## Idea and Problem

Separate dense regions, assign ambiguous border points deterministically, and identify noise in a small integer point set without floating-point distance decisions.

DBSCAN first marks a point as core when its closed squared-radius neighborhood,
including itself, has at least `min_points` members. Core points connected by
radius edges form clusters. Non-core points adjacent to a core component are
border points; the rest are noise.

Classic traversal-based implementations can assign a border point differently
when it touches multiple clusters. This graph policy numbers core components
by their smallest input index and attaches an ambiguous border point to the
smallest neighboring cluster ID, making the result reproducible for a fixed
input order.

## When to Use

Use this bounded implementation for small exact integer datasets, fixtures,
and reference calculations where deterministic labels and visible core/noise
evidence matter more than spatial-index performance. Duplicate points are
valid, and a zero radius can therefore still form a cluster.

Use a mature numerical library for large datasets, floating metrics, sample
weights, sparse neighborhoods, or indexing structures. If the chosen border
policy must match a particular external DBSCAN implementation, adopt and test
that implementation's documented ordering semantics instead.

## Implementation

```python
from dataclasses import dataclass
from random import Random

_MAX_DBSCAN_POINTS = 512
_MAX_DBSCAN_DIMENSIONS = 16
_MAX_DBSCAN_CELLS = 4_096
_MIN_SIGNED_32 = -(2**31)
_MAX_SIGNED_32 = 2**31 - 1
_MAX_EPSILON_SQUARED = 2**70


@dataclass(frozen=True, slots=True)
class CanonicalDbscanResult:
    labels: tuple[int | None, ...]
    core_indexes: tuple[int, ...]
    clusters: tuple[tuple[int, ...], ...]
    noise_indexes: tuple[int, ...]


def canonical_integer_dbscan(
    points: tuple[tuple[int, ...], ...],
    epsilon_squared: int,
    min_points: int,
) -> CanonicalDbscanResult:
    """Cluster a fixed point order under an explicit core-graph policy."""
    if type(points) is not tuple:
        raise TypeError("points must be an exact tuple")
    point_count = len(points)
    if not 1 <= point_count <= _MAX_DBSCAN_POINTS:
        raise ValueError("point count is outside 1..512")

    dimension: int | None = None
    for point_index, point in enumerate(points):
        if type(point) is not tuple:
            raise TypeError(f"points[{point_index}] must be an exact tuple")
        if dimension is None:
            dimension = len(point)
            if not 1 <= dimension <= _MAX_DBSCAN_DIMENSIONS:
                raise ValueError("point dimension is outside 1..16")
        elif len(point) != dimension:
            raise ValueError("all points must have the same dimension")
        for coordinate in point:
            if type(coordinate) is not int:
                raise TypeError("coordinates must be exact integers")
            if not _MIN_SIGNED_32 <= coordinate <= _MAX_SIGNED_32:
                raise ValueError("coordinate is outside the signed 32-bit range")
    if dimension is None or point_count * dimension > _MAX_DBSCAN_CELLS:
        raise ValueError("point matrix exceeds 4096 coordinate cells")

    if type(epsilon_squared) is not int:
        raise TypeError("epsilon_squared must be an exact integer")
    if not 0 <= epsilon_squared <= _MAX_EPSILON_SQUARED:
        raise ValueError("epsilon_squared is outside 0..2**70")
    if type(min_points) is not int:
        raise TypeError("min_points must be an exact integer")
    if not 1 <= min_points <= point_count:
        raise ValueError("min_points is outside 1..point_count")

    def neighbors(left: int, right: int) -> bool:
        squared_distance = 0
        for left_value, right_value in zip(points[left], points[right], strict=True):
            squared_distance += (left_value - right_value) ** 2
            if squared_distance > epsilon_squared:
                return False
        return True

    neighborhood_sizes = [1] * point_count
    for left in range(point_count):
        for right in range(left + 1, point_count):
            if neighbors(left, right):
                neighborhood_sizes[left] += 1
                neighborhood_sizes[right] += 1
    is_core = tuple(size >= min_points for size in neighborhood_sizes)

    parents = list(range(point_count))

    def find(item: int) -> int:
        while parents[item] != item:
            parents[item] = parents[parents[item]]
            item = parents[item]
        return item

    def union(left: int, right: int) -> None:
        left_root = find(left)
        right_root = find(right)
        if left_root == right_root:
            return
        smaller, larger = sorted((left_root, right_root))
        parents[larger] = smaller

    for left in range(point_count):
        if not is_core[left]:
            continue
        for right in range(left + 1, point_count):
            if is_core[right] and neighbors(left, right):
                union(left, right)

    core_indexes = tuple(index for index, core in enumerate(is_core) if core)
    component_roots = tuple(sorted({find(index) for index in core_indexes}))
    cluster_by_root = {root: cluster for cluster, root in enumerate(component_roots)}

    labels: list[int | None] = []
    for point_index in range(point_count):
        if is_core[point_index]:
            labels.append(cluster_by_root[find(point_index)])
            continue
        neighboring_clusters = {
            cluster_by_root[find(core_index)]
            for core_index in core_indexes
            if neighbors(point_index, core_index)
        }
        labels.append(min(neighboring_clusters) if neighboring_clusters else None)

    frozen_labels = tuple(labels)
    clusters = tuple(
        tuple(index for index, label in enumerate(frozen_labels) if label == cluster)
        for cluster in range(len(component_roots))
    )
    noise_indexes = tuple(index for index, label in enumerate(frozen_labels) if label is None)
    return CanonicalDbscanResult(
        labels=frozen_labels,
        core_indexes=core_indexes,
        clusters=clusters,
        noise_indexes=noise_indexes,
    )
```

## Example

```python
def complete_graph_oracle(
    points: tuple[tuple[int, ...], ...],
    epsilon_squared: int,
    min_points: int,
) -> CanonicalDbscanResult:
    adjacency = tuple(
        tuple(
            sum((left - right) ** 2 for left, right in zip(point, other, strict=True))
            <= epsilon_squared
            for other in points
        )
        for point in points
    )
    core = tuple(sum(row) >= min_points for row in adjacency)
    unseen = {index for index, is_core in enumerate(core) if is_core}
    components: list[tuple[int, ...]] = []
    while unseen:
        pending = [min(unseen)]
        unseen.remove(pending[0])
        component: list[int] = []
        while pending:
            current = pending.pop()
            component.append(current)
            discovered = {candidate for candidate in unseen if adjacency[current][candidate]}
            unseen.difference_update(discovered)
            pending.extend(discovered)
        components.append(tuple(sorted(component)))
    components.sort(key=min)

    labels: list[int | None] = []
    for point_index in range(len(points)):
        touching = tuple(
            cluster
            for cluster, members in enumerate(components)
            if any(adjacency[point_index][core_index] for core_index in members)
        )
        labels.append(min(touching) if touching else None)
    frozen_labels = tuple(labels)
    return CanonicalDbscanResult(
        labels=frozen_labels,
        core_indexes=tuple(index for index, is_core in enumerate(core) if is_core),
        clusters=tuple(
            tuple(index for index, label in enumerate(frozen_labels) if label == cluster)
            for cluster in range(len(components))
        ),
        noise_indexes=tuple(index for index, label in enumerate(frozen_labels) if label is None),
    )


sample = ((0,), (0,), (1,), (10,), (10,), (11,), (5,))
assert canonical_integer_dbscan(sample, epsilon_squared=1, min_points=3) == (
    CanonicalDbscanResult(
        labels=(0, 0, 0, 1, 1, 1, None),
        core_indexes=(0, 1, 2, 3, 4, 5),
        clusters=((0, 1, 2), (3, 4, 5)),
        noise_indexes=(6,),
    )
)

rng = Random(0xDB_5C_A0)
checked = 0
for _ in range(3_000):
    point_count = rng.randrange(1, 15)
    dimension = rng.randrange(1, 4)
    points = tuple(
        tuple(rng.randrange(-4, 5) for _ in range(dimension)) for _ in range(point_count)
    )
    epsilon_squared = rng.randrange(11)
    min_points = rng.randrange(1, point_count + 1)
    assert canonical_integer_dbscan(
        points,
        epsilon_squared,
        min_points,
    ) == complete_graph_oracle(points, epsilon_squared, min_points)
    checked += 1

assert checked == 3_000
```

## Trade-offs and Limitations

Two complete pair scans plus border assignment take `O(N²D)` time for `N`
points and `D` dimensions, while union-find, labels, and result storage use
`O(N)` auxiliary state. The independent Example oracle intentionally
materializes `O(N²)` adjacency for clarity. Exact integer distances avoid
rounding but can be slower than vectorized numeric operations.

Cluster IDs are canonical only relative to the supplied point order. The
smallest-ID border rule is an explicit policy, not a claim that all DBSCAN
libraries return the same partition for ambiguous border points. This version
does not provide floating metrics, weights, approximate neighbors, spatial
indexes, incremental updates, or a model for predicting labels of new points.

## Related Snippets

<!-- catalog:related:start -->
- [Compute Exact Silhouette Coefficients from a Bounded Integer Dissimilarity Matrix](compute-exact-silhouette-coefficients-from-a-bounded-integer-dissimilarity-matrix.md)
- [Fit Exact One-Dimensional k-Means with Contiguous-Cluster DP](fit-exact-one-dimensional-k-means-with-contiguous-cluster-dp.md)
- [Compute an Exact Adjusted Rand Index from Two Bounded Integer Partitions](compute-an-exact-adjusted-rand-index-from-two-bounded-integer-partitions.md)
<!-- catalog:related:end -->
