---
title: "Partition the Edges of a Bounded Simple Undirected Graph into Vertex-Biconnected Blocks"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - find-articulation-points-and-bridges-in-a-bounded-undirected-graph.md
  - partition-a-bounded-undirected-graph-into-deterministic-components-with-union-find.md
---

# Partition the Edges of a Bounded Simple Undirected Graph into Vertex-Biconnected Blocks

## Idea and Problem

Partition every edge of a static simple undirected graph into a maximal vertex-biconnected block.

Different blocks may share articulation vertices, but every original edge
index belongs to exactly one block. A bridge forms a singleton block, while an
isolated vertex contributes no block. Within a nontrivial block, removing one
vertex does not disconnect the remaining incident structure.

A depth-first traversal records discovery and low-link positions. Each edge is
pushed onto a stack when traversal first reaches it. When a child cannot reach
above its parent, the edges through that child are popped together as one
vertex-biconnected block.

## When to Use

Use this algorithm when downstream work needs the complete edge partition of
a bounded, fully available graph: for example, to isolate cyclic regions,
identify bridge-only sections, or construct a block-cut representation in a
later step. Original edge indexes make the result easy to join back to caller
metadata without embedding that metadata in the graph algorithm.

Use an articulation-point or bridge finder when only cut locations matter.
Use a different decomposition for edge-biconnected components, directed
graphs, multigraphs, or a graph that changes between queries.

## Implementation

```python
_MAX_BICONNECTED_VERTEX_COUNT = 256
_MAX_BICONNECTED_EDGE_COUNT = 2_048


def partition_vertex_biconnected_edge_blocks(
    vertex_count: int,
    edges: tuple[tuple[int, int], ...],
) -> tuple[tuple[int, ...], ...]:
    """Return canonical blocks containing every original edge index once."""
    if type(vertex_count) is not int:
        raise TypeError("vertex_count must be an exact integer")
    if not 0 <= vertex_count <= _MAX_BICONNECTED_VERTEX_COUNT:
        raise ValueError("vertex_count is outside 0..256")
    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if len(edges) > _MAX_BICONNECTED_EDGE_COUNT:
        raise ValueError("edge count exceeds 2048")

    adjacency: list[list[tuple[int, int]]] = [[] for _ in range(vertex_count)]
    seen_pairs: set[tuple[int, int]] = set()
    for edge_index, edge in enumerate(edges):
        if type(edge) is not tuple:
            raise TypeError(f"edges[{edge_index}] must be an exact tuple")
        if len(edge) != 2:
            raise ValueError(f"edges[{edge_index}] must contain two endpoints")

        first, second = edge
        if type(first) is not int or type(second) is not int:
            raise TypeError(f"edges[{edge_index}] endpoints must be exact integers")
        if not 0 <= first < vertex_count or not 0 <= second < vertex_count:
            raise ValueError(f"edges[{edge_index}] endpoint is outside the graph")
        if first == second:
            raise ValueError(f"edges[{edge_index}] is a self-loop")

        pair = (min(first, second), max(first, second))
        if pair in seen_pairs:
            raise ValueError(f"edges[{edge_index}] duplicates an undirected pair")
        seen_pairs.add(pair)
        adjacency[first].append((second, edge_index))
        adjacency[second].append((first, edge_index))

    discovery = [-1] * vertex_count
    low = [-1] * vertex_count
    edge_stack: list[int] = []
    blocks: list[tuple[int, ...]] = []
    next_discovery = 0

    def visit(vertex: int, parent_edge: int) -> None:
        nonlocal next_discovery
        discovery[vertex] = next_discovery
        low[vertex] = next_discovery
        next_discovery += 1

        for neighbor, edge_index in adjacency[vertex]:
            if edge_index == parent_edge:
                continue
            if discovery[neighbor] == -1:
                edge_stack.append(edge_index)
                visit(neighbor, edge_index)
                low[vertex] = min(low[vertex], low[neighbor])
                if low[neighbor] >= discovery[vertex]:
                    block: list[int] = []
                    while True:
                        stacked_edge = edge_stack.pop()
                        block.append(stacked_edge)
                        if stacked_edge == edge_index:
                            break
                    blocks.append(tuple(sorted(block)))
            elif discovery[neighbor] < discovery[vertex]:
                edge_stack.append(edge_index)
                low[vertex] = min(low[vertex], discovery[neighbor])

    for root in range(vertex_count):
        if discovery[root] == -1:
            visit(root, -1)

    return tuple(sorted(blocks))
```

## Example

```python
def cycle_oracle_blocks(
    vertex_count: int,
    edges: tuple[tuple[int, int], ...],
) -> tuple[tuple[int, ...], ...]:
    """Join edge indexes that occur together on any enumerated simple cycle."""
    from itertools import permutations

    parents = list(range(len(edges)))

    def find(edge_index: int) -> int:
        while parents[edge_index] != edge_index:
            parents[edge_index] = parents[parents[edge_index]]
            edge_index = parents[edge_index]
        return edge_index

    def union(first: int, second: int) -> None:
        first_root = find(first)
        second_root = find(second)
        if first_root != second_root:
            parents[second_root] = first_root

    edge_indexes = {
        (min(first, second), max(first, second)): edge_index
        for edge_index, (first, second) in enumerate(edges)
    }
    for cycle_length in range(3, vertex_count + 1):
        for cycle in permutations(range(vertex_count), cycle_length):
            if cycle[0] != min(cycle) or cycle[1] > cycle[-1]:
                continue
            pairs = tuple(
                (
                    min(cycle[offset], cycle[(offset + 1) % cycle_length]),
                    max(cycle[offset], cycle[(offset + 1) % cycle_length]),
                )
                for offset in range(cycle_length)
            )
            if all(pair in edge_indexes for pair in pairs):
                indexes = tuple(edge_indexes[pair] for pair in pairs)
                for edge_index in indexes[1:]:
                    union(indexes[0], edge_index)

    groups: dict[int, list[int]] = {}
    for edge_index in range(len(edges)):
        groups.setdefault(find(edge_index), []).append(edge_index)
    return tuple(sorted(tuple(group) for group in groups.values()))


def exercise_every_graph_through_four_vertices() -> int:
    from itertools import combinations

    checked = 0
    for vertex_count in range(5):
        possible_edges = tuple(combinations(range(vertex_count), 2))
        for flags in range(1 << len(possible_edges)):
            edges = tuple(
                edge
                for edge_index, edge in enumerate(possible_edges)
                if flags & (1 << edge_index)
            )
            expected = cycle_oracle_blocks(vertex_count, edges)
            assert partition_vertex_biconnected_edge_blocks(
                vertex_count, edges
            ) == expected
            reversed_endpoints = tuple((second, first) for first, second in edges)
            assert partition_vertex_biconnected_edge_blocks(
                vertex_count, reversed_endpoints
            ) == expected
            checked += 1
    return checked


figure_eight_with_bridge = (
    (0, 1),
    (1, 2),
    (2, 0),
    (2, 3),
    (3, 4),
    (4, 2),
    (4, 5),
)
depth_boundary = tuple((vertex, vertex + 1) for vertex in range(255))

duplicate_rejected = False
try:
    partition_vertex_biconnected_edge_blocks(2, ((0, 1), (1, 0)))
except ValueError:
    duplicate_rejected = True

self_loop_rejected = False
try:
    partition_vertex_biconnected_edge_blocks(1, ((0, 0),))
except ValueError:
    self_loop_rejected = True

assert (
    exercise_every_graph_through_four_vertices() == 76
    and partition_vertex_biconnected_edge_blocks(0, ()) == ()
    and partition_vertex_biconnected_edge_blocks(7, figure_eight_with_bridge)
    == ((0, 1, 2), (3, 4, 5), (6,))
    and partition_vertex_biconnected_edge_blocks(256, depth_boundary)
    == tuple((edge_index,) for edge_index in range(255))
    and duplicate_rejected
    and self_loop_rejected
)
```

## Trade-offs and Limitations

Validation, adjacency construction, and Tarjan traversal take `O(V + E)` time
and retain `O(V + E)` references. Sorting edge indexes within blocks and the
block sequence adds `O(E * log(E))` worst-case time. The recursive traversal
can reach one frame per vertex; the explicit 256-vertex cap keeps it below the
usual Python recursion limit.

The result uses original edge indexes, so endpoint orientation does not change
a block but reordering edge declarations changes their identifiers. Arbitrary-
precision integers are unnecessary, and each edge is pushed and popped at
most once.

The function accepts only a static simple loopless graph. It rejects parallel
endpoint pairs and does not compute edge-biconnected components, articulation
vertices as a separate result, a block-cut tree, paths, directed blocks, or
incremental updates.

## Related Snippets

<!-- catalog:related:start -->
- [Find Articulation Points and Bridges in a Bounded Undirected Graph](find-articulation-points-and-bridges-in-a-bounded-undirected-graph.md)
- [Partition a Bounded Undirected Graph into Deterministic Components with Union-Find](partition-a-bounded-undirected-graph-into-deterministic-components-with-union-find.md)
<!-- catalog:related:end -->
