---
title: "Build a Deterministic Minimum Spanning Forest with Kruskal's Algorithm"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - partition-a-bounded-undirected-graph-into-deterministic-components-with-union-find.md
  - find-one-deterministic-shortest-path-in-a-bounded-non-negatively-weighted-directed-graph.md
  - partition-a-bounded-directed-graph-into-deterministic-strongly-connected-components.md
---

# Build a Deterministic Minimum Spanning Forest with Kruskal's Algorithm

## Idea and Problem

Choose one minimum-weight spanning tree for every connected component while making equal-weight edge selection independent of input edge order and orientation.

Node declaration position gives each undirected endpoint pair one canonical
form. After the entire graph input is validated, sort edges by weight and
canonical endpoint positions. Union-find accepts an edge only when it joins two
current components, which is Kruskal's greedy rule.

## When to Use

Use this algorithm for a fully available bounded undirected graph snapshot when
every original connected component needs a minimum-cost acyclic connection.
Signed weights are valid, isolated declared nodes remain zero-edge components,
and the caller must accept node declaration order as the equal-weight policy.

Use a shortest-path algorithm when the objective concerns one route rather
than the total forest. Use a specialized graph library for large graphs,
parallel edges, dynamic updates, degree-constrained trees, or enumeration of
alternative minimum forests.

## Implementation

```python
from dataclasses import dataclass

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_NODES = 2_048
_MAX_EDGES = 20_000
_MAX_NODE_CHARACTERS = 128
_MAX_NODE_BYTES = 128
_MAX_TOTAL_NODE_BYTES = 131_072


class MinimumSpanningForestError(ValueError):
    """Raised when a graph violates the bounded forest contract."""


@dataclass(frozen=True, slots=True)
class WeightedUndirectedEdge:
    first: str
    second: str
    weight: int


@dataclass(frozen=True, slots=True)
class MinimumSpanningForest:
    edges: tuple[WeightedUndirectedEdge, ...]
    total_weight: int
    component_count: int


def _node_byte_length(value: object, *, field: str) -> int:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if not value:
        raise MinimumSpanningForestError(f"{field} must not be empty")
    if len(value) > _MAX_NODE_CHARACTERS:
        raise MinimumSpanningForestError(f"{field} has too many characters")
    try:
        encoded = value.encode("utf-8")
    except UnicodeEncodeError:
        raise MinimumSpanningForestError(f"{field} must be valid UTF-8 text") from None
    if len(encoded) > _MAX_NODE_BYTES:
        raise MinimumSpanningForestError(f"{field} has too many UTF-8 bytes")
    return len(encoded)


def minimum_spanning_forest(
    nodes: tuple[str, ...],
    edges: tuple[WeightedUndirectedEdge, ...],
) -> MinimumSpanningForest:
    """Return one declaration-order-deterministic minimum spanning forest."""
    if type(nodes) is not tuple:
        raise TypeError("nodes must be an exact tuple")
    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if not 1 <= len(nodes) <= _MAX_NODES:
        raise MinimumSpanningForestError("node count is outside the supported range")
    if len(edges) > _MAX_EDGES:
        raise MinimumSpanningForestError("edge count exceeds the supported limit")

    position: dict[str, int] = {}
    total_node_bytes = 0
    for index, node in enumerate(nodes):
        total_node_bytes += _node_byte_length(node, field=f"nodes[{index}]")
        if total_node_bytes > _MAX_TOTAL_NODE_BYTES:
            raise MinimumSpanningForestError("node names exceed the aggregate byte limit")
        if node in position:
            raise MinimumSpanningForestError(f"duplicate node at nodes[{index}]")
        position[node] = index

    validated_edges: list[tuple[int, int, int]] = []
    seen_pairs: set[tuple[int, int]] = set()
    for index, edge in enumerate(edges):
        if type(edge) is not WeightedUndirectedEdge:
            raise TypeError(f"edges[{index}] must be an exact WeightedUndirectedEdge")
        if type(edge.first) is not str or type(edge.second) is not str:
            raise TypeError(f"edges[{index}] endpoints must be exact strings")
        if edge.first not in position or edge.second not in position:
            raise MinimumSpanningForestError(f"edges[{index}] refers to an unknown node")
        if type(edge.weight) is not int:
            raise TypeError(f"edges[{index}].weight must be an exact integer")
        if not _MIN_INT64 <= edge.weight <= _MAX_INT64:
            raise MinimumSpanningForestError(
                f"edges[{index}].weight is outside the signed 64-bit range"
            )

        first_position = position[edge.first]
        second_position = position[edge.second]
        if first_position == second_position:
            raise MinimumSpanningForestError(f"edges[{index}] is a self-edge")
        lower_position, upper_position = sorted((first_position, second_position))
        pair = (lower_position, upper_position)
        if pair in seen_pairs:
            raise MinimumSpanningForestError(
                f"edges[{index}] duplicates an undirected endpoint pair"
            )
        seen_pairs.add(pair)
        validated_edges.append((edge.weight, lower_position, upper_position))

    parent = list(range(len(nodes)))
    size = [1] * len(nodes)

    def find(node_position: int) -> int:
        while parent[node_position] != node_position:
            parent[node_position] = parent[parent[node_position]]
            node_position = parent[node_position]
        return node_position

    selected_edges: list[WeightedUndirectedEdge] = []
    total_weight = 0
    component_count = len(nodes)

    for weight, lower_position, upper_position in sorted(validated_edges):
        lower_root = find(lower_position)
        upper_root = find(upper_position)
        if lower_root == upper_root:
            continue
        if size[lower_root] < size[upper_root] or (
            size[lower_root] == size[upper_root] and lower_root > upper_root
        ):
            lower_root, upper_root = upper_root, lower_root
        parent[upper_root] = lower_root
        size[lower_root] += size[upper_root]
        selected_edges.append(
            WeightedUndirectedEdge(
                first=nodes[lower_position],
                second=nodes[upper_position],
                weight=weight,
            )
        )
        total_weight += weight
        component_count -= 1

    return MinimumSpanningForest(
        edges=tuple(selected_edges),
        total_weight=total_weight,
        component_count=component_count,
    )
```

## Example

```python
nodes = ("alpha", "beta", "gamma", "delta", "epsilon", "isolated")
edges = (
    WeightedUndirectedEdge("gamma", "beta", 1),
    WeightedUndirectedEdge("gamma", "alpha", 1),
    WeightedUndirectedEdge("epsilon", "delta", -2),
    WeightedUndirectedEdge("alpha", "beta", 1),
)
expected = MinimumSpanningForest(
    edges=(
        WeightedUndirectedEdge("delta", "epsilon", -2),
        WeightedUndirectedEdge("alpha", "beta", 1),
        WeightedUndirectedEdge("alpha", "gamma", 1),
    ),
    total_weight=0,
    component_count=3,
)

reoriented = tuple(
    WeightedUndirectedEdge(edge.second, edge.first, edge.weight)
    for edge in reversed(edges)
)

try:
    minimum_spanning_forest(
        nodes,
        (*edges, WeightedUndirectedEdge("beta", "alpha", 5)),
    )
except MinimumSpanningForestError:
    duplicate_pair_rejected = True
else:
    duplicate_pair_rejected = False

assert (
    minimum_spanning_forest(nodes, edges),
    minimum_spanning_forest(nodes, reoriented),
    duplicate_pair_rejected,
) == (expected, expected, True)
```

## Trade-offs and Limitations

Validating names and edges takes `O(V + E)` time before union-find is mutated.
Sorting costs `O(E log E)` time. Union by size with path compression has
near-constant amortized operations, while validation, sorting, union-find, and
the result use `O(V + E)` memory within the fixed limits.

The forest contains one minimum spanning tree for every connected component of
the original graph. The component count includes declared nodes with no edges;
the caller can combine the original node tuple with the selected edges to
recover each membership. Per-edge weights are signed 64-bit integers, while
Python keeps the aggregate total exact even if it leaves that range.

Sorting by `(weight, lower_position, upper_position)` selects one reproducible
Kruskal result under the declared node order. It does not prove that the
minimum forest is unique or promise another lexicographic optimum. Changing
node declaration order can change equal-weight choices. Node strings are
compared exactly without Unicode normalization.

The function rejects self-edges, parallel endpoint pairs, unknown nodes, float
weights, directed edges, and an empty node declaration. It does not connect
originally disconnected components, return paths, support updates or degree
constraints, or enumerate every minimum forest.

## Related Snippets

<!-- catalog:related:start -->
- [Partition a Bounded Undirected Graph into Deterministic Components with Union-Find](partition-a-bounded-undirected-graph-into-deterministic-components-with-union-find.md)
- [Find One Deterministic Shortest Path in a Bounded Non-Negatively Weighted Directed Graph](find-one-deterministic-shortest-path-in-a-bounded-non-negatively-weighted-directed-graph.md)
- [Partition a Bounded Directed Graph into Deterministic Strongly Connected Components](partition-a-bounded-directed-graph-into-deterministic-strongly-connected-components.md)
<!-- catalog:related:end -->
