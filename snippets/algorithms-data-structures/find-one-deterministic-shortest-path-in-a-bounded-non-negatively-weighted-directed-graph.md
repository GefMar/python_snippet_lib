---
title: "Find One Deterministic Shortest Path in a Bounded Non-Negatively Weighted Directed Graph"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - partition-a-bounded-directed-graph-into-deterministic-strongly-connected-components.md
  - traverse-a-parent-graph-with-breadth-first-search.md
  - resolve-stable-ordering-constraints-with-topological-sort.md
---

# Find One Deterministic Shortest Path in a Bounded Non-Negatively Weighted Directed Graph

## Idea and Problem

Run Dijkstra's algorithm on a completely validated graph while making equal-distance path selection independent of edge input order.

Node declaration position defines every operational tie. Adjacency lists are
sorted by target position, and the heap orders equal-distance frontier nodes by
their declared position. A strictly shorter candidate replaces the current
predecessor; an equal-distance candidate does not replace the predecessor that
first established that distance.

## When to Use

Use this for one shortest-path query over a small, fully materialized directed
graph whose edge weights are non-negative integers. It fits configuration
planning, bounded dependency routing, and deterministic tests where
reproducible output matters when several paths have the same total weight.

The declaration order must be a meaningful and stable policy input. Use a
different algorithm for negative weights, and use a specialized graph library
when graphs are large, updated frequently, or queried from many sources.

## Implementation

```python
from dataclasses import dataclass
from heapq import heappop, heappush
from re import fullmatch

_MAX_NODES = 256
_MAX_EDGES = 2_048
_MAX_EDGE_WEIGHT = (1 << 63) - 1
_NODE_PATTERN = r"[A-Za-z0-9][A-Za-z0-9._:-]{0,63}"


class ShortestPathError(ValueError):
    """Raised when a weighted graph violates the bounded input contract."""


@dataclass(frozen=True, slots=True)
class DirectedWeightedEdge:
    source: str
    target: str
    weight: int


@dataclass(frozen=True, slots=True)
class ShortestPath:
    total_weight: int
    nodes: tuple[str, ...]


@dataclass(frozen=True, slots=True)
class UnreachablePath:
    pass


def _validated_node_name(value: object, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact str")
    if fullmatch(_NODE_PATTERN, value, flags=0) is None:
        raise ShortestPathError(f"{field} is not a valid node name")
    return value


def find_deterministic_shortest_path(
    nodes: tuple[str, ...],
    edges: tuple[DirectedWeightedEdge, ...],
    *,
    start: str,
    goal: str,
) -> ShortestPath | UnreachablePath:
    """Return one declaration-order-deterministic minimum-weight path."""
    if type(nodes) is not tuple:
        raise TypeError("nodes must be an exact tuple")
    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if not 1 <= len(nodes) <= _MAX_NODES:
        raise ShortestPathError("node count is outside the supported range")
    if len(edges) > _MAX_EDGES:
        raise ShortestPathError(f"edges must contain at most {_MAX_EDGES} items")

    position: dict[str, int] = {}
    for node_index, raw_name in enumerate(nodes):
        name = _validated_node_name(raw_name, f"nodes[{node_index}]")
        if name in position:
            raise ShortestPathError(f"duplicate node: {name!r}")
        position[name] = node_index

    start_name = _validated_node_name(start, "start")
    goal_name = _validated_node_name(goal, "goal")
    if start_name not in position or goal_name not in position:
        raise ShortestPathError("start and goal must belong to nodes")

    validated_edges: list[tuple[int, int, int]] = []
    seen_pairs: set[tuple[int, int]] = set()
    for edge_index, edge in enumerate(edges):
        if type(edge) is not DirectedWeightedEdge:
            raise TypeError(f"edges[{edge_index}] must be an exact DirectedWeightedEdge")
        source = _validated_node_name(edge.source, f"edges[{edge_index}].source")
        target = _validated_node_name(edge.target, f"edges[{edge_index}].target")
        if source not in position or target not in position:
            raise ShortestPathError(f"edges[{edge_index}] refers to an unknown node")
        if type(edge.weight) is not int:
            raise TypeError(f"edges[{edge_index}].weight must be an exact integer")
        if not 0 <= edge.weight <= _MAX_EDGE_WEIGHT:
            raise ShortestPathError(f"edges[{edge_index}].weight is outside the range")

        pair = (position[source], position[target])
        if pair in seen_pairs:
            raise ShortestPathError(f"duplicate directed edge at edges[{edge_index}]")
        seen_pairs.add(pair)
        validated_edges.append((*pair, edge.weight))

    adjacency: list[list[tuple[int, int]]] = [[] for _ in nodes]
    for source_index, target_index, weight in sorted(validated_edges):
        adjacency[source_index].append((target_index, weight))

    start_index = position[start_name]
    goal_index = position[goal_name]
    distances: list[int | None] = [None] * len(nodes)
    predecessors: list[int | None] = [None] * len(nodes)
    settled = [False] * len(nodes)
    distances[start_index] = 0
    frontier = [(0, start_index)]

    while frontier:
        distance, node_index = heappop(frontier)
        if settled[node_index] or distances[node_index] != distance:
            continue
        settled[node_index] = True
        if node_index == goal_index:
            break

        for target_index, weight in adjacency[node_index]:
            candidate = distance + weight
            known = distances[target_index]
            if known is None or candidate < known:
                distances[target_index] = candidate
                predecessors[target_index] = node_index
                heappush(frontier, (candidate, target_index))

    total_weight = distances[goal_index]
    if total_weight is None:
        return UnreachablePath()

    reversed_path: list[str] = []
    cursor: int | None = goal_index
    while cursor is not None:
        reversed_path.append(nodes[cursor])
        cursor = predecessors[cursor]
    return ShortestPath(total_weight, tuple(reversed(reversed_path)))
```

## Example

```python
nodes = ("start", "left", "right", "middle", "goal", "isolated")
edges = (
    DirectedWeightedEdge("start", "right", 1),
    DirectedWeightedEdge("middle", "left", 0),
    DirectedWeightedEdge("left", "middle", 0),
    DirectedWeightedEdge("middle", "goal", 1),
    DirectedWeightedEdge("start", "left", 1),
    DirectedWeightedEdge("right", "goal", 1),
)
expected = ShortestPath(2, ("start", "right", "goal"))

forward = find_deterministic_shortest_path(nodes, edges, start="start", goal="goal")
reordered = find_deterministic_shortest_path(
    nodes,
    tuple(reversed(edges)),
    start="start",
    goal="goal",
)
unreachable = find_deterministic_shortest_path(
    nodes,
    edges,
    start="start",
    goal="isolated",
)

assert (forward, reordered, unreachable) == (
    expected,
    expected,
    UnreachablePath(),
)
```

## Trade-offs and Limitations

Validation and adjacency construction inspect `V` nodes and `E` edges. Sorting
the normalized edges costs `O(E log E)` in the worst case; the heap traversal
costs `O((V + E) log V)`. Graph state and the heap use `O(V + E)` memory. Python
integer arithmetic keeps path totals exact beyond the per-edge signed 64-bit
bound.

The result is deterministic under edge permutation, but it is not promised to
be the lexicographically smallest path or the path with the fewest edges. In
the example, declaration-index heap ties let `right` establish the goal before
the equal-cost route through `left` and `middle`. Zero-weight edges and cycles
are supported; negative weights, parallel directed edges, float weights,
all-pairs queries, dynamic updates, and persistent indexes are not.

## Related Snippets

<!-- catalog:related:start -->
- [Partition a Bounded Directed Graph into Deterministic Strongly Connected Components](partition-a-bounded-directed-graph-into-deterministic-strongly-connected-components.md)
- [Traverse a Parent Graph with Breadth-First Search](traverse-a-parent-graph-with-breadth-first-search.md)
- [Resolve Stable Ordering Constraints with Topological Sort](resolve-stable-ordering-constraints-with-topological-sort.md)
<!-- catalog:related:end -->
