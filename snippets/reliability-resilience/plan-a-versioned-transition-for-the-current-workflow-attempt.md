---
title: "Plan a Versioned Transition for the Current Workflow Attempt"
snippet_type: pattern
use_cases:
  - concurrency-control
  - lifecycle-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../storage-databases/compare-and-swap-a-versioned-sqlite-setting-with-one-conditional-update.md
  - plan-remaining-stages-from-a-validated-completed-prefix.md
  - ../concurrency-lifecycle/guard-an-async-resource-with-explicit-lifecycle-states.md
---

# Plan a Versioned Transition for the Current Workflow Attempt

## Idea and Problem

Plan one workflow state change only when an immutable snapshot still describes the current attempt and expected revision.

An explicit closed transition table defines every state and allowed directed
edge. The planner validates that bounded table, rejects stale snapshots, and
returns prior and new values without changing either the caller's snapshot or
any external state.

## When to Use

Use this pattern after reading a workflow snapshot when another writer could
replace the attempt or advance its revision before a proposed state change is
stored. It separates a deterministic transition decision from the mechanism
that applies it.

The state vocabulary and edges should be a small, reviewed lifecycle. Use a
richer state machine when guards require additional data or when a transition
must calculate domain output beyond the next state and revision.

## Implementation

```python
import re
from dataclasses import dataclass


_MAX_STATES = 16
_MAX_EDGES = 64
_MAX_REVISION = 2**63 - 1
_ATTEMPT_TOKEN = re.compile(r"[a-z0-9][a-z0-9._-]{0,63}", re.ASCII)
_STATE = re.compile(r"[a-z][a-z0-9_-]{0,31}", re.ASCII)


@dataclass(frozen=True, slots=True)
class WorkflowAttemptSnapshot:
    attempt: str
    state: str
    revision: int


@dataclass(frozen=True, slots=True)
class WorkflowTransitionTable:
    states: tuple[str, ...]
    edges: tuple[tuple[str, str], ...]


@dataclass(frozen=True, slots=True)
class WorkflowTransitionPlan:
    prior: WorkflowAttemptSnapshot
    new: WorkflowAttemptSnapshot


class StaleWorkflowAttemptError(ValueError):
    pass


def _validated_attempt(value: object, *, field: str) -> str:
    if type(value) is not str or _ATTEMPT_TOKEN.fullmatch(value) is None:
        raise ValueError(f"{field} must be a conservative attempt token")
    return value


def _validated_state(value: object, *, field: str) -> str:
    if type(value) is not str or _STATE.fullmatch(value) is None:
        raise ValueError(f"{field} must be a conservative state identifier")
    return value


def _validated_revision(value: object, *, field: str) -> int:
    if type(value) is not int:
        raise TypeError(f"{field} must be an integer")
    if not 0 <= value <= _MAX_REVISION:
        raise ValueError(f"{field} must be between 0 and {_MAX_REVISION}")
    return value


def _validated_table(table: object) -> WorkflowTransitionTable:
    if type(table) is not WorkflowTransitionTable:
        raise TypeError("table must be an exact WorkflowTransitionTable")
    if type(table.states) is not tuple:
        raise TypeError("table.states must be an exact tuple")
    if not 1 <= len(table.states) <= _MAX_STATES:
        raise ValueError("table.states must contain between 1 and 16 states")

    states: list[str] = []
    state_set: set[str] = set()
    for index, value in enumerate(table.states):
        state = _validated_state(value, field=f"table.states[{index}]")
        if state in state_set:
            raise ValueError("table.states must be unique")
        state_set.add(state)
        states.append(state)

    if type(table.edges) is not tuple:
        raise TypeError("table.edges must be an exact tuple")
    if len(table.edges) > _MAX_EDGES:
        raise ValueError("table.edges must contain at most 64 edges")

    edges: list[tuple[str, str]] = []
    edge_set: set[tuple[str, str]] = set()
    for index, value in enumerate(table.edges):
        if type(value) is not tuple or len(value) != 2:
            raise TypeError(f"table.edges[{index}] must be an exact state pair")
        prior_state = _validated_state(
            value[0],
            field=f"table.edges[{index}][0]",
        )
        next_state = _validated_state(
            value[1],
            field=f"table.edges[{index}][1]",
        )
        edge = (prior_state, next_state)
        if prior_state == next_state:
            raise ValueError("table.edges cannot contain self edges")
        if prior_state not in state_set or next_state not in state_set:
            raise ValueError("every edge endpoint must appear in table.states")
        if edge in edge_set:
            raise ValueError("table.edges must be unique")
        edge_set.add(edge)
        edges.append(edge)

    return WorkflowTransitionTable(tuple(states), tuple(edges))


def _validated_snapshot(value: object) -> WorkflowAttemptSnapshot:
    if type(value) is not WorkflowAttemptSnapshot:
        raise TypeError("snapshot must be an exact WorkflowAttemptSnapshot")
    return WorkflowAttemptSnapshot(
        attempt=_validated_attempt(value.attempt, field="snapshot.attempt"),
        state=_validated_state(value.state, field="snapshot.state"),
        revision=_validated_revision(value.revision, field="snapshot.revision"),
    )


def plan_transition_for_current_attempt(
    snapshot: WorkflowAttemptSnapshot,
    *,
    current_attempt: str,
    expected_revision: int,
    requested_state: str,
    table: WorkflowTransitionTable,
) -> WorkflowTransitionPlan:
    validated_table = _validated_table(table)
    prior = _validated_snapshot(snapshot)
    attempt = _validated_attempt(current_attempt, field="current_attempt")
    revision = _validated_revision(expected_revision, field="expected_revision")
    next_state = _validated_state(requested_state, field="requested_state")

    known_states = frozenset(validated_table.states)
    if prior.state not in known_states:
        raise ValueError("snapshot.state is not present in table.states")
    if next_state not in known_states:
        raise ValueError("requested_state is not present in table.states")
    if prior.attempt != attempt:
        raise StaleWorkflowAttemptError("snapshot is for a different attempt")
    if prior.revision != revision:
        raise StaleWorkflowAttemptError("snapshot revision is stale")
    if (prior.state, next_state) not in validated_table.edges:
        raise ValueError("the requested transition is not allowed")
    if prior.revision == _MAX_REVISION:
        raise OverflowError("snapshot revision cannot be incremented")

    return WorkflowTransitionPlan(
        prior=prior,
        new=WorkflowAttemptSnapshot(attempt, next_state, prior.revision + 1),
    )
```

## Example

```python
table = WorkflowTransitionTable(
    states=("queued", "running", "finished"),
    edges=(("queued", "running"), ("running", "finished")),
)
snapshot = WorkflowAttemptSnapshot(
    attempt="round-7",
    state="queued",
    revision=4,
)

plan = plan_transition_for_current_attempt(
    snapshot,
    current_attempt="round-7",
    expected_revision=4,
    requested_state="running",
    table=table,
)

try:
    plan_transition_for_current_attempt(
        snapshot,
        current_attempt="round-7",
        expected_revision=3,
        requested_state="running",
        table=table,
    )
except StaleWorkflowAttemptError:
    stale_rejected = True
else:
    stale_rejected = False

assert (plan, stale_rejected) == (
    WorkflowTransitionPlan(
        prior=WorkflowAttemptSnapshot("round-7", "queued", 4),
        new=WorkflowAttemptSnapshot("round-7", "running", 5),
    ),
    True,
)
```

## Trade-offs and Limitations

Validation is bounded by 16 states and 64 unique, non-self edges. Both edge
endpoints must belong to the declared state tuple, making the table closed.
Revisions use a nonnegative signed 64-bit range, and planning at its maximum
is rejected rather than wrapping.

The function performs no external mutation, persistence, clock reads, retries,
or workflow actions. A persistence layer that applies the plan must compare
both the attempt token and revision atomically in one compare-and-swap; checking
either value separately leaves a race. The plan does not resolve a conflict or
decide whether an action associated with a state is safe to repeat.

## Related Snippets

<!-- catalog:related:start -->
- [Compare and Swap a Versioned SQLite Setting with One Conditional Update](../storage-databases/compare-and-swap-a-versioned-sqlite-setting-with-one-conditional-update.md)
- [Plan Remaining Stages from a Validated Completed Prefix](plan-remaining-stages-from-a-validated-completed-prefix.md)
- [Guard an Async Resource with Explicit Lifecycle States](../concurrency-lifecycle/guard-an-async-resource-with-explicit-lifecycle-states.md)
<!-- catalog:related:end -->
