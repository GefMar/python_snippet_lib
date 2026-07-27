---
title: "Classify Bounded DAG Stages as Ready, Waiting, or Blocked"
snippet_type: algorithm
use_cases:
  - automation
  - concurrency-control
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - resolve-stable-ordering-constraints-with-topological-sort.md
  - ../testing-tooling/build-a-bounded-release-dag-around-a-manual-barrier.md
  - ../reliability-resilience/plan-remaining-stages-from-a-validated-completed-prefix.md
---

# Classify Bounded DAG Stages as Ready, Waiting, or Blocked

## Idea and Problem

Classify the remaining nodes of a small dependency DAG from one immutable lifecycle snapshot.

Graph validation happens before any result is constructed. An errored prerequisite
blocks all of its descendants, a node whose direct prerequisites are all
completed is ready, and every other unclassified node is waiting. Each output
tuple retains declaration order, independent of hash order or the graph order
used internally for validation.

## When to Use

Use this algorithm as the pure decision step for a bounded in-memory workflow
whose node definitions and lifecycle sets are already available together.
`completed`, `in_progress`, and `errored` must describe pairwise-disjoint
subsets of the declared nodes. An in-progress prerequisite does not satisfy a dependency, so
its dependent remains waiting.

This classifier describes the current snapshot only. Claiming a ready node
atomically, executing it, and publishing a later state transition belong to a
separate coordinator. Use a topological sort instead when the goal is one
complete dependency-respecting order rather than state-aware classification.

## Implementation

```python
import re
from collections import deque
from dataclasses import dataclass


_MAX_NODES = 64
_MAX_DEPENDENCIES_PER_NODE = 16
_MAX_EDGES = 256
_NODE_NAME = re.compile(r"[a-z][a-z0-9_-]{0,31}", re.ASCII)


@dataclass(frozen=True, slots=True)
class DagNode:
    name: str
    dependencies: tuple[str, ...] = ()


@dataclass(frozen=True, slots=True)
class DagClassification:
    ready: tuple[str, ...]
    waiting: tuple[str, ...]
    blocked: tuple[str, ...]


def _validated_name(value: object, *, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if _NODE_NAME.fullmatch(value) is None:
        raise ValueError(
            f"{field} must be a conservative 1-32 character ASCII name"
        )
    return value


def _validate_nodes(
    nodes: object,
) -> tuple[
    tuple[str, ...],
    dict[str, tuple[str, ...]],
    tuple[str, ...],
]:
    if type(nodes) is not tuple:
        raise TypeError("nodes must be an exact tuple")
    if not 1 <= len(nodes) <= _MAX_NODES:
        raise ValueError("nodes must contain between 1 and 64 definitions")

    ordered_names: list[str] = []
    dependencies_by_name: dict[str, tuple[str, ...]] = {}
    edge_count = 0

    for index, node in enumerate(nodes):
        if type(node) is not DagNode:
            raise TypeError(f"nodes[{index}] must be a DagNode")
        name = _validated_name(node.name, field=f"nodes[{index}].name")
        if name in dependencies_by_name:
            raise ValueError("node names must be unique")
        if type(node.dependencies) is not tuple:
            raise TypeError(
                f"dependencies for node {name!r} must be an exact tuple"
            )
        if len(node.dependencies) > _MAX_DEPENDENCIES_PER_NODE:
            raise ValueError(
                f"node {name!r} has more than 16 dependencies"
            )

        dependencies: list[str] = []
        seen_dependencies: set[str] = set()
        for dependency_index, raw_dependency in enumerate(node.dependencies):
            dependency = _validated_name(
                raw_dependency,
                field=(
                    f"nodes[{index}].dependencies[{dependency_index}]"
                ),
            )
            if dependency == name:
                raise ValueError(f"node {name!r} cannot depend on itself")
            if dependency in seen_dependencies:
                raise ValueError(
                    f"node {name!r} repeats a dependency"
                )
            seen_dependencies.add(dependency)
            dependencies.append(dependency)

        edge_count += len(dependencies)
        if edge_count > _MAX_EDGES:
            raise ValueError("the graph may contain at most 256 edges")
        ordered_names.append(name)
        dependencies_by_name[name] = tuple(dependencies)

    declared_names = frozenset(ordered_names)
    for name in ordered_names:
        for dependency in dependencies_by_name[name]:
            if dependency not in declared_names:
                raise ValueError(
                    f"node {name!r} has a missing dependency"
                )

    mutable_dependents: dict[str, list[str]] = {
        name: [] for name in ordered_names
    }
    remaining = {
        name: len(dependencies_by_name[name])
        for name in ordered_names
    }
    for name in ordered_names:
        for dependency in dependencies_by_name[name]:
            mutable_dependents[dependency].append(name)

    available = deque(
        name for name in ordered_names if remaining[name] == 0
    )
    topological_order: list[str] = []
    while available:
        name = available.popleft()
        topological_order.append(name)
        for dependent in mutable_dependents[name]:
            remaining[dependent] -= 1
            if remaining[dependent] == 0:
                available.append(dependent)

    if len(topological_order) != len(ordered_names):
        raise ValueError("node dependencies must not contain a cycle")

    return (
        tuple(ordered_names),
        dependencies_by_name,
        tuple(topological_order),
    )


def _validate_state_set(
    value: object,
    *,
    field: str,
    declared_names: frozenset[str],
) -> frozenset[str]:
    if type(value) is not frozenset:
        raise TypeError(f"{field} must be an exact frozenset")
    for name in value:
        _validated_name(name, field=f"a name in {field}")
    if not value <= declared_names:
        raise ValueError(f"{field} contains an unknown node")
    return value


def classify_dag_nodes(
    nodes: tuple[DagNode, ...],
    *,
    completed: frozenset[str],
    in_progress: frozenset[str],
    errored: frozenset[str],
) -> DagClassification:
    """Validate and classify nodes outside the three lifecycle sets."""
    (
        ordered_names,
        dependencies_by_name,
        topological_order,
    ) = _validate_nodes(nodes)
    declared_names = frozenset(ordered_names)
    completed_names = _validate_state_set(
        completed,
        field="completed",
        declared_names=declared_names,
    )
    in_progress_names = _validate_state_set(
        in_progress,
        field="in_progress",
        declared_names=declared_names,
    )
    errored_names = _validate_state_set(
        errored,
        field="errored",
        declared_names=declared_names,
    )
    if (
        completed_names & in_progress_names
        or completed_names & errored_names
        or in_progress_names & errored_names
    ):
        raise ValueError(
            "completed, in_progress, and errored must be pairwise disjoint"
        )

    blocked_by_error: set[str] = set()
    for name in topological_order:
        if name in errored_names or any(
            dependency in blocked_by_error
            for dependency in dependencies_by_name[name]
        ):
            blocked_by_error.add(name)

    excluded = completed_names | in_progress_names | errored_names
    ready: list[str] = []
    waiting: list[str] = []
    blocked: list[str] = []
    for name in ordered_names:
        if name in excluded:
            continue
        if name in blocked_by_error:
            blocked.append(name)
        elif all(
            dependency in completed_names
            for dependency in dependencies_by_name[name]
        ):
            ready.append(name)
        else:
            waiting.append(name)

    return DagClassification(
        ready=tuple(ready),
        waiting=tuple(waiting),
        blocked=tuple(blocked),
    )
```

## Example

```python
nodes = (
    DagNode("ready_node", ("completed_root",)),
    DagNode("waiting_node", ("in_progress_root",)),
    DagNode("direct_blocked", ("errored_root",)),
    DagNode("transitive_blocked", ("direct_blocked",)),
    DagNode("completed_root"),
    DagNode("in_progress_root"),
    DagNode("errored_root"),
)
classification = classify_dag_nodes(
    nodes,
    completed=frozenset({"completed_root"}),
    in_progress=frozenset({"in_progress_root"}),
    errored=frozenset({"errored_root"}),
)

try:
    classify_dag_nodes(
        (
            DagNode("left", ("right",)),
            DagNode("right", ("left",)),
        ),
        completed=frozenset(),
        in_progress=frozenset(),
        errored=frozenset(),
    )
except ValueError:
    cycle_rejected = True
else:
    cycle_rejected = False

assert (classification, cycle_rejected) == (
    DagClassification(
        ready=("ready_node",),
        waiting=("waiting_node",),
        blocked=("direct_blocked", "transitive_blocked"),
    ),
    True,
)
```

## Trade-offs and Limitations

Validation and classification take `O(V + E)` time and memory within the
64-node, 16-dependency-per-node, and 256-edge bounds. Kahn's algorithm checks
the entire graph for cycles without recursion; its internal order is not
returned. The three output tuples are separate declaration-order partitions of
only the nodes that are not completed, in progress, or errored.

Error ancestry takes precedence over readiness, including when an errored
ancestor lies beyond a direct prerequisite. Otherwise only `completed` satisfies
a direct dependency; `in_progress` does not. The classifier accepts the lifecycle
sets as one literal snapshot and does not invent additional history rules for
them.

This is not a total topological sort and does not explain which errored path
caused a block. It invokes no callback, mutates no input, and performs no node
execution, leasing, persistence, retry, rollback, or cross-process
coordination.

## Related Snippets

<!-- catalog:related:start -->
- [Resolve Stable Ordering Constraints with Topological Sort](resolve-stable-ordering-constraints-with-topological-sort.md)
- [Build a Bounded Release DAG Around a Manual Barrier](../testing-tooling/build-a-bounded-release-dag-around-a-manual-barrier.md)
- [Plan Remaining Stages from a Validated Completed Prefix](../reliability-resilience/plan-remaining-stages-from-a-validated-completed-prefix.md)
<!-- catalog:related:end -->
