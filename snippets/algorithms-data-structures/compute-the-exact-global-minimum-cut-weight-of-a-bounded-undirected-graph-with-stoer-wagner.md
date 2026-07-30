---
title: "Compute the Exact Global Minimum-Cut Weight of a Bounded Undirected Graph with Stoer-Wagner"
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
  - compute-a-bounded-integer-maximum-flow-and-canonical-minimum-cut-with-edmonds-karp.md
  - build-a-deterministic-minimum-spanning-forest-with-kruskals-algorithm.md
  - partition-a-bounded-undirected-graph-into-deterministic-components-with-union-find.md
---

# Compute the Exact Global Minimum-Cut Weight of a Bounded Undirected Graph with Stoer-Wagner

## Idea and Problem

Compute the exact global minimum-cut weight of a bounded positively weighted undirected graph without choosing source and sink vertices.

Stoer-Wagner runs a sequence of maximum-adjacency phases. Each phase grows a
vertex set by repeatedly adding the outside vertex with the greatest total
weight into the set. The last vertex added defines one cut, after which it is
merged into the preceding vertex. The lightest phase cut is the global
minimum cut.

Only the cut weight is returned. This avoids inventing a public tie rule among
several equally light partitions while keeping the mathematical result
independent of edge order, endpoint orientation, and internal phase ties.

## When to Use

Use this function for a small static undirected graph when the least total
edge weight whose removal disconnects some non-empty vertex subset is needed.
It is useful for exact robustness checks, clustering thresholds, and reference
calculations that must consider every possible source-and-sink separation.

Use a source-and-sink flow algorithm when two particular terminal sets must be
separated or the actual partition is required. Prefer a specialized graph
library for substantially larger or sparse graphs, repeated queries, dynamic
updates, floating-point capacities, or an implementation that must return all
minimum cuts.

## Implementation

```python
from dataclasses import dataclass

_MAX_GLOBAL_MINIMUM_CUT_VERTICES = 128
_MAX_GLOBAL_MINIMUM_CUT_EDGES = 8_128
_MAX_GLOBAL_MINIMUM_CUT_WEIGHT = (1 << 63) - 1


@dataclass(frozen=True, slots=True)
class UndirectedCutEdge:
    first: int
    second: int
    weight: int


def global_minimum_cut_weight(
    vertex_count: int,
    edges: tuple[UndirectedCutEdge, ...],
) -> int:
    """Return the exact global minimum-cut weight of one bounded graph."""
    if type(vertex_count) is not int:
        raise TypeError("vertex_count must be an exact integer")
    if not 2 <= vertex_count <= _MAX_GLOBAL_MINIMUM_CUT_VERTICES:
        raise ValueError("vertex_count is outside the supported range")
    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if len(edges) > _MAX_GLOBAL_MINIMUM_CUT_EDGES:
        raise ValueError("edge count exceeds the supported limit")

    weights = [[0] * vertex_count for _ in range(vertex_count)]
    seen_pairs: set[tuple[int, int]] = set()

    for edge_index, edge in enumerate(edges):
        if type(edge) is not UndirectedCutEdge:
            raise TypeError(f"edges[{edge_index}] must be an exact UndirectedCutEdge")
        if type(edge.first) is not int or type(edge.second) is not int:
            raise TypeError(f"edges[{edge_index}] endpoints must be exact integers")
        if not 0 <= edge.first < vertex_count or not 0 <= edge.second < vertex_count:
            raise ValueError(f"edges[{edge_index}] endpoint is outside the vertex range")
        if edge.first == edge.second:
            raise ValueError(f"edges[{edge_index}] is a self-edge")
        if type(edge.weight) is not int:
            raise TypeError(f"edges[{edge_index}].weight must be an exact integer")
        if not 1 <= edge.weight <= _MAX_GLOBAL_MINIMUM_CUT_WEIGHT:
            raise ValueError(
                f"edges[{edge_index}].weight is outside the positive signed 64-bit range"
            )

        first, second = sorted((edge.first, edge.second))
        pair = (first, second)
        if pair in seen_pairs:
            raise ValueError(f"edges[{edge_index}] duplicates an undirected endpoint pair")
        seen_pairs.add(pair)
        weights[first][second] = edge.weight
        weights[second][first] = edge.weight

    active_vertices = list(range(vertex_count))
    best_weight: int | None = None

    while len(active_vertices) > 1:
        connection_weights = [0] * vertex_count
        added = [False] * vertex_count
        previous: int | None = None

        for phase_index in range(len(active_vertices)):
            current = min(
                (vertex for vertex in active_vertices if not added[vertex]),
                key=lambda vertex: (-connection_weights[vertex], vertex),
            )

            if phase_index == len(active_vertices) - 1:
                phase_weight = connection_weights[current]
                if best_weight is None or phase_weight < best_weight:
                    best_weight = phase_weight
                if best_weight == 0:
                    return 0
                if previous is None:
                    raise AssertionError("a contraction phase must have two vertices")

                for neighbor in active_vertices:
                    if neighbor == previous or neighbor == current:
                        continue
                    weights[previous][neighbor] += weights[current][neighbor]
                    weights[neighbor][previous] = weights[previous][neighbor]
                active_vertices.remove(current)
                break

            added[current] = True
            previous = current
            for neighbor in active_vertices:
                if not added[neighbor]:
                    connection_weights[neighbor] += weights[current][neighbor]

    if best_weight is None:
        raise AssertionError("a graph with two vertices must produce a cut")
    return best_weight
```

## Example

```python
def global_minimum_cut_weight_by_search(
    vertex_count: int,
    edges: tuple[UndirectedCutEdge, ...],
) -> int:
    best_weight: int | None = None
    all_vertices = (1 << vertex_count) - 1

    for remaining_vertices in range(1 << (vertex_count - 1)):
        first_side = 1 | (remaining_vertices << 1)
        if first_side == all_vertices:
            continue
        cut_weight = sum(
            edge.weight
            for edge in edges
            if bool(first_side & (1 << edge.first))
            != bool(first_side & (1 << edge.second))
        )
        if best_weight is None or cut_weight < best_weight:
            best_weight = cut_weight

    if best_weight is None:
        raise AssertionError("a graph with two vertices must have a non-trivial partition")
    return best_weight


def exercise_small_weighted_graphs() -> int:
    from itertools import product

    checked = 0
    for vertex_count in range(2, 5):
        pairs = tuple(
            (first, second)
            for first in range(vertex_count)
            for second in range(first + 1, vertex_count)
        )
        for edge_weights in product((0, 1, 4), repeat=len(pairs)):
            edges = tuple(
                UndirectedCutEdge(first, second, weight)
                for (first, second), weight in zip(pairs, edge_weights, strict=True)
                if weight
            )
            assert global_minimum_cut_weight(
                vertex_count,
                edges,
            ) == global_minimum_cut_weight_by_search(vertex_count, edges)
            checked += 1
    return checked


def exercise_seeded_graphs() -> int:
    from random import Random

    generator = Random(20_260_730)
    checked = 0
    for _ in range(300):
        vertex_count = generator.randint(2, 9)
        edges = tuple(
            UndirectedCutEdge(first, second, generator.randint(1, 100))
            for first in range(vertex_count)
            for second in range(first + 1, vertex_count)
            if generator.random() < 0.55
        )
        expected = global_minimum_cut_weight_by_search(vertex_count, edges)
        reversed_edges = tuple(
            UndirectedCutEdge(edge.second, edge.first, edge.weight)
            for edge in reversed(edges)
        )
        assert global_minimum_cut_weight(vertex_count, edges) == expected
        assert global_minimum_cut_weight(vertex_count, reversed_edges) == expected
        checked += 1
    return checked


def raises(error_type: type[Exception], operation) -> bool:
    try:
        operation()
    except error_type:
        return True
    return False


maximum_complete_graph = tuple(
    UndirectedCutEdge(first, second, _MAX_GLOBAL_MINIMUM_CUT_WEIGHT)
    for first in range(_MAX_GLOBAL_MINIMUM_CUT_VERTICES)
    for second in range(first + 1, _MAX_GLOBAL_MINIMUM_CUT_VERTICES)
)
disconnected_graph = (
    UndirectedCutEdge(0, 1, 7),
    UndirectedCutEdge(2, 3, 11),
)
weighted_tree = (
    UndirectedCutEdge(0, 1, 9),
    UndirectedCutEdge(1, 2, 4),
    UndirectedCutEdge(1, 3, 12),
    UndirectedCutEdge(3, 4, 5),
)

invalid_calls = (
    lambda: global_minimum_cut_weight(True, ()),
    lambda: global_minimum_cut_weight(1, ()),
    lambda: global_minimum_cut_weight(_MAX_GLOBAL_MINIMUM_CUT_VERTICES + 1, ()),
    lambda: global_minimum_cut_weight(2, []),
    lambda: global_minimum_cut_weight(2, ((0, 1, 1),)),
    lambda: global_minimum_cut_weight(2, (UndirectedCutEdge(True, 1, 1),)),
    lambda: global_minimum_cut_weight(2, (UndirectedCutEdge(0, 2, 1),)),
    lambda: global_minimum_cut_weight(2, (UndirectedCutEdge(0, 0, 1),)),
    lambda: global_minimum_cut_weight(2, (UndirectedCutEdge(0, 1, True),)),
    lambda: global_minimum_cut_weight(2, (UndirectedCutEdge(0, 1, 0),)),
    lambda: global_minimum_cut_weight(
        2,
        (UndirectedCutEdge(0, 1, 1), UndirectedCutEdge(1, 0, 2)),
    ),
)

assert (
    exercise_small_weighted_graphs(),
    exercise_seeded_graphs(),
    global_minimum_cut_weight(4, disconnected_graph),
    global_minimum_cut_weight(5, weighted_tree),
    global_minimum_cut_weight(
        _MAX_GLOBAL_MINIMUM_CUT_VERTICES,
        maximum_complete_graph,
    ),
    len(maximum_complete_graph),
    sum(raises((TypeError, ValueError), call) for call in invalid_calls),
) == (
    759,
    300,
    0,
    4,
    (_MAX_GLOBAL_MINIMUM_CUT_VERTICES - 1) * _MAX_GLOBAL_MINIMUM_CUT_WEIGHT,
    _MAX_GLOBAL_MINIMUM_CUT_EDGES,
    11,
)
```

## Trade-offs and Limitations

Validation and matrix construction take `O(V^2 + E)` time because the dense
matrix initializes every vertex pair. Stoer-Wagner takes `O(V^3)` exact
integer operations in this direct implementation. The matrix, phase arrays,
active vertex list, and duplicate-pair set retain `O(V^2 + E)` Python
objects. Addition and comparison costs grow with accumulated edge-weight bit
lengths rather than remaining constant.

Vertices are the consecutive indexes from zero through `vertex_count - 1`.
Every declared edge has a positive signed-64-bit weight, but contracted weights
and the returned result remain exact Python integers even when their sum
exceeds that range. Missing edges have zero weight, so a disconnected graph
has global minimum-cut weight zero.

Repeated endpoint pairs are rejected even when their orientation is reversed.
The internal maximum-adjacency phase resolves equal connection weights by the
lower vertex index, but that choice is not observable in the returned global
weight. The function does not return either side of a cut, enumerate tied
cuts, accept parallel edges, support directed or negative weights, mutate the
graph, or make cryptographic or constant-time guarantees.

## Related Snippets

<!-- catalog:related:start -->
- [Compute a Bounded Integer Maximum Flow and Canonical Minimum Cut with Edmonds-Karp](compute-a-bounded-integer-maximum-flow-and-canonical-minimum-cut-with-edmonds-karp.md)
- [Build a Deterministic Minimum Spanning Forest with Kruskal's Algorithm](build-a-deterministic-minimum-spanning-forest-with-kruskals-algorithm.md)
- [Partition a Bounded Undirected Graph into Deterministic Components with Union-Find](partition-a-bounded-undirected-graph-into-deterministic-components-with-union-find.md)
<!-- catalog:related:end -->
