---
title: "Traverse a Parent Graph with Breadth-First Search"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../python-language/walk-a-tree-recursively-with-yield-from.md
---

# Traverse a Parent Graph with Breadth-First Search

## Idea and Problem

Use a queue and a visited set to yield each reachable parent except the starting node once in stable breadth-first order, even when the graph contains cycles.

Parent relationships can form a directed graph rather than a tree: several
paths may reach the same node, and malformed input can contain a cycle. Marking
each identifier when it enters the queue prevents duplicate results and
non-terminating traversal.

## When to Use

Use this algorithm when all reachable ancestors other than the starting node
are needed, nearer parents must appear before more distant parents, and node
identifiers are hashable.
Parent iterables must have deterministic order if stable output matters. A
referenced identifier that has no entry in the mapping is treated as a terminal
node and is still yielded.

## Implementation

```python
from collections import deque
from collections.abc import Hashable, Iterable, Iterator, Mapping
from typing import TypeVar


NodeId = TypeVar("NodeId", bound=Hashable)


def iter_ancestors(
    start: NodeId,
    parents_by_node: Mapping[NodeId, Iterable[NodeId]],
) -> Iterator[NodeId]:
    queue = deque([start])
    seen = {start}

    while queue:
        node_id = queue.popleft()
        for parent_id in parents_by_node.get(node_id, ()):
            if parent_id in seen:
                continue
            seen.add(parent_id)
            queue.append(parent_id)
            yield parent_id
```

## Example

```python
parents = {
    "leaf": ("left", "right"),
    "left": ("root",),
    "right": ("root", "external"),
    "root": ("leaf",),
}

ancestors = list(iter_ancestors("leaf", parents))

assert ancestors == ["left", "right", "root", "external"]
```

## Trade-offs and Limitations

The visited set and queue require memory proportional to the reachable graph.
The starting identifier is never yielded, even if a cycle leads back to it.
The function yields all other reachable parents, not only the nearest match,
and it does not calculate paths or distances. Output order among nodes at the
same depth follows the order of the supplied parent iterables; unordered sets
make that order unstable. Treating missing mapping entries as terminal nodes is
a contract choice, so applications that require a closed graph should validate
the mapping before traversal.

## Related Snippets

<!-- catalog:related:start -->
- [Walk a Tree Recursively with yield from](../python-language/walk-a-tree-recursively-with-yield-from.md)
<!-- catalog:related:end -->
