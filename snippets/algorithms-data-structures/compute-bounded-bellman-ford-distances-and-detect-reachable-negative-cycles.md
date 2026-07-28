---
title: "Compute Bounded Bellman-Ford Distances and Detect Reachable Negative Cycles"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - find-one-deterministic-shortest-path-in-a-bounded-non-negatively-weighted-directed-graph.md
  - partition-a-bounded-directed-graph-into-deterministic-strongly-connected-components.md
  - traverse-a-parent-graph-with-breadth-first-search.md
---

# Compute Bounded Bellman-Ford Distances and Detect Reachable Negative Cycles

## Idea and Problem

Compute exact shortest distances from one source while distinguishing an unreachable node from a graph whose reachable region contains a negative cycle.

Bellman-Ford repeatedly relaxes every directed edge. Each pass in this version
reads one unchanged distance snapshot and applies its best updates only after
the scan, so one pass represents one additional permitted edge. If another
relaxation remains possible after at most `V - 1` passes, a negative cycle is
reachable from the source and finite shortest distances are not returned.

## When to Use

Use this algorithm for a small, fully materialized directed graph with signed
integer weights when distances from one source are required and negative edges
cannot be excluded. It is useful for bounded dependency costs, difference-like
constraints, and deterministic validation where unreachable negative cycles
must not invalidate an independent reachable region.

Choose Dijkstra's algorithm when every weight is non-negative and graph size
makes its better asymptotic performance important. Use a specialized graph
library for large graphs, frequent queries, dynamic updates, cycle witnesses,
or classification of every node affected by a negative cycle.

## Implementation

```python
from dataclasses import dataclass

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_NODES = 256
_MAX_EDGES = 2_048
_MAX_NODE_CHARACTERS = 128
_MAX_NODE_BYTES = 128
_MAX_TOTAL_NODE_BYTES = 32_768


class BellmanFordInputError(ValueError):
    """Raised when a graph violates the bounded input contract."""


@dataclass(frozen=True, slots=True)
class DirectedWeightedEdge:
    source: str
    target: str
    weight: int


@dataclass(frozen=True, slots=True)
class NodeDistance:
    node: str
    distance: int | None


@dataclass(frozen=True, slots=True)
class BellmanFordDistances:
    entries: tuple[NodeDistance, ...]


@dataclass(frozen=True, slots=True)
class ReachableNegativeCycle:
    pass


def _node_byte_length(value: object, *, field: str) -> int:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if not 1 <= len(value) <= _MAX_NODE_CHARACTERS:
        raise BellmanFordInputError(f"{field} has an unsupported character count")
    try:
        encoded = value.encode("utf-8")
    except UnicodeEncodeError:
        raise BellmanFordInputError(f"{field} must be valid UTF-8 text") from None
    if len(encoded) > _MAX_NODE_BYTES:
        raise BellmanFordInputError(f"{field} exceeds the encoded byte limit")
    return len(encoded)


def bellman_ford_distances(
    nodes: tuple[str, ...],
    edges: tuple[DirectedWeightedEdge, ...],
    *,
    source: str,
) -> BellmanFordDistances | ReachableNegativeCycle:
    """Return exact source distances or a reachable-negative-cycle marker."""
    if type(nodes) is not tuple:
        raise TypeError("nodes must be an exact tuple")
    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if not 1 <= len(nodes) <= _MAX_NODES:
        raise BellmanFordInputError("node count is outside the supported range")
    if len(edges) > _MAX_EDGES:
        raise BellmanFordInputError("edge count exceeds the supported limit")

    positions: dict[str, int] = {}
    total_node_bytes = 0
    for node_index, node in enumerate(nodes):
        node_bytes = _node_byte_length(node, field=f"nodes[{node_index}]")
        if node_bytes > _MAX_TOTAL_NODE_BYTES - total_node_bytes:
            raise BellmanFordInputError("node names exceed the aggregate byte limit")
        if node in positions:
            raise BellmanFordInputError(f"duplicate node at nodes[{node_index}]")
        positions[node] = node_index
        total_node_bytes += node_bytes

    _node_byte_length(source, field="source")
    if source not in positions:
        raise BellmanFordInputError("source must be a registered node")

    validated_edges: list[tuple[int, int, int]] = []
    seen_pairs: set[tuple[int, int]] = set()
    for edge_index, edge in enumerate(edges):
        if type(edge) is not DirectedWeightedEdge:
            raise TypeError(f"edges[{edge_index}] must be an exact DirectedWeightedEdge")
        if type(edge.source) is not str or type(edge.target) is not str:
            raise TypeError(f"edges[{edge_index}] endpoints must be exact strings")
        if edge.source not in positions or edge.target not in positions:
            raise BellmanFordInputError(f"edges[{edge_index}] refers to an unknown node")
        if type(edge.weight) is not int:
            raise TypeError(f"edges[{edge_index}].weight must be an exact integer")
        if not _MIN_INT64 <= edge.weight <= _MAX_INT64:
            raise BellmanFordInputError(
                f"edges[{edge_index}].weight is outside the signed 64-bit range"
            )

        pair = (positions[edge.source], positions[edge.target])
        if pair in seen_pairs:
            raise BellmanFordInputError(f"edges[{edge_index}] duplicates a directed endpoint pair")
        seen_pairs.add(pair)
        validated_edges.append((*pair, edge.weight))

    validated_edges.sort()
    distances: list[int | None] = [None] * len(nodes)
    distances[positions[source]] = 0

    for _ in range(len(nodes) - 1):
        updates: dict[int, int] = {}
        for source_position, target_position, weight in validated_edges:
            source_distance = distances[source_position]
            if source_distance is None:
                continue
            candidate = source_distance + weight
            target_distance = updates.get(target_position, distances[target_position])
            if target_distance is None or candidate < target_distance:
                updates[target_position] = candidate

        if not updates:
            break
        for target_position, candidate in updates.items():
            distances[target_position] = candidate

    for source_position, target_position, weight in validated_edges:
        source_distance = distances[source_position]
        target_distance = distances[target_position]
        if source_distance is not None and (
            target_distance is None or source_distance + weight < target_distance
        ):
            return ReachableNegativeCycle()

    return BellmanFordDistances(
        entries=tuple(
            NodeDistance(node=node, distance=distances[node_index])
            for node_index, node in enumerate(nodes)
        )
    )
```

## Example

```python
nodes = ("source", "left", "right", "goal", "isolated-a", "isolated-b")
edges = (
    DirectedWeightedEdge("right", "goal", 3),
    DirectedWeightedEdge("source", "right", 5),
    DirectedWeightedEdge("isolated-a", "isolated-b", -2),
    DirectedWeightedEdge("left", "goal", 10),
    DirectedWeightedEdge("isolated-b", "isolated-a", 1),
    DirectedWeightedEdge("source", "left", 4),
    DirectedWeightedEdge("left", "right", -2),
)
expected = BellmanFordDistances(
    entries=(
        NodeDistance("source", 0),
        NodeDistance("left", 4),
        NodeDistance("right", 2),
        NodeDistance("goal", 5),
        NodeDistance("isolated-a", None),
        NodeDistance("isolated-b", None),
    )
)

distances = bellman_ford_distances(nodes, edges, source="source")
reordered = bellman_ford_distances(nodes, tuple(reversed(edges)), source="source")
reachable_cycle = bellman_ford_distances(
    nodes,
    (*edges, DirectedWeightedEdge("goal", "left", -6)),
    source="source",
)

assert (distances, reordered, reachable_cycle) == (
    expected,
    expected,
    ReachableNegativeCycle(),
)
```

## Trade-offs and Limitations

Validation costs `O(V + E)` time. Relaxation performs at most `V - 1` complete
edge scans and one detection scan, so it costs `O(VE)` time. Positions,
normalized edges, distance state, pending snapshot updates, and the result use
`O(V + E)` memory. Edge weights are signed 64-bit integers, while Python keeps
derived distance arithmetic exact outside that range.

The result follows node declaration order and is independent of edge input
order. `None` means that no directed path from the source reached that node.
A negative self-edge at a reachable node produces the marker; a negative cycle
in an unreachable component does not. Node names are exact UTF-8 strings and
are not Unicode-normalized.

The marker intentionally discards partial distances and carries no cycle
witness or affected-node set. The algorithm does not return paths or
predecessors, classify every negative-cycle descendant, answer all-pairs
queries, accept parallel edges or floats, or maintain a graph across updates.

## Related Snippets

<!-- catalog:related:start -->
- [Find One Deterministic Shortest Path in a Bounded Non-Negatively Weighted Directed Graph](find-one-deterministic-shortest-path-in-a-bounded-non-negatively-weighted-directed-graph.md)
- [Partition a Bounded Directed Graph into Deterministic Strongly Connected Components](partition-a-bounded-directed-graph-into-deterministic-strongly-connected-components.md)
- [Traverse a Parent Graph with Breadth-First Search](traverse-a-parent-graph-with-breadth-first-search.md)
<!-- catalog:related:end -->
