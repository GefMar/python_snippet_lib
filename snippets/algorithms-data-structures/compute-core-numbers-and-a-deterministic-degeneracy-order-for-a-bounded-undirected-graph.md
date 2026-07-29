---
title: "Compute Core Numbers and a Deterministic Degeneracy Order for a Bounded Undirected Graph"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - find-articulation-points-and-bridges-in-a-bounded-undirected-graph.md
  - partition-a-bounded-undirected-graph-into-deterministic-components-with-union-find.md
  - two-color-a-bounded-undirected-graph-or-return-an-odd-cycle-witness.md
---

# Compute Core Numbers and a Deterministic Degeneracy Order for a Bounded Undirected Graph

## Idea and Problem

Measure how deeply every vertex belongs to a graph's dense cores and return one reproducible minimum-degree peeling order.

The `k`-core is the maximal induced subgraph in which every remaining vertex
has degree at least `k`. A vertex's core number is the greatest `k` whose core
contains it, and the graph's degeneracy is the largest core number.

Repeatedly removing a current minimum-degree vertex produces all three results.
A heap selects the smallest vertex identifier when residual degrees tie.
Because later removals can have raw degrees below an earlier maximum, each
removed vertex receives the running maximum removal degree rather than its raw
degree.

## When to Use

Use this decomposition for one complete, bounded, simple undirected graph when
core membership, degeneracy-aware processing, or a stable sparse-graph
elimination order is useful. Vertices are the closed integer range from zero
through `vertex_count - 1`, including isolated vertices.

It fits graph exploration, dense-region filtering, ordering before greedy
algorithms, and structural validation. Use a specialized graph library for
dynamic updates, parallel edges, weighted cores, directed variants, very large
graphs, or community models whose density semantics differ from `k`-cores.

## Implementation

```python
from dataclasses import dataclass
from heapq import heapify, heappop, heappush

_MAX_CORE_VERTICES = 50_000
_MAX_CORE_EDGES = 100_000


@dataclass(frozen=True, slots=True)
class CoreDecomposition:
    core_numbers: tuple[int, ...]
    removal_order: tuple[int, ...]
    degeneracy: int


def compute_core_decomposition(
    vertex_count: int,
    edges: tuple[tuple[int, int], ...],
) -> CoreDecomposition:
    """Return core numbers, deterministic peeling order, and degeneracy."""
    if type(vertex_count) is not int:
        raise TypeError("vertex_count must be an exact integer")
    if not 1 <= vertex_count <= _MAX_CORE_VERTICES:
        raise ValueError("vertex_count is outside the supported range")
    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if len(edges) > _MAX_CORE_EDGES:
        raise ValueError("edge count exceeds the supported limit")

    normalized_edges: list[tuple[int, int]] = []
    seen_edges: set[tuple[int, int]] = set()
    for edge_index, edge in enumerate(edges):
        if type(edge) is not tuple:
            raise TypeError(f"edges[{edge_index}] must be an exact tuple")
        if len(edge) != 2:
            raise ValueError(f"edges[{edge_index}] must contain two endpoints")

        first, second = edge
        if type(first) is not int:
            raise TypeError(f"edges[{edge_index}].first must be an exact integer")
        if type(second) is not int:
            raise TypeError(f"edges[{edge_index}].second must be an exact integer")
        if not 0 <= first < vertex_count:
            raise ValueError(f"edges[{edge_index}].first is outside the graph")
        if not 0 <= second < vertex_count:
            raise ValueError(f"edges[{edge_index}].second is outside the graph")
        if first == second:
            raise ValueError(f"edges[{edge_index}] is a self-edge")

        normalized = (first, second) if first < second else (second, first)
        if normalized in seen_edges:
            raise ValueError(f"edges[{edge_index}] duplicates an undirected edge")
        seen_edges.add(normalized)
        normalized_edges.append(normalized)

    adjacency: list[list[int]] = [[] for _ in range(vertex_count)]
    for first, second in sorted(normalized_edges):
        adjacency[first].append(second)
        adjacency[second].append(first)

    residual_degrees = [len(neighbors) for neighbors in adjacency]
    pending = [(degree, vertex) for vertex, degree in enumerate(residual_degrees)]
    heapify(pending)
    removed = [False] * vertex_count
    core_numbers = [0] * vertex_count
    removal_order: list[int] = []
    running_core = 0

    while pending:
        degree, vertex = heappop(pending)
        if removed[vertex] or degree != residual_degrees[vertex]:
            continue

        removed[vertex] = True
        running_core = max(running_core, degree)
        core_numbers[vertex] = running_core
        removal_order.append(vertex)

        for neighbor in adjacency[vertex]:
            if removed[neighbor]:
                continue
            residual_degrees[neighbor] -= 1
            heappush(pending, (residual_degrees[neighbor], neighbor))

    return CoreDecomposition(
        core_numbers=tuple(core_numbers),
        removal_order=tuple(removal_order),
        degeneracy=running_core,
    )
```

## Example

```python
def adjacency_sets(
    vertex_count: int,
    edges: tuple[tuple[int, int], ...],
) -> tuple[set[int], ...]:
    adjacency = tuple(set() for _ in range(vertex_count))
    for first, second in edges:
        adjacency[first].add(second)
        adjacency[second].add(first)
    return adjacency


def core_numbers_by_fixed_point(
    vertex_count: int,
    edges: tuple[tuple[int, int], ...],
) -> tuple[int, ...]:
    adjacency = adjacency_sets(vertex_count, edges)
    core_numbers = [0] * vertex_count

    for level in range(1, vertex_count):
        active = set(range(vertex_count))
        while True:
            removed = {
                vertex
                for vertex in active
                if sum(neighbor in active for neighbor in adjacency[vertex]) < level
            }
            if not removed:
                break
            active.difference_update(removed)
        for vertex in active:
            core_numbers[vertex] = level
    return tuple(core_numbers)


def removal_by_recomputed_degrees(
    vertex_count: int,
    edges: tuple[tuple[int, int], ...],
) -> tuple[tuple[int, ...], int]:
    adjacency = adjacency_sets(vertex_count, edges)
    active = set(range(vertex_count))
    order: list[int] = []
    degeneracy = 0

    while active:
        residual_degrees = {
            vertex: sum(neighbor in active for neighbor in adjacency[vertex]) for vertex in active
        }
        minimum_degree = min(residual_degrees.values())
        vertex = min(
            candidate for candidate, degree in residual_degrees.items() if degree == minimum_degree
        )
        degeneracy = max(degeneracy, minimum_degree)
        order.append(vertex)
        active.remove(vertex)
    return tuple(order), degeneracy


def exercise_all_small_graphs() -> int:
    from itertools import combinations

    checked = 0
    for vertex_count in range(1, 7):
        possible_edges = tuple(combinations(range(vertex_count), 2))
        for edge_mask in range(1 << len(possible_edges)):
            edges = tuple(
                edge
                for edge_index, edge in enumerate(possible_edges)
                if edge_mask & (1 << edge_index)
            )
            expected_order, expected_degeneracy = removal_by_recomputed_degrees(
                vertex_count,
                edges,
            )
            actual = compute_core_decomposition(vertex_count, edges)
            assert actual.core_numbers == core_numbers_by_fixed_point(
                vertex_count,
                edges,
            )
            assert actual.removal_order == expected_order
            assert actual.degeneracy == expected_degeneracy

            reoriented = tuple((second, first) for first, second in reversed(edges))
            assert compute_core_decomposition(vertex_count, reoriented) == actual
            checked += 1
    return checked


ordinary = compute_core_decomposition(
    5,
    ((0, 1), (1, 2), (2, 0), (2, 3), (3, 4)),
)

maximum_edges = tuple(
    (vertex, (vertex + 1) % _MAX_CORE_VERTICES) for vertex in range(_MAX_CORE_VERTICES)
) + tuple((vertex, (vertex + 2) % _MAX_CORE_VERTICES) for vertex in range(_MAX_CORE_VERTICES))
maximum_decomposition = compute_core_decomposition(
    _MAX_CORE_VERTICES,
    maximum_edges,
)

value_errors = 0
for vertex_count, invalid_edges in (
    (0, ()),
    (_MAX_CORE_VERTICES + 1, ()),
    (2, ((0, 1),) * (_MAX_CORE_EDGES + 1)),
    (2, ((0, 0),)),
    (2, ((0, 2),)),
    (2, ((0, 1), (1, 0))),
    (2, ((0, 1, 1),)),
):
    try:
        compute_core_decomposition(vertex_count, invalid_edges)
    except ValueError:
        value_errors += 1

type_errors = 0
for vertex_count, invalid_edges in (
    (True, ()),
    (1, []),
    (2, ([0, 1],)),
    (2, ((False, 1),)),
):
    try:
        compute_core_decomposition(vertex_count, invalid_edges)
    except TypeError:
        type_errors += 1

assert (
    exercise_all_small_graphs(),
    ordinary,
    len(maximum_edges),
    len(maximum_decomposition.removal_order),
    len(set(maximum_decomposition.removal_order)),
    set(maximum_decomposition.core_numbers),
    maximum_decomposition.degeneracy,
    value_errors,
    type_errors,
) == (
    33_867,
    CoreDecomposition(
        core_numbers=(2, 2, 2, 1, 1),
        removal_order=(4, 3, 0, 1, 2),
        degeneracy=2,
    ),
    100_000,
    50_000,
    50_000,
    {4},
    4,
    7,
    4,
)
```

## Trade-offs and Limitations

Validation and adjacency construction use `O(n + m)` work. Each vertex enters
the heap initially, and each edge causes at most one residual-degree update
and one additional heap entry. The complete algorithm therefore uses
`O((n + m) log n)` time and `O(n + m)` memory for a simple graph.

The core-number tuple is mathematically unique. The removal permutation is one
deterministic degeneracy order: the heap always chooses the smallest vertex
identifier among vertices with the current minimum residual degree. Other
valid peeling orders can differ. Running-maximum assignment is essential
because a vertex removed after part of one core has been peeled can have a raw
residual degree below its true core number.

The implementation accepts disconnected graphs and isolated vertices but
rejects self-edges, duplicate undirected edges, and parallel-edge semantics. It
does not enumerate each `k`-core separately, maintain cores after updates,
measure weighted or directed density, find communities, or promise that the
deterministic removal order optimizes any objective beyond minimum-degree
peeling.

## Related Snippets

<!-- catalog:related:start -->
- [Find Articulation Points and Bridges in a Bounded Undirected Graph](find-articulation-points-and-bridges-in-a-bounded-undirected-graph.md)
- [Partition a Bounded Undirected Graph into Deterministic Components with Union-Find](partition-a-bounded-undirected-graph-into-deterministic-components-with-union-find.md)
- [Two-Color a Bounded Undirected Graph or Return an Odd-Cycle Witness](two-color-a-bounded-undirected-graph-or-return-an-odd-cycle-witness.md)
<!-- catalog:related:end -->
