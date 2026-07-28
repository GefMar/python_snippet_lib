---
title: "Find Articulation Points and Bridges in a Bounded Undirected Graph"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - two-color-a-bounded-undirected-graph-or-return-an-odd-cycle-witness.md
  - partition-a-bounded-undirected-graph-into-deterministic-components-with-union-find.md
  - build-a-deterministic-minimum-spanning-forest-with-kruskals-algorithm.md
---

# Find Articulation Points and Bridges in a Bounded Undirected Graph

## Idea and Problem

Find the vertices and edges whose removal would increase the number of connected components in a static undirected graph.

An iterative depth-first traversal records when each node is discovered and
the earliest discovery reachable from its DFS subtree. Those low-link values
identify cut vertices and bridges without depending on the interpreter's
recursion limit. Declaration order controls traversal and public output.

## When to Use

Use this algorithm for a fully available, bounded simple undirected graph when
single points of structural disconnection matter. It fits topology analysis,
validation, and planning that needs to distinguish redundant cycles from
vertices or links whose removal separates existing connectivity.

Use a dynamic connectivity structure when edges change frequently, and use a
specialized graph library when parallel edges, directed cuts, biconnected
components, or block-cut trees are part of the required result.

## Implementation

```python
from dataclasses import dataclass

_MAX_NODES = 256
_MAX_EDGES = 2_048
_MAX_NODE_CHARACTERS = 128
_MAX_NODE_BYTES = 128
_MAX_TOTAL_NODE_BYTES = 32_768


class GraphCutsError(ValueError):
    """Raised when an undirected graph violates the bounded input contract."""


@dataclass(frozen=True, slots=True)
class UndirectedEdge:
    first: str
    second: str


@dataclass(frozen=True, slots=True)
class GraphCuts:
    articulation_points: tuple[str, ...]
    bridges: tuple[UndirectedEdge, ...]


def _node_byte_length(value: object, *, field: str) -> int:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if not value:
        raise GraphCutsError(f"{field} must not be empty")
    if len(value) > _MAX_NODE_CHARACTERS:
        raise GraphCutsError(f"{field} has too many characters")
    try:
        encoded = value.encode("utf-8")
    except UnicodeEncodeError:
        raise GraphCutsError(f"{field} must be valid UTF-8 text") from None
    if len(encoded) > _MAX_NODE_BYTES:
        raise GraphCutsError(f"{field} has too many UTF-8 bytes")
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
        raise GraphCutsError("node count is outside the supported range")
    if len(edges) > _MAX_EDGES:
        raise GraphCutsError("edge count exceeds the supported limit")

    position: dict[str, int] = {}
    total_node_bytes = 0
    for index, node in enumerate(nodes):
        total_node_bytes += _node_byte_length(node, field=f"nodes[{index}]")
        if total_node_bytes > _MAX_TOTAL_NODE_BYTES:
            raise GraphCutsError("node names exceed the aggregate byte limit")
        if node in position:
            raise GraphCutsError(f"duplicate node at nodes[{index}]")
        position[node] = index

    normalized_edges: list[tuple[int, int]] = []
    seen_pairs: set[tuple[int, int]] = set()
    for index, edge in enumerate(edges):
        if type(edge) is not UndirectedEdge:
            raise TypeError(f"edges[{index}] must be an exact UndirectedEdge")
        if type(edge.first) is not str or type(edge.second) is not str:
            raise TypeError(f"edges[{index}] endpoints must be exact strings")
        if edge.first not in position or edge.second not in position:
            raise GraphCutsError(f"edges[{index}] refers to an unknown node")

        first_position = position[edge.first]
        second_position = position[edge.second]
        if first_position == second_position:
            raise GraphCutsError(f"edges[{index}] is a self-edge")
        pair = tuple(sorted((first_position, second_position)))
        if pair in seen_pairs:
            raise GraphCutsError(f"edges[{index}] duplicates an undirected endpoint pair")
        seen_pairs.add(pair)
        normalized_edges.append(pair)

    return nodes, tuple(sorted(normalized_edges))


def find_graph_cuts(
    nodes: tuple[str, ...],
    edges: tuple[UndirectedEdge, ...],
) -> GraphCuts:
    """Return deterministic articulation points and bridges without recursion."""
    names, normalized_edges = _validate_graph(nodes, edges)

    adjacency: list[list[int]] = [[] for _ in names]
    for first, second in normalized_edges:
        adjacency[first].append(second)
        adjacency[second].append(first)
    for neighbors in adjacency:
        neighbors.sort()

    discovery = [-1] * len(names)
    low = [-1] * len(names)
    parents = [-1] * len(names)
    child_counts = [0] * len(names)
    articulation = [False] * len(names)
    bridge_positions: list[tuple[int, int]] = []
    next_discovery = 0

    for root in range(len(names)):
        if discovery[root] != -1:
            continue
        discovery[root] = next_discovery
        low[root] = next_discovery
        next_discovery += 1
        stack: list[tuple[int, int]] = [(root, 0)]

        while stack:
            current, neighbor_offset = stack[-1]
            if neighbor_offset < len(adjacency[current]):
                neighbor = adjacency[current][neighbor_offset]
                stack[-1] = (current, neighbor_offset + 1)

                if discovery[neighbor] == -1:
                    parents[neighbor] = current
                    child_counts[current] += 1
                    discovery[neighbor] = next_discovery
                    low[neighbor] = next_discovery
                    next_discovery += 1
                    stack.append((neighbor, 0))
                elif neighbor != parents[current]:
                    low[current] = min(low[current], discovery[neighbor])
                continue

            stack.pop()
            parent = parents[current]
            if parent == -1:
                if child_counts[current] > 1:
                    articulation[current] = True
                continue

            low[parent] = min(low[parent], low[current])
            if low[current] > discovery[parent]:
                bridge_positions.append(tuple(sorted((parent, current))))
            if parents[parent] != -1 and low[current] >= discovery[parent]:
                articulation[parent] = True

    return GraphCuts(
        articulation_points=tuple(
            name for name, is_cut in zip(names, articulation, strict=True) if is_cut
        ),
        bridges=tuple(
            UndirectedEdge(names[first], names[second])
            for first, second in sorted(bridge_positions)
        ),
    )
```

## Example

```python
nodes = (
    "alpha",
    "beta",
    "gamma",
    "delta",
    "epsilon",
    "zeta",
    "leaf",
    "isolated",
    "hub",
    "spoke-a",
    "spoke-b",
)
edges = (
    UndirectedEdge("beta", "gamma"),
    UndirectedEdge("gamma", "alpha"),
    UndirectedEdge("alpha", "beta"),
    UndirectedEdge("gamma", "delta"),
    UndirectedEdge("delta", "epsilon"),
    UndirectedEdge("epsilon", "zeta"),
    UndirectedEdge("zeta", "delta"),
    UndirectedEdge("delta", "leaf"),
    UndirectedEdge("hub", "spoke-a"),
    UndirectedEdge("spoke-b", "hub"),
)
reoriented = tuple(UndirectedEdge(edge.second, edge.first) for edge in reversed(edges))

expected = GraphCuts(
    articulation_points=("gamma", "delta", "hub"),
    bridges=(
        UndirectedEdge("gamma", "delta"),
        UndirectedEdge("delta", "leaf"),
        UndirectedEdge("hub", "spoke-a"),
        UndirectedEdge("hub", "spoke-b"),
    ),
)

try:
    find_graph_cuts(nodes, (UndirectedEdge("alpha", "alpha"),))
except GraphCutsError:
    self_edge_rejected = True
else:
    self_edge_rejected = False

assert (find_graph_cuts(nodes, edges), find_graph_cuts(nodes, reoriented)) == (
    expected,
    expected,
)
assert self_edge_rejected
```

## Trade-offs and Limitations

Validation, adjacency construction, and low-link traversal use `O(V + E)`
time and memory. Sorting normalized edges, adjacency lists, and returned
bridges adds `O(E log E)` worst-case time. Explicit DFS frames make the
256-node depth bound independent of the interpreter's recursion limit.

For a DFS tree child, `low[child] > discovery[parent]` identifies a bridge.
The non-root articulation rule is `low[child] >= discovery[parent]`; a DFS root
is an articulation point only when it has more than one tree child.
Disconnected and isolated components are traversed independently and do not
require special output entries.

The function models a static simple undirected graph. It rejects self-edges,
parallel endpoint pairs, unknown nodes, directed or weighted semantics, and
does not return biconnected components, a block-cut tree, paths, dynamic
updates, or an incremental connectivity index.

## Related Snippets

<!-- catalog:related:start -->
- [Two-Color a Bounded Undirected Graph or Return an Odd-Cycle Witness](two-color-a-bounded-undirected-graph-or-return-an-odd-cycle-witness.md)
- [Partition a Bounded Undirected Graph into Deterministic Components with Union-Find](partition-a-bounded-undirected-graph-into-deterministic-components-with-union-find.md)
- [Build a Deterministic Minimum Spanning Forest with Kruskal's Algorithm](build-a-deterministic-minimum-spanning-forest-with-kruskals-algorithm.md)
<!-- catalog:related:end -->
