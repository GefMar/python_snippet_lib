---
title: "Count Exact Shortest Paths from One Source in a Bounded Positively Weighted Directed Graph"
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
  - find-one-deterministic-shortest-path-in-a-bounded-non-negatively-weighted-directed-graph.md
  - compute-bounded-bellman-ford-distances-and-detect-reachable-negative-cycles.md
  - count-exact-source-to-target-paths-in-a-bounded-dag.md
---

# Count Exact Shortest Paths from One Source in a Bounded Positively Weighted Directed Graph

## Idea and Problem

Return the exact shortest distance and number of distinct shortest vertex paths from one source to every vertex in a bounded directed graph.

Dijkstra's heap establishes distances in increasing order. A strictly shorter
candidate replaces a target's distance and count, while an equal candidate
adds the predecessor's count. Because every edge weight is positive, a
shortest-path predecessor always has a smaller distance and is processed
before its target; no later equal-distance cycle can invalidate the count.

## When to Use

Use this function for one static positively weighted graph when both distance
and the number of optimal alternatives matter. It fits bounded route analysis,
redundancy checks, and independent validation where materializing every
shortest path would be unnecessarily expensive.

Use a single-witness Dijkstra implementation when only one route is needed.
Use Bellman-Ford for negative weights, and define a different counting policy
before admitting zero-weight cycles because they can make the predecessor
relation cyclic. Use a specialized graph library for repeated or dynamic
queries over larger graphs.

## Implementation

```python
from heapq import heappop, heappush

_MAX_SHORTEST_COUNT_VERTEX_COUNT = 10_000
_MAX_SHORTEST_COUNT_EDGE_COUNT = 100_000
_MAX_SHORTEST_COUNT_EDGE_WEIGHT = (1 << 63) - 1

_WeightedDirectedEdge = tuple[int, int, int]
_DistanceAndCount = tuple[int | None, int]


def count_shortest_paths_from_source(
    vertex_count: int,
    edges: tuple[_WeightedDirectedEdge, ...],
    *,
    source: int,
) -> tuple[_DistanceAndCount, ...]:
    """Return one exact distance and shortest vertex-path count per vertex."""
    if type(vertex_count) is not int:
        raise TypeError("vertex_count must be an exact integer")
    if not 1 <= vertex_count <= _MAX_SHORTEST_COUNT_VERTEX_COUNT:
        raise ValueError("vertex_count is outside 1..10000")
    if type(source) is not int:
        raise TypeError("source must be an exact integer")
    if not 0 <= source < vertex_count:
        raise ValueError("source is outside the graph")
    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if len(edges) > _MAX_SHORTEST_COUNT_EDGE_COUNT:
        raise ValueError("edge count exceeds 100000")

    adjacency: list[list[tuple[int, int]]] = [[] for _ in range(vertex_count)]
    seen_pairs: set[tuple[int, int]] = set()
    for edge_index, edge in enumerate(edges):
        if type(edge) is not tuple:
            raise TypeError(f"edges[{edge_index}] must be an exact tuple")
        if len(edge) != 3:
            raise ValueError(f"edges[{edge_index}] must contain two endpoints and a weight")

        edge_source, target, weight = edge
        if type(edge_source) is not int or type(target) is not int:
            raise TypeError(f"edges[{edge_index}] endpoints must be exact integers")
        if type(weight) is not int:
            raise TypeError(f"edges[{edge_index}].weight must be an exact integer")
        if not 0 <= edge_source < vertex_count or not 0 <= target < vertex_count:
            raise ValueError(f"edges[{edge_index}] endpoint is outside the graph")
        if not 1 <= weight <= _MAX_SHORTEST_COUNT_EDGE_WEIGHT:
            raise ValueError(f"edges[{edge_index}].weight is outside the supported range")

        pair = (edge_source, target)
        if pair in seen_pairs:
            raise ValueError(f"edges[{edge_index}] duplicates a directed endpoint pair")
        seen_pairs.add(pair)
        adjacency[edge_source].append((target, weight))

    distances: list[int | None] = [None] * vertex_count
    path_counts = [0] * vertex_count
    distances[source] = 0
    path_counts[source] = 1
    frontier = [(0, source)]

    while frontier:
        distance, vertex = heappop(frontier)
        if distances[vertex] != distance:
            continue

        for target, weight in adjacency[vertex]:
            candidate = distance + weight
            known = distances[target]
            if known is None or candidate < known:
                distances[target] = candidate
                path_counts[target] = path_counts[vertex]
                heappush(frontier, (candidate, target))
            elif candidate == known:
                path_counts[target] += path_counts[vertex]

    return tuple(zip(distances, path_counts, strict=True))
```

## Example

```python
def count_shortest_paths_by_simple_search(
    vertex_count: int,
    edges: tuple[_WeightedDirectedEdge, ...],
    source: int,
) -> tuple[_DistanceAndCount, ...]:
    adjacency: list[list[tuple[int, int]]] = [[] for _ in range(vertex_count)]
    for edge_source, target, weight in edges:
        adjacency[edge_source].append((target, weight))

    distances: list[int | None] = [None] * vertex_count
    counts = [0] * vertex_count
    used = [False] * vertex_count
    used[source] = True

    def visit(vertex: int, distance: int) -> None:
        known = distances[vertex]
        if known is None or distance < known:
            distances[vertex] = distance
            counts[vertex] = 1
        elif distance == known:
            counts[vertex] += 1

        for target, weight in adjacency[vertex]:
            if used[target]:
                continue
            used[target] = True
            visit(target, distance + weight)
            used[target] = False

    visit(source, 0)
    return tuple(zip(distances, counts, strict=True))


def count_shortest_paths_by_relaxation(
    vertex_count: int,
    edges: tuple[_WeightedDirectedEdge, ...],
    source: int,
) -> tuple[_DistanceAndCount, ...]:
    distances: list[int | None] = [None] * vertex_count
    distances[source] = 0
    for _ in range(vertex_count - 1):
        changed = False
        for edge_source, target, weight in edges:
            source_distance = distances[edge_source]
            if source_distance is None:
                continue
            candidate = source_distance + weight
            if distances[target] is None or candidate < distances[target]:
                distances[target] = candidate
                changed = True
        if not changed:
            break

    path_counts = [0] * vertex_count
    path_counts[source] = 1
    reachable = sorted(
        (distance, vertex)
        for vertex, distance in enumerate(distances)
        if distance is not None
    )
    outgoing: list[list[tuple[int, int]]] = [[] for _ in range(vertex_count)]
    for edge_source, target, weight in edges:
        outgoing[edge_source].append((target, weight))
    for distance, vertex in reachable:
        for target, weight in outgoing[vertex]:
            if distances[target] == distance + weight:
                path_counts[target] += path_counts[vertex]
    return tuple(zip(distances, path_counts, strict=True))


def exercise_every_three_vertex_graph() -> int:
    possible_edges = tuple(
        (edge_source, target, 1)
        for edge_source in range(3)
        for target in range(3)
    )
    checked = 0
    for edge_flags in range(1 << len(possible_edges)):
        edges = tuple(
            edge
            for edge_index, edge in enumerate(possible_edges)
            if edge_flags & (1 << edge_index)
        )
        for source in range(3):
            expected = count_shortest_paths_by_simple_search(3, edges, source)
            assert count_shortest_paths_from_source(
                3,
                tuple(reversed(edges)),
                source=source,
            ) == expected
            assert count_shortest_paths_by_relaxation(3, edges, source) == expected
            checked += 1
    return checked


def build_maximum_edge_graph() -> tuple[_WeightedDirectedEdge, ...]:
    from itertools import islice

    vertex_count = 317
    return tuple(
        islice(
            (
                (edge_source, target, 1)
                for edge_source in range(vertex_count)
                for target in range(vertex_count)
            ),
            _MAX_SHORTEST_COUNT_EDGE_COUNT,
        )
    )


diamond = (
    (0, 1, 2),
    (0, 2, 2),
    (1, 3, 3),
    (2, 3, 3),
    (3, 4, _MAX_SHORTEST_COUNT_EDGE_WEIGHT),
    (0, 0, 1),
)
maximum_vertices = count_shortest_paths_from_source(
    _MAX_SHORTEST_COUNT_VERTEX_COUNT,
    (),
    source=_MAX_SHORTEST_COUNT_VERTEX_COUNT - 1,
)
maximum_edges = build_maximum_edge_graph()
maximum_edge_result = count_shortest_paths_from_source(
    317,
    maximum_edges,
    source=0,
)

duplicate_rejected = False
try:
    count_shortest_paths_from_source(
        2,
        ((0, 1, 1), (0, 1, 2)),
        source=0,
    )
except ValueError:
    duplicate_rejected = True

zero_weight_rejected = False
try:
    count_shortest_paths_from_source(1, ((0, 0, 0),), source=0)
except ValueError:
    zero_weight_rejected = True

assert (
    exercise_every_three_vertex_graph(),
    count_shortest_paths_from_source(6, diamond, source=0),
    len(maximum_vertices),
    maximum_vertices[-1],
    maximum_vertices[0],
    len(maximum_edges),
    maximum_edge_result[0],
    maximum_edge_result[-1],
    duplicate_rejected,
    zero_weight_rejected,
) == (
    1_536,
    (
        (0, 1),
        (2, 1),
        (2, 1),
        (5, 2),
        (5 + _MAX_SHORTEST_COUNT_EDGE_WEIGHT, 2),
        (None, 0),
    ),
    _MAX_SHORTEST_COUNT_VERTEX_COUNT,
    (0, 1),
    (None, 0),
    _MAX_SHORTEST_COUNT_EDGE_COUNT,
    (0, 1),
    (1, 1),
    True,
    True,
)
```

## Trade-offs and Limitations

For `V` vertices and `E` edges, validation takes expected `O(E)` set work.
The binary heap performs `O((V + E) * log(V))` work because unique directed
endpoint pairs imply `E <= V**2`; adjacency, validation state, distances,
counts, and the heap use `O(V + E)` references.

Edge weights are signed-64-bit positive integers, while accumulated distances
and counts remain exact Python integers beyond fixed-width ranges. Their
arithmetic and storage costs grow with operand bit length. Edge declaration
order does not affect distances or counts.

Counts distinguish vertex paths, and rejecting parallel endpoint pairs makes
that identity unambiguous. Positive self-loops are accepted but cannot improve
a shortest distance. The function excludes zero or negative weights,
parallel edges, path witnesses, enumeration, all-pairs queries, dynamic graph
updates, and persistent indexes.

## Related Snippets

<!-- catalog:related:start -->
- [Find One Deterministic Shortest Path in a Bounded Non-Negatively Weighted Directed Graph](find-one-deterministic-shortest-path-in-a-bounded-non-negatively-weighted-directed-graph.md)
- [Compute Bounded Bellman-Ford Distances and Detect Reachable Negative Cycles](compute-bounded-bellman-ford-distances-and-detect-reachable-negative-cycles.md)
- [Count Exact Source-to-Target Paths in a Bounded DAG](count-exact-source-to-target-paths-in-a-bounded-dag.md)
<!-- catalog:related:end -->
