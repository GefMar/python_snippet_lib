---
title: "Find the Lexicographically First Eulerian Trail in a Bounded Directed Multigraph"
snippet_type: algorithm
use_cases:
  - data-transformation
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - partition-a-bounded-directed-graph-into-deterministic-strongly-connected-components.md
  - resolve-stable-ordering-constraints-with-topological-sort.md
  - find-one-deterministic-shortest-path-in-a-bounded-non-negatively-weighted-directed-graph.md
---

# Find the Lexicographically First Eulerian Trail in a Bounded Directed Multigraph

## Idea and Problem

Find one directed trail that consumes every declared edge exactly once while making parallel-edge choices deterministic.

Each edge is identified by its declaration index. Degree differences determine
whether an open trail has one forced start or a closed circuit may start at the
smallest vertex with an outgoing edge. Iterative Hierholzer traversal consumes
outgoing edge indexes in ascending order and appends them during backtracking;
reversing that postorder produces the lexicographically first complete
edge-index trail from the required start.

The result contains both views of the same trail. For `E > 0`, the edge-index
path has length `E`, the vertex path has length `E + 1`, and declared edge
`edge_index_path[i]` connects `vertex_path[i]` to `vertex_path[i + 1]`.

## When to Use

Use this algorithm for one complete bounded directed multigraph when every edge
must be traversed exactly once. It fits route reconstruction, chained fixtures,
and audit paths where parallel declarations must remain distinguishable and a
repeatable choice among several Eulerian trails matters.

Use a shortest-path algorithm when only one source-to-target route matters, or
a topological sort when edges express acyclic precedence instead of traversable
steps. Use a specialized graph library for dynamic graphs, very large inputs,
undirected edges, weighted route optimization, or enumeration of every valid
trail.

## Implementation

```python
_MAX_EULER_VERTICES = 10_000
_MAX_EULER_EDGES = 100_000


def find_lexicographically_first_eulerian_trail(
    vertex_count: int,
    edges: tuple[tuple[int, int], ...],
) -> tuple[tuple[int, ...], tuple[int, ...]] | None:
    """Return vertex and edge-index paths for the first complete trail."""
    if type(vertex_count) is not int:
        raise TypeError("vertex_count must be an exact non-boolean integer")
    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if not 0 <= vertex_count <= _MAX_EULER_VERTICES:
        raise ValueError("vertex_count is outside the supported range")
    if len(edges) > _MAX_EULER_EDGES:
        raise ValueError("edge count exceeds the supported limit")

    for edge_index, edge in enumerate(edges):
        if type(edge) is not tuple:
            raise TypeError(f"edges[{edge_index}] must be an exact tuple")
        if len(edge) != 2:
            raise ValueError(f"edges[{edge_index}] must contain two endpoints")
        source, target = edge
        if type(source) is not int:
            raise TypeError(f"edges[{edge_index}].source must be an exact integer")
        if type(target) is not int:
            raise TypeError(f"edges[{edge_index}].target must be an exact integer")
        if not 0 <= source < vertex_count:
            raise ValueError(f"edges[{edge_index}].source is outside the graph")
        if not 0 <= target < vertex_count:
            raise ValueError(f"edges[{edge_index}].target is outside the graph")

    if not edges:
        return (), ()

    outgoing: list[list[int]] = [[] for _ in range(vertex_count)]
    in_degree = [0] * vertex_count
    out_degree = [0] * vertex_count
    for edge_index, (source, target) in enumerate(edges):
        outgoing[source].append(edge_index)
        out_degree[source] += 1
        in_degree[target] += 1

    start_vertices: list[int] = []
    end_vertices: list[int] = []
    for vertex in range(vertex_count):
        difference = out_degree[vertex] - in_degree[vertex]
        if difference == 1:
            start_vertices.append(vertex)
        elif difference == -1:
            end_vertices.append(vertex)
        elif difference != 0:
            return None

    if len(start_vertices) == 1 and len(end_vertices) == 1:
        start = start_vertices[0]
    elif not start_vertices and not end_vertices:
        start = next(vertex for vertex in range(vertex_count) if out_degree[vertex])
    else:
        return None

    next_outgoing_offset = [0] * vertex_count
    walk_stack: list[tuple[int, int | None]] = [(start, None)]
    reverse_vertices: list[int] = []
    reverse_edges: list[int] = []

    while walk_stack:
        vertex = walk_stack[-1][0]
        offset = next_outgoing_offset[vertex]
        if offset < len(outgoing[vertex]):
            edge_index = outgoing[vertex][offset]
            next_outgoing_offset[vertex] += 1
            walk_stack.append((edges[edge_index][1], edge_index))
        else:
            vertex, incoming_edge = walk_stack.pop()
            reverse_vertices.append(vertex)
            if incoming_edge is not None:
                reverse_edges.append(incoming_edge)

    if len(reverse_edges) != len(edges):
        return None

    vertex_path = tuple(reversed(reverse_vertices))
    edge_index_path = tuple(reversed(reverse_edges))
    return vertex_path, edge_index_path
```

## Example

```python
def eulerian_trail_by_edge_index_search(
    vertex_count: int,
    edges: tuple[tuple[int, int], ...],
) -> tuple[tuple[int, ...], tuple[int, ...]] | None:
    if not edges:
        return (), ()

    in_degree = [0] * vertex_count
    out_degree = [0] * vertex_count
    for source, target in edges:
        out_degree[source] += 1
        in_degree[target] += 1

    starts: list[int] = []
    ends: list[int] = []
    for vertex in range(vertex_count):
        difference = out_degree[vertex] - in_degree[vertex]
        if difference == 1:
            starts.append(vertex)
        elif difference == -1:
            ends.append(vertex)
        elif difference != 0:
            return None

    if len(starts) == 1 and len(ends) == 1:
        start = starts[0]
    elif not starts and not ends:
        start = next(vertex for vertex in range(vertex_count) if out_degree[vertex])
    else:
        return None

    used = [False] * len(edges)

    def search(vertex: int, depth: int) -> tuple[int, ...] | None:
        if depth == len(edges):
            return ()
        for edge_index, (source, target) in enumerate(edges):
            if source != vertex or used[edge_index]:
                continue
            used[edge_index] = True
            suffix = search(target, depth + 1)
            used[edge_index] = False
            if suffix is not None:
                return edge_index, *suffix
        return None

    edge_index_path = search(start, 0)
    if edge_index_path is None:
        return None

    vertices = [start]
    for edge_index in edge_index_path:
        source, target = edges[edge_index]
        assert source == vertices[-1]
        vertices.append(target)
    return tuple(vertices), edge_index_path


def exercise_tiny_directed_multigraphs() -> int:
    from itertools import product

    edge_options = tuple(product(range(2), repeat=2))
    checked = 0
    for edge_count in range(8):
        for edges in product(edge_options, repeat=edge_count):
            assert find_lexicographically_first_eulerian_trail(
                2, edges
            ) == eulerian_trail_by_edge_index_search(2, edges)
            checked += 1
    return checked


open_trail = find_lexicographically_first_eulerian_trail(
    3,
    ((0, 1), (0, 2), (1, 0)),
)
circuit = find_lexicographically_first_eulerian_trail(
    3,
    ((0, 1), (0, 2), (1, 0), (2, 0)),
)
disconnected = find_lexicographically_first_eulerian_trail(
    4,
    ((0, 1), (1, 0), (2, 3), (3, 2)),
)

maximum_edges = ((0, 0),) * _MAX_EULER_EDGES
maximum_trail = find_lexicographically_first_eulerian_trail(1, maximum_edges)

try:
    find_lexicographically_first_eulerian_trail(True, ())
except TypeError:
    boolean_vertex_count_rejected = True
else:
    boolean_vertex_count_rejected = False

try:
    find_lexicographically_first_eulerian_trail(
        1,
        ((0, 0),) * (_MAX_EULER_EDGES + 1),
    )
except ValueError:
    edge_cap_enforced = True
else:
    edge_cap_enforced = False

assert (
    exercise_tiny_directed_multigraphs(),
    open_trail,
    circuit,
    disconnected,
    maximum_trail is not None,
    len(maximum_trail[0]) if maximum_trail is not None else 0,
    maximum_trail[1] == tuple(range(_MAX_EULER_EDGES)) if maximum_trail is not None else False,
    boolean_vertex_count_rejected,
    edge_cap_enforced,
) == (
    21_845,
    ((0, 1, 0, 2), (0, 2, 1)),
    ((0, 1, 0, 2, 0), (0, 2, 1, 3)),
    None,
    True,
    100_001,
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

Complete validation, degree checks, and traversal take `O(V + E)` time. Edge
indexes are appended to each adjacency list in declaration order, and a
per-vertex cursor consumes them without sorting or front deletion. Adjacency,
degrees, cursors, traversal stacks, and immutable results use `O(V + E)`
memory.

An open trail is possible only with one degree-plus-one start and one
degree-minus-one end. A circuit uses the smallest vertex with outgoing edges.
Those degree conditions are not sufficient across disconnected edge-bearing
regions, so the final consumed-edge count rejects an incomplete traversal.

Edge declaration order is observable: it defines edge identities and resolves
ties lexicographically. Reordering the same parallel or branching edges does
not change Eulerian existence, but it can renumber edges and select a different
trail. The empty edge set has no meaningful start and returns two empty tuples;
isolated vertices otherwise do not appear in a returned path.

The function handles one fixed directed multigraph. It does not accept
undirected or weighted edges, optimize distance, return a partial trail, update
the graph, enumerate every trail, or make arbitrary route-planning constraints
Eulerian.

## Related Snippets

<!-- catalog:related:start -->
- [Partition a Bounded Directed Graph into Deterministic Strongly Connected Components](partition-a-bounded-directed-graph-into-deterministic-strongly-connected-components.md)
- [Resolve Stable Ordering Constraints with Topological Sort](resolve-stable-ordering-constraints-with-topological-sort.md)
- [Find One Deterministic Shortest Path in a Bounded Non-Negatively Weighted Directed Graph](find-one-deterministic-shortest-path-in-a-bounded-non-negatively-weighted-directed-graph.md)
<!-- catalog:related:end -->
