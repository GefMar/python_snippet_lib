---
title: "Assign Each Vertex to Its Nearest Source with Multi-Source Breadth-First Search"
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
  - compute-one-source-shortest-distances-in-a-bounded-zero-one-weighted-directed-graph.md
  - partition-a-bounded-undirected-graph-into-deterministic-components-with-union-find.md
  - traverse-a-parent-graph-with-breadth-first-search.md
---

# Assign Each Vertex to Its Nearest Source with Multi-Source Breadth-First Search

## Idea and Problem

Assign every reachable vertex in a simple undirected graph to its nearest member of a declared source set, returning both the source and unweighted distance while choosing the smallest source index on ties.

One multi-source breadth-first search first computes the minimum distance from
the complete source set. A second pass follows increasing distance layers. The
owner of each non-source vertex is the smallest owner among its neighbors in
the preceding layer. Separating distance discovery from owner propagation
makes the tie rule explicit and independent of queue and adjacency ordering.

## When to Use

Use this approach to partition every vertex of a static graph by nearest seed, such as
assigning network nodes to service locations, expanding labels from several
known vertices, or building an unweighted graph Voronoi partition. It is most
useful when every vertex needs a distance and deterministic source ownership.

Use an ordinary one-source breadth-first search when only one origin matters.
Use multi-source Dijkstra when edges have non-negative unequal weights. Choose
a different contract when paths, predecessor choices, directed edges, dynamic
updates, or a tie rule other than the smallest source index are required.

## Implementation

```python
from collections import deque
from dataclasses import dataclass

_MAX_NEAREST_VERTICES = 10_000
_MAX_NEAREST_EDGES = 100_000


@dataclass(frozen=True, slots=True)
class NearestSource:
    source: int | None
    distance: int | None


def assign_nearest_sources(
    vertex_count: int,
    edges: tuple[tuple[int, int], ...],
    sources: tuple[int, ...],
) -> tuple[NearestSource, ...]:
    """Return deterministic nearest-source ownership and distances."""
    if type(vertex_count) is not int:
        raise TypeError("vertex_count must be an exact non-boolean integer")
    if not 1 <= vertex_count <= _MAX_NEAREST_VERTICES:
        raise ValueError("vertex_count is outside the supported range")
    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if len(edges) > _MAX_NEAREST_EDGES:
        raise ValueError("edge count exceeds the supported limit")

    adjacency: list[list[int]] = [[] for _ in range(vertex_count)]
    seen_edges: set[tuple[int, int]] = set()
    for edge_index, edge in enumerate(edges):
        if type(edge) is not tuple or len(edge) != 2:
            raise TypeError(f"edges[{edge_index}] must be an exact two-item tuple")
        first, second = edge
        if type(first) is not int or type(second) is not int:
            raise TypeError(f"edges[{edge_index}] endpoints must be exact integers")
        if not 0 <= first < vertex_count or not 0 <= second < vertex_count:
            raise ValueError(f"edges[{edge_index}] endpoint is outside the vertex range")
        if first == second:
            raise ValueError(f"edges[{edge_index}] must not be a self-loop")
        normalized = (min(first, second), max(first, second))
        if normalized in seen_edges:
            raise ValueError(f"edges[{edge_index}] duplicates an undirected edge")
        seen_edges.add(normalized)
        adjacency[first].append(second)
        adjacency[second].append(first)

    if type(sources) is not tuple:
        raise TypeError("sources must be an exact tuple")
    if not sources:
        raise ValueError("sources must not be empty")
    seen_sources: set[int] = set()
    for source_index, source in enumerate(sources):
        if type(source) is not int:
            raise TypeError(f"sources[{source_index}] must be an exact integer")
        if not 0 <= source < vertex_count:
            raise ValueError(f"sources[{source_index}] is outside the vertex range")
        if source in seen_sources:
            raise ValueError(f"sources[{source_index}] duplicates a source")
        seen_sources.add(source)

    distances: list[int | None] = [None] * vertex_count
    layers: list[list[int]] = [list(sources)]
    pending = deque(sources)
    for source in sources:
        distances[source] = 0

    while pending:
        vertex = pending.popleft()
        distance = distances[vertex]
        if distance is None:
            raise RuntimeError("internal distance invariant failed")
        for neighbor in adjacency[vertex]:
            if distances[neighbor] is not None:
                continue
            neighbor_distance = distance + 1
            distances[neighbor] = neighbor_distance
            if neighbor_distance == len(layers):
                layers.append([])
            layers[neighbor_distance].append(neighbor)
            pending.append(neighbor)

    owners: list[int | None] = [None] * vertex_count
    for source in sources:
        owners[source] = source

    for distance in range(1, len(layers)):
        for vertex in layers[distance]:
            best_source: int | None = None
            for neighbor in adjacency[vertex]:
                if distances[neighbor] != distance - 1:
                    continue
                candidate = owners[neighbor]
                if candidate is not None and (best_source is None or candidate < best_source):
                    best_source = candidate
            if best_source is None:
                raise RuntimeError("internal owner invariant failed")
            owners[vertex] = best_source

    return tuple(
        NearestSource(source=owners[vertex], distance=distances[vertex])
        for vertex in range(vertex_count)
    )
```

## Example

```python
def assign_by_independent_searches(
    vertex_count: int,
    edges: tuple[tuple[int, int], ...],
    sources: tuple[int, ...],
) -> tuple[NearestSource, ...]:
    adjacency: list[list[int]] = [[] for _ in range(vertex_count)]
    for first, second in edges:
        adjacency[first].append(second)
        adjacency[second].append(first)

    distances_by_source: dict[int, tuple[int | None, ...]] = {}
    for source in sources:
        distances: list[int | None] = [None] * vertex_count
        distances[source] = 0
        pending = deque((source,))
        while pending:
            vertex = pending.popleft()
            distance = distances[vertex]
            if distance is None:
                raise RuntimeError("oracle distance invariant failed")
            for neighbor in adjacency[vertex]:
                if distances[neighbor] is None:
                    distances[neighbor] = distance + 1
                    pending.append(neighbor)
        distances_by_source[source] = tuple(distances)

    result: list[NearestSource] = []
    for vertex in range(vertex_count):
        candidates = (
            (distance, source)
            for source in sources
            if (distance := distances_by_source[source][vertex]) is not None
        )
        nearest = min(candidates, default=None)
        if nearest is None:
            result.append(NearestSource(source=None, distance=None))
        else:
            distance, source = nearest
            result.append(NearestSource(source=source, distance=distance))
    return tuple(result)


def decode_simple_graph(
    vertex_count: int,
    encoding: int,
) -> tuple[tuple[int, int], ...]:
    possible_edges = tuple(
        (first, second)
        for first in range(vertex_count)
        for second in range(first + 1, vertex_count)
    )
    return tuple(
        edge for edge_index, edge in enumerate(possible_edges) if encoding & (1 << edge_index)
    )


checked_assignments = 0
for small_vertex_count in range(1, 6):
    possible_edge_count = small_vertex_count * (small_vertex_count - 1) // 2
    for graph_encoding in range(1 << possible_edge_count):
        small_edges = decode_simple_graph(small_vertex_count, graph_encoding)
        for source_encoding in range(1, 1 << small_vertex_count):
            small_sources = tuple(
                vertex for vertex in range(small_vertex_count) if source_encoding & (1 << vertex)
            )
            expected = assign_by_independent_searches(
                small_vertex_count, small_edges, small_sources
            )
            assert (
                assign_nearest_sources(small_vertex_count, small_edges, small_sources) == expected
            )
            assert (
                assign_nearest_sources(
                    small_vertex_count, tuple(reversed(small_edges)), small_sources
                )
                == expected
            )
            assert (
                assign_nearest_sources(
                    small_vertex_count, small_edges, tuple(reversed(small_sources))
                )
                == expected
            )
            checked_assignments += 1

tie_edges = ((0, 2), (1, 2), (2, 3), (4, 5))
tie_expected = (
    NearestSource(0, 0),
    NearestSource(1, 0),
    NearestSource(0, 1),
    NearestSource(0, 2),
    NearestSource(None, None),
    NearestSource(None, None),
)
assert assign_nearest_sources(6, tie_edges, (1, 0)) == tie_expected

maximum_edges = tuple(
    (start, (start + offset) % _MAX_NEAREST_VERTICES)
    for offset in range(1, 11)
    for start in range(_MAX_NEAREST_VERTICES)
)
maximum_assignment = assign_nearest_sources(
    _MAX_NEAREST_VERTICES,
    maximum_edges,
    (0, _MAX_NEAREST_VERTICES - 1),
)


def rejects(vertex_count: object, edges: object, sources: object) -> bool:
    try:
        assign_nearest_sources(  # type: ignore[arg-type]
            vertex_count,
            edges,
            sources,
        )
    except (TypeError, ValueError):
        return True
    return False


assert (
    checked_assignments,
    len(maximum_edges),
    maximum_assignment[0],
    maximum_assignment[-1],
    maximum_assignment[_MAX_NEAREST_VERTICES // 2].distance is not None,
    rejects(True, (), (0,)),
    rejects(1, [], (0,)),
    rejects(2, ((0, True),), (0,)),
    rejects(2, ((0, 0),), (0,)),
    rejects(2, ((0, 1), (1, 0)), (0,)),
    rejects(1, (), ()),
    rejects(1, (), [0]),
    rejects(1, (), (True,)),
    rejects(2, (), (0, 0)),
    rejects(1, (), (1,)),
) == (
    32_767,
    _MAX_NEAREST_EDGES,
    NearestSource(0, 0),
    NearestSource(_MAX_NEAREST_VERTICES - 1, 0),
    True,
    True,
    True,
    True,
    True,
    True,
    True,
    True,
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

Validation, adjacency construction, distance discovery, and owner propagation
take `O(V + E)` time. The adjacency lists, normalized-edge set, queue, distance
layers, owner state, and immutable result use `O(V + E)` memory. Each edge is
examined during validation and in both traversal phases, without sorting the
edge or source collections.

The contract accepts 1-10,000 indexed vertices, at most 100,000 unique simple
undirected edges, and a non-empty tuple of distinct source indexes. Containers
must be exact tuples, and vertex counts, endpoints, and sources must be exact
non-Boolean integers. Every source owns itself at distance zero. Vertices in
components without a source receive `NearestSource(None, None)`.

The entire graph is copied before traversal. The function returns no paths
or predecessor tree, accepts no parallel edges, self-loops, weights, directed
edges, or changing graph, and provides no incremental reuse. The smallest-index
tie rule is fixed rather than configurable.

## Related Snippets

<!-- catalog:related:start -->
- [Compute One-Source Shortest Distances in a Bounded Zero-One Weighted Directed Graph](compute-one-source-shortest-distances-in-a-bounded-zero-one-weighted-directed-graph.md)
- [Partition a Bounded Undirected Graph into Deterministic Components with Union-Find](partition-a-bounded-undirected-graph-into-deterministic-components-with-union-find.md)
- [Traverse a Parent Graph with Breadth-First Search](traverse-a-parent-graph-with-breadth-first-search.md)
<!-- catalog:related:end -->
