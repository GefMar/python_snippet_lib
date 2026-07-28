---
title: "Compute a Bounded Integer Maximum Flow and Canonical Minimum Cut with Edmonds-Karp"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-bounded-bellman-ford-distances-and-detect-reachable-negative-cycles.md
  - find-one-deterministic-shortest-path-in-a-bounded-non-negatively-weighted-directed-graph.md
  - build-a-deterministic-minimum-spanning-forest-with-kruskals-algorithm.md
---

# Compute a Bounded Integer Maximum Flow and Canonical Minimum Cut with Edmonds-Karp

## Idea and Problem

Compute an exact maximum flow and the inclusion-minimal source side shared by every minimum cut in a bounded directed network.

Edmonds-Karp repeatedly uses breadth-first search to find a shortest residual
augmenting path. A matrix stores one residual capacity for each ordered node
pair, so antiparallel original edges and cancellation capacity combine as net
flow without requiring a per-edge flow representation.

## When to Use

Use this algorithm for a small, fully materialized integer-capacity network
when the maximum transferable amount and one canonical minimum-cut partition
are sufficient. It is useful for bounded allocation, reachability under
capacity constraints, and deterministic checks where declaration order should
control traversal but not the mathematical result.

Use a specialized flow library for large or dense networks. Choose another
algorithm when edges have costs or lower bounds, several sources or sinks must
be modeled directly, per-edge flows are required, or the network changes
between queries.

## Implementation

```python
from collections import deque
from dataclasses import dataclass

_MAX_INT64 = (1 << 63) - 1
_MAX_NODES = 64
_MAX_EDGES = 512
_MAX_NODE_CHARACTERS = 128
_MAX_NODE_BYTES = 128
_MAX_TOTAL_NODE_BYTES = 8_192


class MaximumFlowInputError(ValueError):
    """Raised when a network violates the bounded flow contract."""


@dataclass(frozen=True, slots=True)
class DirectedCapacityEdge:
    source: str
    target: str
    capacity: int


@dataclass(frozen=True, slots=True)
class MaximumFlowCut:
    value: int
    source_side: tuple[str, ...]
    sink_side: tuple[str, ...]


def _node_byte_length(value: object, *, field: str) -> int:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if not value:
        raise MaximumFlowInputError(f"{field} must not be empty")
    if len(value) > _MAX_NODE_CHARACTERS:
        raise MaximumFlowInputError(f"{field} has too many characters")
    try:
        encoded = value.encode("utf-8")
    except UnicodeEncodeError:
        raise MaximumFlowInputError(f"{field} must be valid UTF-8 text") from None
    if len(encoded) > _MAX_NODE_BYTES:
        raise MaximumFlowInputError(f"{field} has too many UTF-8 bytes")
    return len(encoded)


def maximum_flow_and_minimum_cut(
    nodes: tuple[str, ...],
    edges: tuple[DirectedCapacityEdge, ...],
    *,
    source: str,
    sink: str,
) -> MaximumFlowCut:
    """Return the exact maximum value and canonical minimum-cut partitions."""
    if type(nodes) is not tuple:
        raise TypeError("nodes must be an exact tuple")
    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if not 2 <= len(nodes) <= _MAX_NODES:
        raise MaximumFlowInputError("node count is outside the supported range")
    if len(edges) > _MAX_EDGES:
        raise MaximumFlowInputError("edge count exceeds the supported limit")

    positions: dict[str, int] = {}
    total_node_bytes = 0
    for node_index, node in enumerate(nodes):
        node_bytes = _node_byte_length(node, field=f"nodes[{node_index}]")
        if node_bytes > _MAX_TOTAL_NODE_BYTES - total_node_bytes:
            raise MaximumFlowInputError("node names exceed the aggregate byte limit")
        if node in positions:
            raise MaximumFlowInputError(f"duplicate node at nodes[{node_index}]")
        positions[node] = node_index
        total_node_bytes += node_bytes

    _node_byte_length(source, field="source")
    _node_byte_length(sink, field="sink")
    if source not in positions or sink not in positions:
        raise MaximumFlowInputError("source and sink must be registered nodes")
    if source == sink:
        raise MaximumFlowInputError("source and sink must be distinct")

    validated_edges: list[tuple[int, int, int]] = []
    seen_pairs: set[tuple[int, int]] = set()
    for edge_index, edge in enumerate(edges):
        if type(edge) is not DirectedCapacityEdge:
            raise TypeError(f"edges[{edge_index}] must be an exact DirectedCapacityEdge")
        if type(edge.source) is not str or type(edge.target) is not str:
            raise TypeError(f"edges[{edge_index}] endpoints must be exact strings")
        if edge.source not in positions or edge.target not in positions:
            raise MaximumFlowInputError(f"edges[{edge_index}] refers to an unknown node")
        source_position = positions[edge.source]
        target_position = positions[edge.target]
        if source_position == target_position:
            raise MaximumFlowInputError(f"edges[{edge_index}] is a self-edge")
        if type(edge.capacity) is not int:
            raise TypeError(f"edges[{edge_index}].capacity must be an exact integer")
        if not 1 <= edge.capacity <= _MAX_INT64:
            raise MaximumFlowInputError(
                f"edges[{edge_index}].capacity is outside the positive signed 64-bit range"
            )

        pair = (source_position, target_position)
        if pair in seen_pairs:
            raise MaximumFlowInputError(f"edges[{edge_index}] duplicates a directed endpoint pair")
        seen_pairs.add(pair)
        validated_edges.append((*pair, edge.capacity))

    residual = [[0] * len(nodes) for _ in nodes]
    adjacency_sets: list[set[int]] = [set() for _ in nodes]
    for source_position, target_position, capacity in validated_edges:
        residual[source_position][target_position] = capacity
        adjacency_sets[source_position].add(target_position)
        adjacency_sets[target_position].add(source_position)
    adjacency = tuple(tuple(sorted(neighbors)) for neighbors in adjacency_sets)

    source_position = positions[source]
    sink_position = positions[sink]
    maximum_value = 0

    while True:
        parents = [-1] * len(nodes)
        parents[source_position] = source_position
        queue = deque([source_position])

        while queue and parents[sink_position] == -1:
            current = queue.popleft()
            for neighbor in adjacency[current]:
                if parents[neighbor] != -1 or residual[current][neighbor] <= 0:
                    continue
                parents[neighbor] = current
                if neighbor == sink_position:
                    break
                queue.append(neighbor)

        if parents[sink_position] == -1:
            break

        path_capacity: int | None = None
        cursor = sink_position
        while cursor != source_position:
            parent = parents[cursor]
            edge_capacity = residual[parent][cursor]
            path_capacity = (
                edge_capacity if path_capacity is None else min(path_capacity, edge_capacity)
            )
            cursor = parent
        if path_capacity is None:
            raise AssertionError("an augmenting path must contain an edge")

        cursor = sink_position
        while cursor != source_position:
            parent = parents[cursor]
            residual[parent][cursor] -= path_capacity
            residual[cursor][parent] += path_capacity
            cursor = parent
        maximum_value += path_capacity

    reachable = [False] * len(nodes)
    reachable[source_position] = True
    queue = deque([source_position])
    while queue:
        current = queue.popleft()
        for neighbor in adjacency[current]:
            if reachable[neighbor] or residual[current][neighbor] <= 0:
                continue
            reachable[neighbor] = True
            queue.append(neighbor)

    return MaximumFlowCut(
        value=maximum_value,
        source_side=tuple(
            node for node, is_reachable in zip(nodes, reachable, strict=True) if is_reachable
        ),
        sink_side=tuple(
            node for node, is_reachable in zip(nodes, reachable, strict=True) if not is_reachable
        ),
    )
```

## Example

```python
nodes = ("source", "left", "right", "sink", "isolated")
edges = (
    DirectedCapacityEdge("source", "left", 5),
    DirectedCapacityEdge("source", "right", 4),
    DirectedCapacityEdge("left", "sink", 2),
    DirectedCapacityEdge("right", "sink", 3),
    DirectedCapacityEdge("left", "right", 2),
    DirectedCapacityEdge("right", "left", 1),
)
expected = MaximumFlowCut(
    value=5,
    source_side=("source", "left", "right"),
    sink_side=("sink", "isolated"),
)

forward = maximum_flow_and_minimum_cut(
    nodes,
    edges,
    source="source",
    sink="sink",
)
reordered = maximum_flow_and_minimum_cut(
    nodes,
    tuple(reversed(edges)),
    source="source",
    sink="sink",
)

try:
    maximum_flow_and_minimum_cut(
        nodes,
        (*edges, DirectedCapacityEdge("left", "left", 1)),
        source="source",
        sink="sink",
    )
except MaximumFlowInputError:
    self_edge_rejected = True
else:
    self_edge_rejected = False

assert (forward, reordered, self_edge_rejected) == (expected, expected, True)
```

## Trade-offs and Limitations

Validation takes `O(V + E)` time before residual state exists. Edmonds-Karp
uses `O(VE^2)` time. The matrix-backed residual network, sorted adjacency,
search state, and frozen result use `O(V^2 + E)` memory within the fixed node
and edge limits. Input capacities are positive signed 64-bit integers, while
Python keeps the maximum-flow total exact beyond that range.

After augmentation stops, residual reachability from the source gives a
minimum cut. Its source side is contained in every other minimum-cut source
side and is therefore their intersection, making the returned partition
canonical. Both partitions follow node declaration order; edge input order and
BFS path choices do not change them.

Antiparallel edges are valid. A reverse residual combines unused original
capacity in that direction with capacity that can cancel opposite net flow.
This representation preserves the maximum value and canonical cut, but it does
not retain enough information to report a separate flow for each original
edge.

The function rejects self-edges, repeated ordered endpoint pairs, zero or float
capacities, unknown nodes, and equal terminals. It provides no lower bounds,
costs, multiple terminals, edge-flow decomposition, augmenting paths, dynamic
updates, persistence, or concurrency control.

## Related Snippets

<!-- catalog:related:start -->
- [Compute Bounded Bellman-Ford Distances and Detect Reachable Negative Cycles](compute-bounded-bellman-ford-distances-and-detect-reachable-negative-cycles.md)
- [Find One Deterministic Shortest Path in a Bounded Non-Negatively Weighted Directed Graph](find-one-deterministic-shortest-path-in-a-bounded-non-negatively-weighted-directed-graph.md)
- [Build a Deterministic Minimum Spanning Forest with Kruskal's Algorithm](build-a-deterministic-minimum-spanning-forest-with-kruskals-algorithm.md)
<!-- catalog:related:end -->
