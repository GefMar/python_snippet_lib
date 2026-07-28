---
title: "Compute the Transitive Reduction of a Bounded Directed Acyclic Graph"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - resolve-stable-ordering-constraints-with-topological-sort.md
  - partition-a-bounded-directed-graph-into-deterministic-strongly-connected-components.md
  - classify-bounded-dag-stages-as-ready-waiting-or-blocked.md
---

# Compute the Transitive Reduction of a Bounded Directed Acyclic Graph

## Idea and Problem

Remove redundant edges from a bounded directed acyclic graph without changing which declared nodes can reach one another.

An edge is unnecessary when its target remains reachable from its source after
that edge alone is omitted. Testing every edge against the complete validated
graph yields the unique transitive reduction of a DAG. Ordering the retained
edges by declared node positions makes the result independent of input edge
order.

## When to Use

Use this algorithm for a complete, small DAG whose edges express dependency or
precedence and whose direct view should omit relationships already implied by
longer paths. It is useful before rendering a dependency diagram or presenting
the smallest direct edge set with the same reachability.

Confirm that reachability, rather than edge history or weight, is the
relevant meaning. Keep the original graph when redundant declarations are
intentional evidence. Use a specialized graph library for large graphs,
incremental updates, path indexes, or cyclic minimum-equivalent-graph problems.

## Implementation

```python
from collections import deque
from dataclasses import dataclass

_MAX_NODES = 64
_MAX_EDGES = 512
_MAX_NAME_CHARACTERS = 128
_MAX_NAME_BYTES = 128
_MAX_TOTAL_NAME_BYTES = 8_192


@dataclass(frozen=True, slots=True)
class DirectedEdge:
    source: str
    target: str


def _validated_node_name(value: object, *, field: str) -> tuple[str, int]:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if not 1 <= len(value) <= _MAX_NAME_CHARACTERS:
        raise ValueError(f"{field} character count is outside the supported range")
    try:
        encoded = value.encode("utf-8")
    except UnicodeEncodeError:
        raise ValueError(f"{field} must be valid UTF-8 text") from None
    if len(encoded) > _MAX_NAME_BYTES:
        raise ValueError(f"{field} exceeds the supported UTF-8 byte length")
    return value, len(encoded)


def _validate_graph_boundary(
    nodes: object,
    edges: object,
) -> tuple[tuple[str, ...], tuple[tuple[int, int], ...]]:
    if type(nodes) is not tuple:
        raise TypeError("nodes must be an exact tuple")
    if not 1 <= len(nodes) <= _MAX_NODES:
        raise ValueError("node count is outside the supported range")

    ordered_names: list[str] = []
    positions: dict[str, int] = {}
    total_name_bytes = 0
    for index, raw_name in enumerate(nodes):
        name, byte_count = _validated_node_name(
            raw_name,
            field=f"nodes[{index}]",
        )
        if name in positions:
            raise ValueError("node names must be unique")
        total_name_bytes += byte_count
        if total_name_bytes > _MAX_TOTAL_NAME_BYTES:
            raise ValueError("node names exceed the aggregate UTF-8 byte limit")
        positions[name] = index
        ordered_names.append(name)

    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if len(edges) > _MAX_EDGES:
        raise ValueError("edge count exceeds the supported range")

    seen_pairs: set[tuple[int, int]] = set()
    indexed_edges: list[tuple[int, int]] = []
    for index, edge in enumerate(edges):
        if type(edge) is not DirectedEdge:
            raise TypeError(f"edges[{index}] must be an exact DirectedEdge")
        if type(edge.source) is not str or type(edge.target) is not str:
            raise TypeError("edge endpoints must be exact strings")
        if edge.source not in positions or edge.target not in positions:
            raise ValueError("every edge endpoint must name a declared node")

        pair = (positions[edge.source], positions[edge.target])
        if pair[0] == pair[1]:
            raise ValueError("edge endpoints must be distinct")
        if pair in seen_pairs:
            raise ValueError("directed edges must be unique")
        seen_pairs.add(pair)
        indexed_edges.append(pair)

    return tuple(ordered_names), tuple(sorted(indexed_edges))


def _build_adjacency(
    node_count: int,
    edges: tuple[tuple[int, int], ...],
) -> tuple[tuple[int, ...], ...]:
    mutable: list[list[int]] = [[] for _ in range(node_count)]
    for source, target in edges:
        mutable[source].append(target)
    return tuple(tuple(targets) for targets in mutable)


def _require_acyclic(adjacency: tuple[tuple[int, ...], ...]) -> None:
    indegrees = [0] * len(adjacency)
    for targets in adjacency:
        for target in targets:
            indegrees[target] += 1

    ready = deque(index for index, degree in enumerate(indegrees) if degree == 0)
    visited_count = 0
    while ready:
        source = ready.popleft()
        visited_count += 1
        for target in adjacency[source]:
            indegrees[target] -= 1
            if indegrees[target] == 0:
                ready.append(target)

    if visited_count != len(adjacency):
        raise ValueError("the directed graph must be acyclic")


def _reachable_without_edge(
    source: int,
    target: int,
    adjacency: tuple[tuple[int, ...], ...],
) -> bool:
    visited = [False] * len(adjacency)
    visited[source] = True
    stack = [source]

    while stack:
        current = stack.pop()
        for candidate in reversed(adjacency[current]):
            if current == source and candidate == target:
                continue
            if candidate == target:
                return True
            if not visited[candidate]:
                visited[candidate] = True
                stack.append(candidate)
    return False


def transitive_reduction(
    nodes: tuple[str, ...],
    edges: tuple[DirectedEdge, ...],
) -> tuple[DirectedEdge, ...]:
    """Return the unique reachability-preserving reduction of a bounded DAG."""
    names, indexed_edges = _validate_graph_boundary(nodes, edges)
    adjacency = _build_adjacency(len(names), indexed_edges)
    _require_acyclic(adjacency)

    retained: list[DirectedEdge] = []
    for source, target in indexed_edges:
        if not _reachable_without_edge(source, target, adjacency):
            retained.append(DirectedEdge(names[source], names[target]))
    return tuple(retained)
```

## Example

```python
nodes = ("fetch", "parse", "validate", "store", "notify")
edges = (
    DirectedEdge("fetch", "parse"),
    DirectedEdge("fetch", "validate"),
    DirectedEdge("fetch", "store"),
    DirectedEdge("parse", "validate"),
    DirectedEdge("parse", "store"),
    DirectedEdge("parse", "notify"),
    DirectedEdge("validate", "store"),
    DirectedEdge("store", "notify"),
)
expected = (
    DirectedEdge("fetch", "parse"),
    DirectedEdge("parse", "validate"),
    DirectedEdge("validate", "store"),
    DirectedEdge("store", "notify"),
)

try:
    transitive_reduction(
        nodes,
        (*edges, DirectedEdge("notify", "fetch")),
    )
except ValueError:
    cycle_rejected = True
else:
    cycle_rejected = False

assert (
    transitive_reduction(nodes, edges),
    transitive_reduction(nodes, tuple(reversed(edges))),
    transitive_reduction(("solo",), ()),
    cycle_rejected,
    nodes,
) == (
    expected,
    expected,
    (),
    True,
    ("fetch", "parse", "validate", "store", "notify"),
)
```

## Trade-offs and Limitations

Boundary validation takes `O(V + E)` time, canonical edge ordering takes
`O(E log E)`, and adjacency construction plus cycle detection take
`O(V + E)`. Each of the `E` omission checks takes `O(V + E)` time, so a graph
with edges takes `O(E(V + E))` total time; an edgeless graph takes `O(V)`.
Adjacency, traversal state, and the returned edge tuple use `O(V + E)` memory,
while one omission check uses `O(V)` temporary state.

The result is unique only because the accepted graph is a DAG. The function
does not compute a cyclic minimum equivalent graph, transitive closure, paths,
weights, edge history, or every redundant-path witness. It also excludes
incremental updates and does not mutate either input tuple. The repeated
reachability scans favor a small auditable implementation over asymptotically
faster bit-set or matrix methods for larger graphs.

## Related Snippets

<!-- catalog:related:start -->
- [Resolve Stable Ordering Constraints with Topological Sort](resolve-stable-ordering-constraints-with-topological-sort.md)
- [Partition a Bounded Directed Graph into Deterministic Strongly Connected Components](partition-a-bounded-directed-graph-into-deterministic-strongly-connected-components.md)
- [Classify Bounded DAG Stages as Ready, Waiting, or Blocked](classify-bounded-dag-stages-as-ready-waiting-or-blocked.md)
<!-- catalog:related:end -->
