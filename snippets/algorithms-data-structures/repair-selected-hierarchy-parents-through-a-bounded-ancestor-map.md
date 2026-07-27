---
title: "Repair Selected Hierarchy Parents Through a Bounded Ancestor Map"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - traverse-a-parent-graph-with-breadth-first-search.md
  - resolve-stable-ordering-constraints-with-topological-sort.md
  - rank-hierarchy-paths-with-bounded-weighted-edit-distance.md
---

# Repair Selected Hierarchy Parents Through a Bounded Ancestor Map

## Idea and Problem

Project a selected subset of a hierarchy into its own forest by replacing each original parent with the nearest selected ancestor and recomputing depth.

Filtering hierarchical records can leave parent references pointing to omitted
nodes. A closed parent map lets the projection skip those nodes without losing
ancestry. Validating every component first is important: a cycle or dangling
link outside the selected subset still means the supplied forest is not a
trustworthy input.

## When to Use

Use this approach when all selected nodes and every ancestor they can reach are
already available as immutable records. It fits export planning, tree views,
and other transformations that need a self-contained selected hierarchy while
preserving selected input order. It is appropriate only when non-negative
63-bit integer identifiers, explicit roots, and the fixed size limits below
match the surrounding data contract.

## Implementation

```python
from collections import deque
from dataclasses import dataclass


_MAX_LINKS = 10_000
_MAX_SELECTED = 10_000
_MAX_NODE_ID = (1 << 63) - 1
_MAX_WORK = 30_000


@dataclass(frozen=True, slots=True)
class ParentLink:
    node_id: int
    parent_id: int | None


@dataclass(frozen=True, slots=True)
class ProjectedNode:
    node_id: int
    parent_id: int | None
    depth: int


def _validate_node_id(value: object, field_name: str) -> int:
    if type(value) is not int:
        raise TypeError(f"{field_name} must be an exact int")
    if not 0 <= value <= _MAX_NODE_ID:
        raise ValueError(f"{field_name} is outside the supported range")
    return value


def repair_selected_hierarchy(
    parent_links: tuple[ParentLink, ...],
    selected_ids: tuple[int, ...],
) -> tuple[ProjectedNode, ...]:
    if type(parent_links) is not tuple or type(selected_ids) is not tuple:
        raise TypeError("inputs must be exact tuples")
    if len(parent_links) > _MAX_LINKS:
        raise ValueError("parent-link limit exceeded")
    if len(selected_ids) > _MAX_SELECTED:
        raise ValueError("selected-node limit exceeded")

    parents: dict[int, int | None] = {}
    children: dict[int, list[int]] = {}
    for link in parent_links:
        if type(link) is not ParentLink:
            raise TypeError("parent links must be exact ParentLink records")
        node_id = _validate_node_id(link.node_id, "node_id")
        parent_id = link.parent_id
        if parent_id is not None:
            parent_id = _validate_node_id(parent_id, "parent_id")
        if node_id in parents:
            raise ValueError("duplicate node identifier")
        parents[node_id] = parent_id
        children[node_id] = []

    selected: set[int] = set()
    for value in selected_ids:
        node_id = _validate_node_id(value, "selected_id")
        if node_id in selected:
            raise ValueError("duplicate selected identifier")
        if node_id not in parents:
            raise ValueError("unknown selected identifier")
        selected.add(node_id)

    roots: list[int] = []
    for node_id, parent_id in parents.items():
        if parent_id is None:
            roots.append(node_id)
        elif parent_id == node_id:
            raise ValueError("self-parent link")
        elif parent_id not in parents:
            raise ValueError("parent map is not closed")
        else:
            children[parent_id].append(node_id)

    nearest_selected: dict[int, int | None] = {}
    selected_depth: dict[int, int] = {}
    queue = deque(roots)
    visited = 0
    work = len(parent_links) + len(selected_ids)

    while queue:
        node_id = queue.popleft()
        parent_id = parents[node_id]
        nearest = (
            None
            if parent_id is None
            else parent_id
            if parent_id in selected
            else nearest_selected[parent_id]
        )
        nearest_selected[node_id] = nearest
        if node_id in selected:
            selected_depth[node_id] = (
                0 if nearest is None else selected_depth[nearest] + 1
            )

        visited += 1
        work += 1 + len(children[node_id])
        if work > _MAX_WORK:
            raise ValueError("hierarchy work limit exceeded")
        queue.extend(children[node_id])

    if visited != len(parent_links):
        raise ValueError("parent map contains a cycle")

    return tuple(
        ProjectedNode(
            node_id=node_id,
            parent_id=nearest_selected[node_id],
            depth=selected_depth[node_id],
        )
        for node_id in selected_ids
    )
```

## Example

```python
links = (
    ParentLink(10, None),
    ParentLink(20, 10),
    ParentLink(30, 20),
    ParentLink(40, 20),
    ParentLink(50, 30),
)
selected_ids = (30, 10, 50, 40)
snapshot = tuple((link.node_id, link.parent_id) for link in links)

projection = repair_selected_hierarchy(links, selected_ids)

cycle_rejected = False
try:
    repair_selected_hierarchy(
        links + (ParentLink(60, 70), ParentLink(70, 60)),
        (10,),
    )
except ValueError:
    cycle_rejected = True

assert (
    projection
    == (
        ProjectedNode(30, 10, 1),
        ProjectedNode(10, None, 0),
        ProjectedNode(50, 30, 2),
        ProjectedNode(40, 10, 1),
    )
    and tuple((link.node_id, link.parent_id) for link in links) == snapshot
    and cycle_rejected
)
```

## Trade-offs and Limitations

Validation and projection take `O(n)` time and `O(n)` additional memory for
the parent, child, and ancestry maps. The function requires the complete
bounded forest rather than fetching missing ancestors, and its exact-type and
identifier-range rules are intentionally strict. A large selection can hit the
combined work cap even when both tuple-size limits pass. The function preserves
selected order but removes all unselected structure, so it is not suitable when
skipped nodes or original depths must remain visible. The result is a
transformation plan; it performs no mutation, persistence, SQL generation, or
I/O.

## Related Snippets

<!-- catalog:related:start -->
- [Traverse a Parent Graph with Breadth-First Search](traverse-a-parent-graph-with-breadth-first-search.md)
- [Resolve Stable Ordering Constraints with Topological Sort](resolve-stable-ordering-constraints-with-topological-sort.md)
- [Rank Hierarchy Paths with Bounded Weighted Edit Distance](rank-hierarchy-paths-with-bounded-weighted-edit-distance.md)
<!-- catalog:related:end -->
