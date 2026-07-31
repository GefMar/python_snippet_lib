---
title: "Build a Deterministic Fundamental Cycle Basis for a Bounded Undirected Graph"
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
  - build-a-deterministic-minimum-spanning-forest-with-kruskals-algorithm.md
  - solve-a-bounded-affine-gf-2-system-with-a-canonical-particular-solution-and-nullspace-basis.md
  - partition-the-edges-of-a-bounded-simple-undirected-graph-into-vertex-biconnected-blocks.md
---

# Build a Deterministic Fundamental Cycle Basis for a Bounded Undirected Graph

## Idea and Problem

Represent every even-degree edge subset of a bounded undirected graph as an exclusive-or combination of reproducible fundamental cycles.

Processing normalized edges in lexicographic order with union-find selects one
spanning forest. Every rejected edge is a chord whose endpoints already have a
unique path through that forest. The chord together with this tree path is a
cycle. There is one such cycle for every edge outside the forest, so these
cycles are independent over GF(2) and span the graph's complete cycle space.

The result keeps each normalized chord beside its start-to-end tree path. Its
outer order therefore follows chord order and does not depend on input edge
orientation or declaration order.

## When to Use

Use a fundamental basis when a complete, bounded graph snapshot needs a compact
set of cycles for parity reasoning, independent test fixtures, electrical or
flow constraints, or encoding arbitrary even-degree edge sets. It is also a
useful structural oracle: a graph with `V` vertices, `E` edges, and `C`
connected components must yield exactly `E - V + C` basis cycles.

Use a minimum-weight cycle-basis algorithm when total cycle weight matters.
Use a dedicated simple-cycle enumerator when every simple cycle is required,
and use a dynamic graph structure when edges change between queries.

## Implementation

```python
from dataclasses import dataclass

_MAX_CYCLE_VERTICES = 128
_MAX_CYCLE_EDGES = 2_048
_MAX_CYCLE_EDGE_REFERENCES = 262_144

Edge = tuple[int, int]


@dataclass(frozen=True, slots=True)
class FundamentalCycle:
    chord: Edge
    tree_path: tuple[Edge, ...]


def build_fundamental_cycle_basis(
    vertex_count: int,
    edges: tuple[Edge, ...],
) -> tuple[FundamentalCycle, ...]:
    """Return lexicographic-forest fundamental cycles ordered by chord."""
    if type(vertex_count) is not int:
        raise TypeError("vertex_count must be an exact integer")
    if not 0 <= vertex_count <= _MAX_CYCLE_VERTICES:
        raise ValueError("vertex_count is outside 0..128")
    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if len(edges) > _MAX_CYCLE_EDGES:
        raise ValueError("edge count exceeds 2,048")

    normalized_edges: list[Edge] = []
    seen_edges: set[Edge] = set()
    for index, raw_edge in enumerate(edges):
        if type(raw_edge) is not tuple or len(raw_edge) != 2:
            raise TypeError(f"edges[{index}] must be an exact endpoint pair")
        first, second = raw_edge
        if type(first) is not int or type(second) is not int:
            raise TypeError(f"edges[{index}] endpoints must be exact integers")
        if not 0 <= first < vertex_count or not 0 <= second < vertex_count:
            raise ValueError(f"edges[{index}] has an endpoint outside the graph")
        if first == second:
            raise ValueError(f"edges[{index}] is a loop")

        edge = (first, second) if first < second else (second, first)
        if edge in seen_edges:
            raise ValueError(f"edges[{index}] duplicates an undirected edge")
        seen_edges.add(edge)
        normalized_edges.append(edge)

    normalized_edges.sort()
    parent = list(range(vertex_count))

    def find(vertex: int) -> int:
        while parent[vertex] != vertex:
            parent[vertex] = parent[parent[vertex]]
            vertex = parent[vertex]
        return vertex

    forest_edges: list[Edge] = []
    chords: list[Edge] = []
    for first, second in normalized_edges:
        first_root = find(first)
        second_root = find(second)
        if first_root == second_root:
            chords.append((first, second))
            continue
        if first_root > second_root:
            first_root, second_root = second_root, first_root
        parent[second_root] = first_root
        forest_edges.append((first, second))

    adjacency: list[list[int]] = [[] for _ in range(vertex_count)]
    for first, second in forest_edges:
        adjacency[first].append(second)
        adjacency[second].append(first)

    basis: list[FundamentalCycle] = []
    output_references = 0
    for chord in chords:
        start, goal = chord
        predecessor = [-1] * vertex_count
        predecessor[start] = start
        pending = [start]
        while pending:
            vertex = pending.pop()
            if vertex == goal:
                break
            for neighbor in adjacency[vertex]:
                if predecessor[neighbor] == -1:
                    predecessor[neighbor] = vertex
                    pending.append(neighbor)

        if predecessor[goal] == -1:
            raise AssertionError("a chord must have a path through the forest")

        reversed_path: list[Edge] = []
        vertex = goal
        while vertex != start:
            previous = predecessor[vertex]
            edge = (previous, vertex) if previous < vertex else (vertex, previous)
            reversed_path.append(edge)
            vertex = previous
        tree_path = tuple(reversed(reversed_path))

        output_references += 1 + len(tree_path)
        if output_references > _MAX_CYCLE_EDGE_REFERENCES:
            raise ValueError("cycle basis exceeds 262,144 edge references")
        basis.append(FundamentalCycle(chord=chord, tree_path=tree_path))

    return tuple(basis)
```

## Example

```python
def component_count(vertex_count: int, edges: tuple[Edge, ...]) -> int:
    adjacency = [[] for _ in range(vertex_count)]
    for first, second in edges:
        adjacency[first].append(second)
        adjacency[second].append(first)
    unseen = set(range(vertex_count))
    components = 0
    while unseen:
        components += 1
        pending = [unseen.pop()]
        while pending:
            for neighbor in adjacency[pending.pop()]:
                if neighbor in unseen:
                    unseen.remove(neighbor)
                    pending.append(neighbor)
    return components


def cycle_masks(
    edges: tuple[Edge, ...],
    basis: tuple[FundamentalCycle, ...],
) -> tuple[int, ...]:
    edge_bits = {edge: 1 << index for index, edge in enumerate(edges)}
    masks: list[int] = []
    for cycle in basis:
        cycle_edges = (cycle.chord, *cycle.tree_path)
        assert len(set(cycle_edges)) == len(cycle_edges)
        mask = 0
        vertex_parity = 0
        for first, second in cycle_edges:
            mask |= edge_bits[(first, second)]
            vertex_parity ^= (1 << first) | (1 << second)
        assert vertex_parity == 0
        masks.append(mask)
    return tuple(masks)


def complete_cycle_space(edges: tuple[Edge, ...]) -> set[int]:
    cycle_space: set[int] = set()
    for edge_subset in range(1 << len(edges)):
        vertex_parity = 0
        for index, (first, second) in enumerate(edges):
            if (edge_subset >> index) & 1:
                vertex_parity ^= (1 << first) | (1 << second)
        if vertex_parity == 0:
            cycle_space.add(edge_subset)
    return cycle_space


exhaustive_graphs = 0
for vertex_count in range(6):
    possible_edges = tuple(
        (first, second)
        for first in range(vertex_count)
        for second in range(first + 1, vertex_count)
    )
    for graph_mask in range(1 << len(possible_edges)):
        graph_edges = tuple(
            edge for index, edge in enumerate(possible_edges) if (graph_mask >> index) & 1
        )
        basis = build_fundamental_cycle_basis(vertex_count, graph_edges)
        masks = cycle_masks(graph_edges, basis)
        span = {0}
        for mask in masks:
            span |= {value ^ mask for value in tuple(span)}

        components = component_count(vertex_count, graph_edges)
        expected_rank = len(graph_edges) - vertex_count + components
        assert len(basis) == expected_rank
        assert len(span) == 1 << len(basis)
        assert span == complete_cycle_space(graph_edges)

        reordered = tuple((second, first) for first, second in reversed(graph_edges))
        assert build_fundamental_cycle_basis(vertex_count, reordered) == basis
        exhaustive_graphs += 1

disconnected_edges = (
    (0, 1),
    (0, 2),
    (1, 2),
    (3, 4),
    (3, 5),
    (4, 5),
)
disconnected_basis = build_fundamental_cycle_basis(7, disconnected_edges)

maximum_candidates = tuple(
    (first, second) for first in range(65) for second in range(first + 1, 65)
)
maximum_edges = maximum_candidates[:_MAX_CYCLE_EDGES]
maximum_basis = build_fundamental_cycle_basis(65, maximum_edges)
empty_at_vertex_cap = build_fundamental_cycle_basis(_MAX_CYCLE_VERTICES, ())

invalid_inputs = (
    (3, ((0, 0),), ValueError),
    (3, ((0, 1), (1, 0)), ValueError),
    (3, ((0, 3),), ValueError),
    (3, ((False, 1),), TypeError),
    (3, ((0, 1, 2),), TypeError),
    (65, maximum_candidates[: _MAX_CYCLE_EDGES + 1], ValueError),
)
for invalid_vertex_count, invalid_edges, expected_error in invalid_inputs:
    try:
        build_fundamental_cycle_basis(invalid_vertex_count, invalid_edges)
    except expected_error:
        pass
    else:
        raise AssertionError("accepted a graph outside the closed profile")

assert (
    exhaustive_graphs,
    disconnected_basis,
    len(maximum_basis),
    empty_at_vertex_cap,
) == (
    1_100,
    (
        FundamentalCycle((1, 2), ((0, 1), (0, 2))),
        FundamentalCycle((4, 5), ((3, 4), (3, 5))),
    ),
    1_984,
    (),
)
```

## Trade-offs and Limitations

Normalizing and sorting takes `O(E log E)` time. The direct forest traversal
for every chord takes `O(V * E)` time in the worst case. The implementation
stores `O(V + E + output)` edge or vertex references, and it rejects a complete
basis above 262,144 output edge references.

The basis is canonical only under this exact vertex numbering, normalized-edge
ordering, and lexicographic forest rule. Renumbering vertices can select a
different forest and therefore a different valid basis. Individual tree-path
edges are normalized endpoint pairs and appear in traversal order from the
smaller chord endpoint to the larger one; they are not a vertex walk by
themselves when normalization reverses an edge.

The function accepts only one static simple undirected graph. It rejects loops,
parallel edges, malformed endpoints, and out-of-profile sizes. It does not
handle directed cycles, multigraphs, minimum-weight cycle bases, enumeration of
every simple cycle, dynamic updates, or compact bit-matrix output.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Deterministic Minimum Spanning Forest with Kruskal's Algorithm](build-a-deterministic-minimum-spanning-forest-with-kruskals-algorithm.md)
- [Solve a Bounded Affine GF(2) System with a Canonical Particular Solution and Nullspace Basis](solve-a-bounded-affine-gf-2-system-with-a-canonical-particular-solution-and-nullspace-basis.md)
- [Partition the Edges of a Bounded Simple Undirected Graph into Vertex-Biconnected Blocks](partition-the-edges-of-a-bounded-simple-undirected-graph-into-vertex-biconnected-blocks.md)
<!-- catalog:related:end -->
