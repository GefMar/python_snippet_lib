---
title: "Partition a Bounded Undirected Graph into Deterministic Components with Union-Find"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - partition-a-bounded-directed-graph-into-deterministic-strongly-connected-components.md
  - traverse-a-parent-graph-with-breadth-first-search.md
  - partition-tagged-items-into-minimum-stable-conflict-free-groups.md
---

# Partition a Bounded Undirected Graph into Deterministic Components with Union-Find

## Idea and Problem

Partition a bounded undirected graph with union-find while normalizing members and components by declared node position for deterministic public output.

This separates an efficient mutable implementation detail from a deterministic,
immutable output contract.

## When to Use

Use this for a complete, bounded snapshot of an undirected graph when only connected
components are required. Typical inputs include small relation batches, equivalence
groups, and validation-time dependency clusters whose node declaration order is the
desired presentation order.

Choose a traversal-based or dynamic graph structure when you also need paths,
directed reachability, edge deletion, rollback, or incremental query state.

## Implementation

```python
from dataclasses import dataclass
from re import fullmatch

_MAX_NODES = 256
_MAX_EDGES = 2_048
_NODE_PATTERN = r"[A-Za-z0-9][A-Za-z0-9._:-]{0,63}"


class UndirectedGraphError(ValueError):
    """Raised when an undirected graph violates the bounded input contract."""


@dataclass(frozen=True, slots=True)
class UndirectedEdge:
    first: str
    second: str


def _validate_node_name(value: object, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact str")
    if fullmatch(_NODE_PATTERN, value, flags=0) is None:
        raise UndirectedGraphError(f"{field} is not a valid node name")
    return value


def partition_undirected_components(
    nodes: tuple[str, ...],
    edges: tuple[UndirectedEdge, ...],
) -> tuple[tuple[str, ...], ...]:
    """Return connected components normalized by node declaration position."""
    if type(nodes) is not tuple:
        raise TypeError("nodes must be an exact tuple")
    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if len(nodes) > _MAX_NODES:
        raise UndirectedGraphError(f"nodes must contain at most {_MAX_NODES} items")
    if len(edges) > _MAX_EDGES:
        raise UndirectedGraphError(f"edges must contain at most {_MAX_EDGES} items")

    position: dict[str, int] = {}
    for index, raw_name in enumerate(nodes):
        name = _validate_node_name(raw_name, f"nodes[{index}]")
        if name in position:
            raise UndirectedGraphError(f"duplicate node: {name!r}")
        position[name] = index

    parent = list(range(len(nodes)))
    size = [1] * len(nodes)

    def find(index: int) -> int:
        while parent[index] != index:
            parent[index] = parent[parent[index]]
            index = parent[index]
        return index

    for edge_index, edge in enumerate(edges):
        if type(edge) is not UndirectedEdge:
            raise TypeError(f"edges[{edge_index}] must be an exact UndirectedEdge")
        first = _validate_node_name(edge.first, f"edges[{edge_index}].first")
        second = _validate_node_name(edge.second, f"edges[{edge_index}].second")
        if first not in position or second not in position:
            raise UndirectedGraphError(f"edges[{edge_index}] refers to an unknown node")

        first_root = find(position[first])
        second_root = find(position[second])
        if first_root == second_root:
            continue
        if size[first_root] < size[second_root]:
            first_root, second_root = second_root, first_root
        parent[second_root] = first_root
        size[first_root] += size[second_root]

    members_by_root: dict[int, list[str]] = {}
    for index, name in enumerate(nodes):
        members_by_root.setdefault(find(index), []).append(name)

    components = list(members_by_root.values())
    components.sort(key=lambda members: position[members[0]])
    return tuple(tuple(members) for members in components)
```

## Example

```python
nodes = ("alpha", "beta", "gamma", "delta", "epsilon", "isolated")
edges = (
    UndirectedEdge("beta", "gamma"),
    UndirectedEdge("alpha", "beta"),
    UndirectedEdge("epsilon", "delta"),
    UndirectedEdge("gamma", "alpha"),
    UndirectedEdge("alpha", "alpha"),
)
expected = (("alpha", "beta", "gamma"), ("delta", "epsilon"), ("isolated",))
reordered = tuple(UndirectedEdge(edge.second, edge.first) for edge in reversed(edges))

unknown_node_rejected = False
try:
    partition_undirected_components(nodes, (UndirectedEdge("alpha", "missing"),))
except UndirectedGraphError:
    unknown_node_rejected = True

assert (
    partition_undirected_components(nodes, edges) == expected
    and partition_undirected_components(nodes, reordered) == expected
    and unknown_node_rejected
)
```

## Trade-offs and Limitations

Union by size and path compression give near-constant amortized union and lookup
cost, conventionally described with the inverse-Ackermann factor. Normalization
stores all `V` nodes and orders the component list. The configured limits keep the
complete snapshot and its validation work bounded.

Self-edges and repeated undirected edges are accepted as harmless redundancy. The
returned component and member order is deterministic, but internal parent roots are
deliberately not part of the API and may depend on edge order. The function models
only closed, undirected connectivity: it does not provide paths, weights, directed
components, deletion, rollback, or a persistent incremental index.

## Related Snippets

<!-- catalog:related:start -->
- [Partition a Bounded Directed Graph into Deterministic Strongly Connected Components](partition-a-bounded-directed-graph-into-deterministic-strongly-connected-components.md)
- [Traverse a Parent Graph with Breadth-First Search](traverse-a-parent-graph-with-breadth-first-search.md)
- [Partition Tagged Items into Minimum Stable Conflict-Free Groups](partition-tagged-items-into-minimum-stable-conflict-free-groups.md)
<!-- catalog:related:end -->
