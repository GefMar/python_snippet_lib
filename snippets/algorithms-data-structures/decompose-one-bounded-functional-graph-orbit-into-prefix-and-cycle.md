---
title: "Decompose One Bounded Functional-Graph Orbit into Prefix and Cycle"
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
  - find-one-deterministic-shortest-path-in-a-bounded-non-negatively-weighted-directed-graph.md
---

# Decompose One Bounded Functional-Graph Orbit into Prefix and Cycle

## Idea and Problem

Follow one node in a closed functional graph and split its deterministic orbit into a non-repeating prefix and a non-empty repeating cycle.

Every node has exactly one successor. Record the walk position where each
visited node first appears, then stop when the next node is already present.
That stored position separates the transient prefix from the cycle, whose first
element is the first repeated node.

## When to Use

Use this algorithm when a bounded state space has one total deterministic next
state per node and the behavior from one start must be described explicitly.
It fits finite-state simulations, successor tables, and deterministic transition
analysis where both the transient path and eventual loop matter.

Use a general graph algorithm when nodes may have zero or several outgoing
edges, and build a shared index when many start nodes must be queried. The
result covers only the chosen start's orbit, not every component in the graph.

## Implementation

```python
from dataclasses import dataclass

_MAX_NODE_COUNT = 10_000


@dataclass(frozen=True, slots=True)
class FunctionalOrbit:
    prefix: tuple[int, ...]
    cycle: tuple[int, ...]


def decompose_functional_orbit(
    successors: tuple[int, ...],
    start: int,
) -> FunctionalOrbit:
    """Return the prefix and cycle reachable from one validated start node."""
    if type(successors) is not tuple:
        raise TypeError("successors must be an exact tuple")
    if not 1 <= len(successors) <= _MAX_NODE_COUNT:
        raise ValueError("node count is outside the supported range")

    node_count = len(successors)
    for node, successor in enumerate(successors):
        if type(successor) is not int:
            raise TypeError(f"successors[{node}] must be an exact integer")
        if not 0 <= successor < node_count:
            raise ValueError(f"successors[{node}] is outside the closed graph")

    if type(start) is not int:
        raise TypeError("start must be an exact integer")
    if not 0 <= start < node_count:
        raise ValueError("start is outside the closed graph")

    first_position: dict[int, int] = {}
    walk: list[int] = []
    node = start

    while node not in first_position:
        first_position[node] = len(walk)
        walk.append(node)
        node = successors[node]

    cycle_start = first_position[node]
    return FunctionalOrbit(
        prefix=tuple(walk[:cycle_start]),
        cycle=tuple(walk[cycle_start:]),
    )
```

## Example

```python
successors = (1, 2, 3, 1, 5, 4)
from_zero = decompose_functional_orbit(successors, 0)
from_four = decompose_functional_orbit(successors, 4)

try:
    decompose_functional_orbit(successors, False)
except TypeError:
    bool_start_rejected = True
else:
    bool_start_rejected = False

assert (from_zero, from_four, bool_start_rejected) == (
    FunctionalOrbit(prefix=(0,), cycle=(1, 2, 3)),
    FunctionalOrbit(prefix=(), cycle=(4, 5)),
    True,
)
```

## Trade-offs and Limitations

Validating the complete closed graph takes `O(n)` time even when the selected
orbit is short. Walking takes `O(prefix length + cycle length)` time and memory;
the visited-position mapping makes the split direct and avoids dependence on
recursion limits. The returned tuples are independent immutable containers.

The cycle is always non-empty and begins at the first node encountered twice;
a self-loop therefore has an empty prefix and a one-node cycle. Unreachable
nodes are validated but omitted. Node identities are tuple indexes, and the
function does not partition every component, combine multiple starts, accept a
partial successor relation, update the graph, or provide constant-memory cycle
detection.

## Related Snippets

<!-- catalog:related:start -->
- [Partition a Bounded Directed Graph into Deterministic Strongly Connected Components](partition-a-bounded-directed-graph-into-deterministic-strongly-connected-components.md)
- [Traverse a Parent Graph with Breadth-First Search](traverse-a-parent-graph-with-breadth-first-search.md)
- [Find One Deterministic Shortest Path in a Bounded Non-Negatively Weighted Directed Graph](find-one-deterministic-shortest-path-in-a-bounded-non-negatively-weighted-directed-graph.md)
<!-- catalog:related:end -->
