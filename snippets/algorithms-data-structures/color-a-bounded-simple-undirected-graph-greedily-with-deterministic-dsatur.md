---
title: "Color a Bounded Simple Undirected Graph Greedily with Deterministic DSATUR"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - two-color-a-bounded-undirected-graph-or-return-an-odd-cycle-witness.md
  - compute-core-numbers-and-a-deterministic-degeneracy-order-for-a-bounded-undirected-graph.md
  - partition-tagged-items-into-minimum-stable-conflict-free-groups.md
---

# Color a Bounded Simple Undirected Graph Greedily with Deterministic DSATUR

## Idea and Problem

Assign reusable integer colors to pairwise-conflicting vertices with a deterministic saturation-first greedy order.

DSATUR next selects the uncolored vertex adjacent to the greatest number of
different colors. This implementation resolves equal saturation by greater
static degree and then smaller vertex index. It assigns the smallest color not
used by a colored neighbor, making the result independent of edge declaration
order and orientation.

The algorithm always returns a proper coloring, but it is a heuristic: its
color count need not equal the graph's chromatic number.

## When to Use

Use this algorithm when a complete, bounded conflict graph is already
available and a reproducible, usually compact grouping matters more than a
proof that the number of groups is minimal. Typical uses include planning
non-conflicting work rounds, test partitions, frequency slots, and transparent
small-graph heuristics.

Use two-coloring when the graph is expected to be bipartite and an odd-cycle
witness is valuable. Choose an exact solver or a maintained graph library when
minimum coloring, precolored vertices, large sparse instances, incremental
updates, or specialized graph classes matter.

## Implementation

```python
from dataclasses import dataclass

_MAX_VERTICES = 4_096
_MAX_EDGES = 32_768


@dataclass(frozen=True, slots=True)
class UndirectedEdge:
    first: int
    second: int


@dataclass(frozen=True, slots=True)
class DsaturColoring:
    colors_by_vertex: tuple[int, ...]
    color_classes: tuple[tuple[int, ...], ...]
    selection_order: tuple[int, ...]


def greedy_dsatur_coloring(
    vertex_count: int,
    edges: tuple[UndirectedEdge, ...],
) -> DsaturColoring:
    """Return one deterministic greedy DSATUR coloring."""
    if type(vertex_count) is not int:
        raise TypeError("vertex_count must be an exact integer")
    if not 0 <= vertex_count <= _MAX_VERTICES:
        raise ValueError("vertex_count is outside the supported range")
    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if len(edges) > _MAX_EDGES:
        raise ValueError("edge count exceeds the supported limit")

    adjacency = [set() for _ in range(vertex_count)]
    seen: set[tuple[int, int]] = set()
    for position, edge in enumerate(edges):
        if type(edge) is not UndirectedEdge:
            raise TypeError(f"edges[{position}] must be an exact UndirectedEdge")
        if type(edge.first) is not int or type(edge.second) is not int:
            raise TypeError(f"edges[{position}] endpoints must be exact integers")
        if not 0 <= edge.first < vertex_count:
            raise ValueError(f"edges[{position}].first is outside the graph")
        if not 0 <= edge.second < vertex_count:
            raise ValueError(f"edges[{position}].second is outside the graph")
        if edge.first == edge.second:
            raise ValueError(f"edges[{position}] is a self-loop")

        pair = tuple(sorted((edge.first, edge.second)))
        if pair in seen:
            raise ValueError(f"edges[{position}] duplicates an undirected pair")
        seen.add(pair)
        first, second = pair
        adjacency[first].add(second)
        adjacency[second].add(first)

    static_degrees = tuple(len(neighbors) for neighbors in adjacency)
    colored_neighbor_colors = [set() for _ in range(vertex_count)]
    colors = [-1] * vertex_count
    selection_order: list[int] = []

    for _ in range(vertex_count):
        vertex = max(
            (index for index, color in enumerate(colors) if color == -1),
            key=lambda index: (
                len(colored_neighbor_colors[index]),
                static_degrees[index],
                -index,
            ),
        )
        forbidden = colored_neighbor_colors[vertex]
        color = 0
        while color in forbidden:
            color += 1
        colors[vertex] = color
        selection_order.append(vertex)

        for neighbor in adjacency[vertex]:
            if colors[neighbor] == -1:
                colored_neighbor_colors[neighbor].add(color)

    color_count = max(colors, default=-1) + 1
    classes = tuple(
        tuple(vertex for vertex, assigned in enumerate(colors) if assigned == color)
        for color in range(color_count)
    )
    return DsaturColoring(
        colors_by_vertex=tuple(colors),
        color_classes=classes,
        selection_order=tuple(selection_order),
    )
```

## Example

```python


odd_cycle = (
    UndirectedEdge(0, 1),
    UndirectedEdge(1, 2),
    UndirectedEdge(2, 3),
    UndirectedEdge(3, 4),
    UndirectedEdge(4, 0),
)

colored = greedy_dsatur_coloring(5, odd_cycle)
reoriented = greedy_dsatur_coloring(
    5,
    tuple(UndirectedEdge(edge.second, edge.first) for edge in reversed(odd_cycle)),
)
empty = greedy_dsatur_coloring(0, ())

assert (
    colored
    == reoriented
    == DsaturColoring(
        colors_by_vertex=(0, 1, 0, 1, 2),
        color_classes=((0, 2), (1, 3), (4,)),
        selection_order=(0, 1, 2, 3, 4),
    )
)
assert empty == DsaturColoring((), (), ())

try:
    greedy_dsatur_coloring(1, (UndirectedEdge(0, 0),))
except ValueError:
    self_loop_rejected = True
else:
    self_loop_rejected = False

assert self_loop_rejected
```

## Trade-offs and Limitations

For `V` vertices and `E` edges, validation and saturation updates take
expected `O(V + E)` work, while repeatedly scanning the uncolored vertices
takes `O(V^2)` time. Adjacency and saturation sets use `O(V + E)` memory. The
fixed limits bound both the quadratic selection cost and retained edge state.

Every edge joins differently colored vertices, assigned colors are contiguous
from zero, and the result is deterministic under edge reordering and endpoint
reversal. The static-degree tie rule is deliberate; a variant using remaining
uncolored degree can make different choices.

DSATUR often finds compact colorings, but this greedy form neither proves nor
guarantees a minimum. It provides no lower bound, optimality certificate,
parallel-edge semantics, dynamic recoloring, weighted objective, capacity
constraint, or mapping from application objects to integer vertices.

## Related Snippets

<!-- catalog:related:start -->
- [Two-Color a Bounded Undirected Graph or Return an Odd-Cycle Witness](two-color-a-bounded-undirected-graph-or-return-an-odd-cycle-witness.md)
- [Compute Core Numbers and a Deterministic Degeneracy Order for a Bounded Undirected Graph](compute-core-numbers-and-a-deterministic-degeneracy-order-for-a-bounded-undirected-graph.md)
- [Partition Tagged Items into Minimum Stable Conflict-Free Groups](partition-tagged-items-into-minimum-stable-conflict-free-groups.md)
<!-- catalog:related:end -->
