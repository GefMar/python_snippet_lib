---
title: "Partition a Bounded Directed Graph into Deterministic Strongly Connected Components"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - classify-bounded-dag-stages-as-ready-waiting-or-blocked.md
  - resolve-stable-ordering-constraints-with-topological-sort.md
  - traverse-a-parent-graph-with-breadth-first-search.md
---

# Partition a Bounded Directed Graph into Deterministic Strongly Connected Components

## Idea and Problem

Partition a closed directed graph into maximal mutually reachable components and derive its component-level directed acyclic graph with deterministic tuple ordering.

Two iterative depth-first passes implement Kosaraju's algorithm without
depending on the interpreter's recursion limit. Components are ordered by
their earliest declared member, members retain declaration order, and outgoing
component identifiers are sorted, so traversal details do not leak into the
published result.

## When to Use

Use this algorithm after collecting one complete, small graph when cycles are
meaningful groups rather than invalid input. It fits dependency analysis,
module grouping, and cycle-aware planning where every target must name a
declared node and a deterministic condensation graph is useful downstream.

Use topological sorting directly when the original graph must be acyclic. Use
a specialized graph library for much larger graphs, dynamic updates, weighted
edges, path queries, or detailed cycle witnesses.

## Implementation

```python
import re
from dataclasses import dataclass

_MAX_NODES = 64
_MAX_TARGETS_PER_NODE = 16
_MAX_EDGES = 256
_MAX_NAME_BYTES = 64
_MAX_TOTAL_TEXT_BYTES = 4_096
_NODE_NAME = re.compile(r"[A-Za-z0-9][A-Za-z0-9._:-]{0,63}", re.ASCII)


@dataclass(frozen=True, slots=True)
class DirectedNode:
    name: str
    targets: tuple[str, ...] = ()


@dataclass(frozen=True, slots=True)
class StronglyConnectedPartition:
    components: tuple[tuple[str, ...], ...]
    condensation_targets: tuple[tuple[int, ...], ...]


def _validated_name(value: object, *, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if not 1 <= len(value) <= _MAX_NAME_BYTES:
        raise ValueError(f"{field} length is outside the supported range")
    if _NODE_NAME.fullmatch(value) is None:
        raise ValueError(f"{field} must be a conservative ASCII node name")
    return value


def _validate_graph(
    nodes: object,
) -> tuple[tuple[str, ...], tuple[tuple[int, ...], ...]]:
    if type(nodes) is not tuple:
        raise TypeError("nodes must be an exact tuple")
    if len(nodes) > _MAX_NODES:
        raise ValueError("nodes may contain at most 64 definitions")

    ordered_names: list[str] = []
    raw_targets: list[tuple[str, ...]] = []
    positions: dict[str, int] = {}
    edge_count = 0
    text_bytes = 0

    for node_index, node in enumerate(nodes):
        if type(node) is not DirectedNode:
            raise TypeError(f"nodes[{node_index}] must be an exact DirectedNode")
        name = _validated_name(node.name, field=f"nodes[{node_index}].name")
        if name in positions:
            raise ValueError("node names must be unique")
        if type(node.targets) is not tuple:
            raise TypeError(f"targets for node {name!r} must be an exact tuple")
        if len(node.targets) > _MAX_TARGETS_PER_NODE:
            raise ValueError(f"node {name!r} has more than 16 targets")

        text_bytes += len(name.encode("utf-8"))
        targets: list[str] = []
        seen_targets: set[str] = set()
        for target_index, raw_target in enumerate(node.targets):
            target = _validated_name(
                raw_target,
                field=f"nodes[{node_index}].targets[{target_index}]",
            )
            if target in seen_targets:
                raise ValueError(f"node {name!r} repeats a target")
            seen_targets.add(target)
            targets.append(target)
            text_bytes += len(target.encode("utf-8"))

        edge_count += len(targets)
        if edge_count > _MAX_EDGES:
            raise ValueError("the graph may contain at most 256 edges")
        if text_bytes > _MAX_TOTAL_TEXT_BYTES:
            raise ValueError("graph text exceeds the supported byte limit")
        positions[name] = len(ordered_names)
        ordered_names.append(name)
        raw_targets.append(tuple(targets))

    adjacency: list[tuple[int, ...]] = []
    for source_index, targets in enumerate(raw_targets):
        target_positions: list[int] = []
        for target in targets:
            if target not in positions:
                raise ValueError(
                    f"node {ordered_names[source_index]!r} has an undeclared target"
                )
            target_positions.append(positions[target])
        adjacency.append(tuple(target_positions))
    return tuple(ordered_names), tuple(adjacency)


def _iterative_finish_order(
    adjacency: tuple[tuple[int, ...], ...],
) -> tuple[int, ...]:
    visited = [False] * len(adjacency)
    finished: list[int] = []

    for start in range(len(adjacency)):
        if visited[start]:
            continue
        visited[start] = True
        stack: list[tuple[int, int]] = [(start, 0)]
        while stack:
            node, target_index = stack[-1]
            if target_index == len(adjacency[node]):
                stack.pop()
                finished.append(node)
                continue
            stack[-1] = (node, target_index + 1)
            target = adjacency[node][target_index]
            if not visited[target]:
                visited[target] = True
                stack.append((target, 0))
    return tuple(finished)


def _reverse_adjacency(
    adjacency: tuple[tuple[int, ...], ...],
) -> tuple[tuple[int, ...], ...]:
    mutable_reverse: list[list[int]] = [[] for _ in adjacency]
    for source, targets in enumerate(adjacency):
        for target in targets:
            mutable_reverse[target].append(source)
    return tuple(tuple(sources) for sources in mutable_reverse)


def _component_members(
    adjacency: tuple[tuple[int, ...], ...],
) -> list[tuple[int, ...]]:
    finish_order = _iterative_finish_order(adjacency)
    reverse = _reverse_adjacency(adjacency)
    assigned = [False] * len(adjacency)
    components: list[tuple[int, ...]] = []

    for start in reversed(finish_order):
        if assigned[start]:
            continue
        assigned[start] = True
        stack = [start]
        members: list[int] = []
        while stack:
            node = stack.pop()
            members.append(node)
            for source in reversed(reverse[node]):
                if not assigned[source]:
                    assigned[source] = True
                    stack.append(source)
        components.append(tuple(sorted(members)))

    components.sort(key=lambda members: members[0])
    return components


def partition_strongly_connected_components(
    nodes: tuple[DirectedNode, ...],
) -> StronglyConnectedPartition:
    names, adjacency = _validate_graph(nodes)
    member_positions = _component_members(adjacency)
    component_by_node = [0] * len(names)
    for component_index, members in enumerate(member_positions):
        for member in members:
            component_by_node[member] = component_index

    condensation_sets: list[set[int]] = [set() for _ in member_positions]
    for source, targets in enumerate(adjacency):
        source_component = component_by_node[source]
        for target in targets:
            target_component = component_by_node[target]
            if source_component != target_component:
                condensation_sets[source_component].add(target_component)

    return StronglyConnectedPartition(
        components=tuple(
            tuple(names[member] for member in members)
            for members in member_positions
        ),
        condensation_targets=tuple(
            tuple(sorted(targets)) for targets in condensation_sets
        ),
    )
```

## Example

```python
graph = (
    DirectedNode("api", ("parse",)),
    DirectedNode("parse", ("validate",)),
    DirectedNode("validate", ("parse", "store")),
    DirectedNode("store", ("notify",)),
    DirectedNode("notify", ("store",)),
    DirectedNode("isolated"),
)
partition = partition_strongly_connected_components(graph)

reordered_targets = (
    DirectedNode("api", ("parse",)),
    DirectedNode("parse", ("validate",)),
    DirectedNode("validate", ("store", "parse")),
    DirectedNode("store", ("notify",)),
    DirectedNode("notify", ("store",)),
    DirectedNode("isolated"),
)

try:
    partition_strongly_connected_components(
        (DirectedNode("known", ("missing",)),)
    )
except ValueError:
    undeclared_target_rejected = True
else:
    undeclared_target_rejected = False

try:
    partition_strongly_connected_components(
        (DirectedNode("same", ("same", "same")),)
    )
except ValueError:
    duplicate_edge_rejected = True
else:
    duplicate_edge_rejected = False

assert (
    partition,
    partition_strongly_connected_components(reordered_targets),
    partition_strongly_connected_components(()),
    graph[2].targets,
    undeclared_target_rejected,
    duplicate_edge_rejected,
) == (
    StronglyConnectedPartition(
        components=(
            ("api",),
            ("parse", "validate"),
            ("store", "notify"),
            ("isolated",),
        ),
        condensation_targets=((1,), (2,), (), ()),
    ),
    partition,
    StronglyConnectedPartition(components=(), condensation_targets=()),
    ("parse", "store"),
    True,
    True,
)
```

## Trade-offs and Limitations

Validation and partitioning take O(V + E) time and memory within the 64-node,
16-target-per-node, 256-edge, and 4,096-text-byte limits. The algorithm keeps
both forward and reverse adjacency plus explicit DFS stacks. It performs no
recursive calls and does not mutate the caller's tuple or frozen node values.

Component identifiers are their positions in the returned component tuple.
They follow the earliest declared member, not a topological order. Members use
declaration order, and condensation targets use increasing component IDs.
Reordering target tuples therefore cannot change the result, while reordering
node declarations can renumber components without changing graph semantics.
Self-edges are accepted inside a component and omitted from the condensation
graph; repeated edges are rejected as malformed input.

The condensation graph records direct inter-component edges only. It does not
compute transitive closure, shortest paths, a cycle witness, or an execution
schedule. Strong connectivity means mutual reachability, not that every pair
has a direct edge. The complete closed graph must fit in memory before any
result is returned.

## Related Snippets

<!-- catalog:related:start -->
- [Classify Bounded DAG Stages as Ready, Waiting, or Blocked](classify-bounded-dag-stages-as-ready-waiting-or-blocked.md)
- [Resolve Stable Ordering Constraints with Topological Sort](resolve-stable-ordering-constraints-with-topological-sort.md)
- [Traverse a Parent Graph with Breadth-First Search](traverse-a-parent-graph-with-breadth-first-search.md)
<!-- catalog:related:end -->
