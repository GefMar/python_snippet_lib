---
title: "Build a Bounded Release DAG Around a Manual Barrier"
snippet_type: pattern
use_cases:
  - automation
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../algorithms-data-structures/resolve-stable-ordering-constraints-with-topological-sort.md
---

# Build a Bounded Release DAG Around a Manual Barrier

## Idea and Problem

Build a deterministic prepare-barrier-release graph in which one explicit manual decision separates all preparation work from every release step.

The graph uses structured keys instead of deriving names by concatenating or
slugging work identifiers. For `n` identifiers, it always contains `2n + 1`
nodes and `2n` dependency edges: each prepare node is independent, the barrier
depends on every prepare node, and each release node depends only on the
barrier.

## When to Use

Use this pattern when several bounded preparations may run independently, but
no corresponding release may begin until all preparations finish and one
operator approves the transition. The caller must preserve the returned keys
when adapting the plan to another system; string serialization and external
job names belong at that boundary.

This shape is useful only when every release shares the same approval decision.
Use a more general graph model when work items have additional dependencies,
separate approval groups, retries, or conditional branches.

## Implementation

```python
from dataclasses import dataclass
from enum import StrEnum


_MAX_WORK_ITEMS = 256
_MAX_WORK_ID_BYTES = 64


class NodeKind(StrEnum):
    PREPARE = "prepare"
    MANUAL_BARRIER = "manual_barrier"
    RELEASE = "release"


@dataclass(frozen=True, slots=True)
class NodeKey:
    kind: NodeKind
    work_id: str | None


@dataclass(frozen=True, slots=True)
class PlannedNode:
    key: NodeKey
    dependencies: tuple[NodeKey, ...]
    requires_manual_approval: bool


@dataclass(frozen=True, slots=True)
class ReleaseDAG:
    nodes: tuple[PlannedNode, ...]


def _validate_work_ids(work_ids: object) -> tuple[str, ...]:
    if type(work_ids) is not tuple:
        raise TypeError("work_ids must be an exact tuple")
    if not 1 <= len(work_ids) <= _MAX_WORK_ITEMS:
        raise ValueError("work_ids must contain between 1 and 256 items")

    validated: list[str] = []
    seen: set[str] = set()
    for work_id in work_ids:
        if type(work_id) is not str:
            raise TypeError("each work ID must be exact text")
        if not 1 <= len(work_id) <= _MAX_WORK_ID_BYTES:
            raise ValueError("each work ID must contain between 1 and 64 bytes")
        if any(not 33 <= ord(character) <= 126 for character in work_id):
            raise ValueError("work IDs must use non-whitespace printable ASCII")
        if work_id in seen:
            raise ValueError(f"duplicate work ID: {work_id!r}")
        seen.add(work_id)
        validated.append(work_id)
    return tuple(validated)


def build_release_dag(work_ids: tuple[str, ...]) -> ReleaseDAG:
    identifiers = _validate_work_ids(work_ids)
    prepare_keys = tuple(
        NodeKey(NodeKind.PREPARE, work_id)
        for work_id in identifiers
    )
    barrier_key = NodeKey(NodeKind.MANUAL_BARRIER, None)

    prepare_nodes = tuple(
        PlannedNode(
            key=key,
            dependencies=(),
            requires_manual_approval=False,
        )
        for key in prepare_keys
    )
    barrier_node = PlannedNode(
        key=barrier_key,
        dependencies=prepare_keys,
        requires_manual_approval=True,
    )
    release_nodes = tuple(
        PlannedNode(
            key=NodeKey(NodeKind.RELEASE, work_id),
            dependencies=(barrier_key,),
            requires_manual_approval=False,
        )
        for work_id in identifiers
    )
    return ReleaseDAG(prepare_nodes + (barrier_node,) + release_nodes)
```

## Example

```python
plan = build_release_dag(("manual", "manual-release"))
barrier = plan.nodes[2]

try:
    build_release_dag(("api", "api"))
except ValueError:
    duplicate_rejected = True
else:
    duplicate_rejected = False

try:
    build_release_dag(("contains space",))
except ValueError:
    unsafe_text_rejected = True
else:
    unsafe_text_rejected = False

assert (
    len(plan.nodes),
    sum(len(node.dependencies) for node in plan.nodes),
    tuple(node.key.kind for node in plan.nodes),
    barrier.requires_manual_approval,
    barrier.dependencies,
    plan.nodes[-1].dependencies,
    duplicate_rejected,
    unsafe_text_rejected,
) == (
    5,
    4,
    (
        NodeKind.PREPARE,
        NodeKind.PREPARE,
        NodeKind.MANUAL_BARRIER,
        NodeKind.RELEASE,
        NodeKind.RELEASE,
    ),
    True,
    (
        NodeKey(NodeKind.PREPARE, "manual"),
        NodeKey(NodeKind.PREPARE, "manual-release"),
    ),
    (NodeKey(NodeKind.MANUAL_BARRIER, None),),
    True,
    True,
)
```

## Trade-offs and Limitations

Construction uses `O(n)` time and memory, with at most 513 nodes and 512 edges.
Input order determines node and dependency order, while structured keys keep
identifiers such as `manual` and `manual-release` distinct without relying on a
lossy slug. The returned values are frozen, but an adapter must still enforce
its target system's naming and size rules.

This helper does not execute work, persist approval, generate configuration,
perform retries, or prove that preparation actually completed. A single barrier
also creates one coordination point; independent release groups should use a
different graph contract rather than modifying this result after construction.

## Related Snippets

<!-- catalog:related:start -->
- [Resolve Stable Ordering Constraints with Topological Sort](../algorithms-data-structures/resolve-stable-ordering-constraints-with-topological-sort.md)
<!-- catalog:related:end -->
