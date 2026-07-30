---
title: "Compute One-Source Shortest Distances in a Bounded Zero-One Weighted Directed Graph"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - find-one-deterministic-shortest-path-in-a-bounded-non-negatively-weighted-directed-graph.md
  - compute-bounded-bellman-ford-distances-and-detect-reachable-negative-cycles.md
  - compute-bounded-all-pairs-shortest-distances-with-floyd-warshall.md
---

# Compute One-Source Shortest Distances in a Bounded Zero-One Weighted Directed Graph

## Idea and Problem

Compute every distance from one indexed source when each directed edge costs exactly zero or one.

A deque replaces Dijkstra's priority queue for this restricted weight set.
Relaxing a zero-cost edge places its target at the front, while relaxing a
one-cost edge places its target at the back. The deque therefore finalizes
vertices in non-decreasing distance order. Stale entries are ignored, so each
vertex's outgoing edges are scanned only once.

The result uses `None` for an unreachable vertex and a non-negative integer for every reachable distance. Parallel edges and self-loops are ordinary directed edges.

## When to Use

Use zero-one BFS for a materialized graph whose only transition costs are zero
and one, such as a state graph that distinguishes free moves from charged
moves. It gives all distances from one source without a general-purpose heap.

Use ordinary breadth-first search when every edge has the same cost. Use Dijkstra's algorithm for other non-negative weights and Bellman-Ford for negative edges.
Choose a separate contract when paths or predecessor tie-breaking, rather than distances alone, are required.

## Implementation

```python
from collections import deque

_MAX_ZERO_ONE_VERTICES = 4_096
_MAX_ZERO_ONE_EDGES = 65_536


def compute_zero_one_shortest_distances(
    vertex_count: int,
    edges: tuple[tuple[int, int, int], ...],
    *,
    source: int,
) -> tuple[int | None, ...]:
    """Return exact shortest distances from source in a zero-one graph."""
    if type(vertex_count) is not int:
        raise TypeError("vertex_count must be an exact non-boolean integer")
    if not 1 <= vertex_count <= _MAX_ZERO_ONE_VERTICES:
        raise ValueError("vertex_count is outside the supported range")
    if type(source) is not int:
        raise TypeError("source must be an exact non-boolean integer")
    if not 0 <= source < vertex_count:
        raise ValueError("source is outside the vertex range")
    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if len(edges) > _MAX_ZERO_ONE_EDGES:
        raise ValueError("edge count exceeds the supported limit")

    adjacency: list[list[tuple[int, int]]] = [[] for _ in range(vertex_count)]
    for edge_index, edge in enumerate(edges):
        if type(edge) is not tuple or len(edge) != 3:
            raise TypeError(f"edges[{edge_index}] must be an exact three-item tuple")
        start, stop, weight = edge
        if type(start) is not int or type(stop) is not int:
            raise TypeError(f"edges[{edge_index}] endpoints must be exact integers")
        if not 0 <= start < vertex_count or not 0 <= stop < vertex_count:
            raise ValueError(f"edges[{edge_index}] endpoint is outside the vertex range")
        if type(weight) is not int:
            raise TypeError(f"edges[{edge_index}] weight must be an exact integer")
        if weight not in (0, 1):
            raise ValueError(f"edges[{edge_index}] weight must be zero or one")
        adjacency[start].append((stop, weight))

    distances: list[int | None] = [None] * vertex_count
    distances[source] = 0
    pending = deque(((source, 0),))
    finalized = [False] * vertex_count

    while pending:
        vertex, queued_distance = pending.popleft()
        if finalized[vertex] or distances[vertex] != queued_distance:
            continue
        finalized[vertex] = True

        for target, weight in adjacency[vertex]:
            if finalized[target]:
                continue
            candidate = queued_distance + weight
            current = distances[target]
            if current is not None and candidate >= current:
                continue
            distances[target] = candidate
            entry = (target, candidate)
            if weight == 0:
                pending.appendleft(entry)
            else:
                pending.append(entry)

    return tuple(distances)
```

## Example

```python
def shortest_distances_by_repeated_relaxation(
    vertex_count: int,
    edges: tuple[tuple[int, int, int], ...],
    source: int,
) -> tuple[int | None, ...]:
    distances: list[int | None] = [None] * vertex_count
    distances[source] = 0
    for _ in range(vertex_count - 1):
        previous = distances
        updated = previous.copy()
        for start, stop, weight in edges:
            if previous[start] is None:
                continue
            candidate = previous[start] + weight
            if updated[stop] is None or candidate < updated[stop]:
                updated[stop] = candidate
        distances = updated
        if distances == previous:
            break
    return tuple(distances)


def decode_tiny_graph(vertex_count: int, encoding: int) -> tuple[tuple[int, int, int], ...]:
    edges: list[tuple[int, int, int]] = []
    for start in range(vertex_count):
        for stop in range(vertex_count):
            encoding, state = divmod(encoding, 3)
            if state:
                edges.append((start, stop, state - 1))
    return tuple(edges)


checked_graph_sources = 0
for vertex_count in range(1, 4):
    for encoding in range(3 ** (vertex_count * vertex_count)):
        tiny_edges = decode_tiny_graph(vertex_count, encoding)
        for source in range(vertex_count):
            assert compute_zero_one_shortest_distances(
                vertex_count, tiny_edges, source=source
            ) == shortest_distances_by_repeated_relaxation(
                vertex_count, tiny_edges, source
            )
            checked_graph_sources += 1

special_edges = ((0, 0, 0), (0, 1, 1), (0, 1, 0), (1, 1, 1), (1, 2, 1))
special_distances = compute_zero_one_shortest_distances(4, special_edges, source=0)
assert special_distances == (0, 0, 1, None)

maximum_edges = tuple(
    (
        index % _MAX_ZERO_ONE_VERTICES,
        (index + 1) % _MAX_ZERO_ONE_VERTICES,
        (index // _MAX_ZERO_ONE_VERTICES) % 2,
    )
    for index in range(_MAX_ZERO_ONE_EDGES)
)
maximum_distances = compute_zero_one_shortest_distances(
    _MAX_ZERO_ONE_VERTICES, maximum_edges, source=0
)

rejected = 0
for arguments in (
    (True, (), 0),
    (1, (), True),
    (1, ((0, 0, 2),), 0),
    (1, ((0, 1, 0),), 0),
    (1, ((0, True, 0),), 0),
):
    try:
        compute_zero_one_shortest_distances(arguments[0], arguments[1], source=arguments[2])
    except (TypeError, ValueError):
        rejected += 1

assert (
    checked_graph_sources,
    len(maximum_edges),
    maximum_distances[0],
    maximum_distances[-1],
    rejected,
) == (59_214, _MAX_ZERO_ONE_EDGES, 0, 0, 5)
```

## Trade-offs and Limitations

Validation, adjacency construction, and traversal take `O(V + E)` time. Each
vertex is finalized once, each outgoing edge is scanned once, and every
successful relaxation adds at most one stale-checkable deque entry. The
adjacency lists, distances, finalized flags, and deque use `O(V + E)` memory.

The contract accepts 1-4,096 indexed vertices and at most 65,536 exact
three-item edge tuples. Endpoints are zero-based indexes and weights are exact
non-Boolean integers restricted to zero and one. Input edge order and duplicate
edges cannot change the distances, but the complete graph is copied into
adjacency lists before traversal.

Only distances are returned. The function does not reconstruct paths, define
predecessor ties, accept negative or general weights, mutate the supplied edge
tuple, maintain a changing graph, or reuse preprocessing across sources.

## Related Snippets

<!-- catalog:related:start -->
- [Find One Deterministic Shortest Path in a Bounded Non-Negatively Weighted Directed Graph](find-one-deterministic-shortest-path-in-a-bounded-non-negatively-weighted-directed-graph.md)
- [Compute Bounded Bellman-Ford Distances and Detect Reachable Negative Cycles](compute-bounded-bellman-ford-distances-and-detect-reachable-negative-cycles.md)
- [Compute Bounded All-Pairs Shortest Distances with Floyd-Warshall](compute-bounded-all-pairs-shortest-distances-with-floyd-warshall.md)
<!-- catalog:related:end -->
