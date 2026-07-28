---
title: "Two-Color a Bounded Undirected Graph or Return an Odd-Cycle Witness"
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
  - traverse-a-parent-graph-with-breadth-first-search.md
---

# Two-Color a Bounded Undirected Graph or Return an Odd-Cycle Witness

## Idea and Problem

Color every connected component with two colors, or return a concrete odd cycle that proves such a coloring is impossible.

A deterministic breadth-first traversal assigns opposite colors across each
edge. When it encounters a same-color edge, parent paths join that edge into an
odd cycle. Declaration order controls traversal, and a final normalization
makes the closed witness independent of edge input order and orientation.

## When to Use

Use this algorithm for one fully available, bounded undirected graph with no
parallel endpoint pairs when downstream logic needs either a reproducible
bipartition or evidence that the graph is not bipartite. It is useful for
conflict separation, alternating assignments, and validation that should
explain failure with actual nodes.

Node declaration order must be an acceptable presentation policy. Use a graph
library when inputs are large, updated incrementally, directed, or contain
parallel edges, or when the shortest odd cycle is required.

## Implementation

```python
from collections import deque
from dataclasses import dataclass

_MAX_NODES = 256
_MAX_EDGES = 2_048
_MAX_NODE_CHARACTERS = 128
_MAX_NODE_BYTES = 128
_MAX_TOTAL_NODE_BYTES = 32_768


class GraphTwoColorError(ValueError):
    """Raised when an undirected graph violates the bounded input contract."""


@dataclass(frozen=True, slots=True)
class UndirectedEdge:
    first: str
    second: str


@dataclass(frozen=True, slots=True)
class BipartiteColoring:
    zero: tuple[str, ...]
    one: tuple[str, ...]


@dataclass(frozen=True, slots=True)
class OddCycle:
    nodes: tuple[str, ...]


def _node_byte_length(value: object, *, field: str) -> int:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if not value:
        raise GraphTwoColorError(f"{field} must not be empty")
    if len(value) > _MAX_NODE_CHARACTERS:
        raise GraphTwoColorError(f"{field} has too many characters")
    try:
        encoded = value.encode("utf-8")
    except UnicodeEncodeError:
        raise GraphTwoColorError(f"{field} must be valid UTF-8 text") from None
    if len(encoded) > _MAX_NODE_BYTES:
        raise GraphTwoColorError(f"{field} has too many UTF-8 bytes")
    return len(encoded)


def _validate_graph(
    nodes: object,
    edges: object,
) -> tuple[tuple[str, ...], tuple[tuple[int, int], ...]]:
    if type(nodes) is not tuple:
        raise TypeError("nodes must be an exact tuple")
    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if not 1 <= len(nodes) <= _MAX_NODES:
        raise GraphTwoColorError("node count is outside the supported range")
    if len(edges) > _MAX_EDGES:
        raise GraphTwoColorError("edge count exceeds the supported limit")

    position: dict[str, int] = {}
    total_node_bytes = 0
    for index, node in enumerate(nodes):
        total_node_bytes += _node_byte_length(node, field=f"nodes[{index}]")
        if total_node_bytes > _MAX_TOTAL_NODE_BYTES:
            raise GraphTwoColorError("node names exceed the aggregate byte limit")
        if node in position:
            raise GraphTwoColorError(f"duplicate node at nodes[{index}]")
        position[node] = index

    normalized_edges: list[tuple[int, int]] = []
    seen_pairs: set[tuple[int, int]] = set()
    for index, edge in enumerate(edges):
        if type(edge) is not UndirectedEdge:
            raise TypeError(f"edges[{index}] must be an exact UndirectedEdge")
        if type(edge.first) is not str or type(edge.second) is not str:
            raise TypeError(f"edges[{index}] endpoints must be exact strings")
        if edge.first not in position or edge.second not in position:
            raise GraphTwoColorError(f"edges[{index}] refers to an unknown node")

        first_position = position[edge.first]
        second_position = position[edge.second]
        pair = tuple(sorted((first_position, second_position)))
        if pair in seen_pairs:
            raise GraphTwoColorError(f"edges[{index}] duplicates an undirected endpoint pair")
        seen_pairs.add(pair)
        normalized_edges.append(pair)

    return nodes, tuple(sorted(normalized_edges))


def _canonical_closed_cycle(open_cycle: tuple[int, ...]) -> tuple[int, ...]:
    smallest = min(open_cycle)

    forward_start = open_cycle.index(smallest)
    forward = open_cycle[forward_start:] + open_cycle[:forward_start]

    reversed_cycle = tuple(reversed(open_cycle))
    reverse_start = reversed_cycle.index(smallest)
    backward = reversed_cycle[reverse_start:] + reversed_cycle[:reverse_start]

    chosen = min(forward, backward)
    return (*chosen, chosen[0])


def _cycle_through_conflict(
    first: int,
    second: int,
    parents: list[int],
) -> tuple[int, ...]:
    first_path: list[int] = []
    cursor = first
    while cursor != -1:
        first_path.append(cursor)
        cursor = parents[cursor]
    first_offsets = {node: offset for offset, node in enumerate(first_path)}

    second_path: list[int] = []
    cursor = second
    while cursor not in first_offsets:
        if cursor == -1:
            raise AssertionError("conflict endpoints must share a BFS root")
        second_path.append(cursor)
        cursor = parents[cursor]

    first_to_common = first_path[: first_offsets[cursor] + 1]
    open_cycle = tuple(first_to_common + list(reversed(second_path)))
    return _canonical_closed_cycle(open_cycle)


def two_color_or_odd_cycle(
    nodes: tuple[str, ...],
    edges: tuple[UndirectedEdge, ...],
) -> BipartiteColoring | OddCycle:
    """Return a deterministic coloring or one canonical closed odd cycle."""
    names, normalized_edges = _validate_graph(nodes, edges)

    adjacency: list[list[int]] = [[] for _ in names]
    for first, second in normalized_edges:
        adjacency[first].append(second)
        if first != second:
            adjacency[second].append(first)
    for neighbors in adjacency:
        neighbors.sort()

    colors: list[int | None] = [None] * len(names)
    parents = [-1] * len(names)

    for root in range(len(names)):
        if colors[root] is not None:
            continue
        colors[root] = 0
        queue = deque([root])

        while queue:
            current = queue.popleft()
            for neighbor in adjacency[current]:
                if neighbor == current:
                    return OddCycle((names[current], names[current]))
                if colors[neighbor] is None:
                    colors[neighbor] = 1 - colors[current]
                    parents[neighbor] = current
                    queue.append(neighbor)
                    continue
                if colors[neighbor] == colors[current]:
                    cycle = _cycle_through_conflict(current, neighbor, parents)
                    return OddCycle(tuple(names[position] for position in cycle))

    return BipartiteColoring(
        zero=tuple(name for name, color in zip(names, colors, strict=True) if color == 0),
        one=tuple(name for name, color in zip(names, colors, strict=True) if color == 1),
    )
```

## Example

```python
nodes = ("alpha", "beta", "gamma", "delta", "isolated")
bipartite_edges = (
    UndirectedEdge("beta", "alpha"),
    UndirectedEdge("gamma", "beta"),
    UndirectedEdge("delta", "gamma"),
)
triangle = (
    UndirectedEdge("beta", "gamma"),
    UndirectedEdge("gamma", "alpha"),
    UndirectedEdge("alpha", "beta"),
)
reoriented_triangle = tuple(UndirectedEdge(edge.second, edge.first) for edge in reversed(triangle))

coloring = two_color_or_odd_cycle(nodes, bipartite_edges)
odd_cycle = two_color_or_odd_cycle(nodes, triangle)
reoriented_cycle = two_color_or_odd_cycle(nodes, reoriented_triangle)
self_cycle = two_color_or_odd_cycle(
    nodes,
    (UndirectedEdge("delta", "delta"),),
)

assert (coloring, odd_cycle, reoriented_cycle, self_cycle) == (
    BipartiteColoring(
        zero=("alpha", "gamma", "isolated"),
        one=("beta", "delta"),
    ),
    OddCycle(("alpha", "beta", "gamma", "alpha")),
    OddCycle(("alpha", "beta", "gamma", "alpha")),
    OddCycle(("delta", "delta")),
)
```

## Trade-offs and Limitations

Validation, adjacency construction, and breadth-first traversal use `O(V + E)`
time and memory. Sorting normalized edges and adjacency lists adds
`O(E log E)` worst-case time. The fixed node, edge, and UTF-8 byte limits bound
the complete materialized graph and traversal state.

Each disconnected component starts at its earliest declared node with color
zero, so the two groups are deterministic but not the only valid coloring. The
first same-color conflict under deterministic BFS selects the witness. Its
rotation and direction are canonical, but it is not promised to be the
shortest or globally lexicographically smallest odd cycle. A self-edge is the
valid one-edge witness `(node, node)`.

The function accepts exact node strings and an undirected edge set with no
parallel endpoint pairs; registered self-edges remain valid odd cycles. It does
not normalize Unicode, model directed edges, return every coloring or odd
cycle, process updates, or maintain an incremental index.

## Related Snippets

<!-- catalog:related:start -->
- [Find Articulation Points and Bridges in a Bounded Undirected Graph](find-articulation-points-and-bridges-in-a-bounded-undirected-graph.md)
- [Partition a Bounded Undirected Graph into Deterministic Components with Union-Find](partition-a-bounded-undirected-graph-into-deterministic-components-with-union-find.md)
- [Traverse a Parent Graph with Breadth-First Search](traverse-a-parent-graph-with-breadth-first-search.md)
<!-- catalog:related:end -->
