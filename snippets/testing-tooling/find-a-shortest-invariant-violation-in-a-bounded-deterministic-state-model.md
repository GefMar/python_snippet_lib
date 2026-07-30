---
title: "Find a Shortest Invariant Violation in a Bounded Deterministic State Model"
snippet_type: testing-technique
use_cases:
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - shrink-a-bounded-failing-sequence-to-a-one-deletion-minimal-subsequence.md
  - enumerate-every-topological-order-of-a-tiny-dag-for-schedule-tests.md
  - ../algorithms-data-structures/traverse-a-parent-graph-with-breadth-first-search.md
---

# Find a Shortest Invariant Violation in a Bounded Deterministic State Model

## Idea and Problem

Explore a small deterministic state model in breadth-first order and return a shortest action trace whose resulting state violates an invariant.

Action declaration order resolves equal-length ties. States are marked seen
when admitted, so ordinary FIFO traversal makes the first violating trace the
lexicographically smallest declaration-index tuple among all shortest traces.
Predecessor links retain that witness without copying a complete trace into
every queued item.

## When to Use

Use this as a focused model-based test when state is a short integer tuple and
each action deterministically produces one next state or declares itself
inapplicable. It fits tiny workflow, protocol, allocator, and retry-policy
models whose important behavior can be expressed without clocks, I/O, or
randomness.

Keep actions and the invariant pure, deterministic, quick, and trusted. The
fixed depth and state limits make an incomplete search explicit; they do not
prove a large or unbounded system safe.

## Implementation

```python
from collections import deque
from collections.abc import Callable
from dataclasses import dataclass
from enum import StrEnum

State = tuple[int, ...]
Action = Callable[[State], State | None]
Invariant = Callable[[State], bool]

_MAX_ACTIONS = 16
_MAX_DEPTH = 32
_MAX_STATE_COUNT = 50_000
_MAX_STATE_WIDTH = 16
_MIN_INTEGER = -(1 << 63)
_MAX_INTEGER = (1 << 63) - 1


class InvariantSearchStatus(StrEnum):
    VIOLATION = "VIOLATION"
    SAFE = "SAFE"
    DEPTH_TRUNCATED = "DEPTH_TRUNCATED"
    STATE_BUDGET_EXHAUSTED = "STATE_BUDGET_EXHAUSTED"


@dataclass(frozen=True, slots=True)
class InvariantSearchResult:
    status: InvariantSearchStatus
    admitted_state_count: int
    violating_state: State | None
    action_indexes: tuple[int, ...] | None


def _validate_state(
    state: object,
    *,
    expected_width: int | None,
    field: str,
) -> State:
    if type(state) is not tuple:
        raise TypeError(f"{field} must be an exact tuple")
    if expected_width is None:
        if not 1 <= len(state) <= _MAX_STATE_WIDTH:
            raise ValueError(f"{field} width is outside the supported range")
    elif len(state) != expected_width:
        raise ValueError(f"{field} must preserve the state width")

    for coordinate in state:
        if type(coordinate) is not int:
            raise TypeError(f"{field} coordinates must be exact integers")
        if not _MIN_INTEGER <= coordinate <= _MAX_INTEGER:
            raise ValueError(f"{field} contains an out-of-range integer")
    return state


def _invariant_holds(invariant: Invariant, state: State) -> bool:
    outcome = invariant(state)
    if type(outcome) is not bool:
        raise TypeError("invariant must return an exact bool")
    return outcome


def _reconstruct_action_indexes(
    state: State,
    predecessors: dict[State, tuple[State, int] | None],
) -> tuple[int, ...]:
    reversed_indexes: list[int] = []
    link = predecessors[state]
    while link is not None:
        state, action_index = link
        reversed_indexes.append(action_index)
        link = predecessors[state]
    return tuple(reversed(reversed_indexes))


def find_shortest_invariant_violation(
    initial_state: State,
    actions: tuple[Action, ...],
    invariant: Invariant,
) -> InvariantSearchResult:
    initial = _validate_state(
        initial_state,
        expected_width=None,
        field="initial_state",
    )
    if type(actions) is not tuple:
        raise TypeError("actions must be an exact tuple")
    if not 1 <= len(actions) <= _MAX_ACTIONS:
        raise ValueError("action count is outside the supported range")
    for action in actions:
        if not callable(action):
            raise TypeError("each action must be callable")
    if not callable(invariant):
        raise TypeError("invariant must be callable")

    predecessors: dict[State, tuple[State, int] | None] = {initial: None}
    if not _invariant_holds(invariant, initial):
        return InvariantSearchResult(
            InvariantSearchStatus.VIOLATION,
            1,
            initial,
            (),
        )

    frontier = deque([(initial, 0)])
    reached_depth_limit = False
    while frontier:
        state, depth = frontier.popleft()
        for action_index, action in enumerate(actions):
            candidate = action(state)
            if candidate is None:
                continue
            successor = _validate_state(
                candidate,
                expected_width=len(initial),
                field=f"actions[{action_index}] result",
            )
            if successor in predecessors:
                continue
            if len(predecessors) == _MAX_STATE_COUNT:
                return InvariantSearchResult(
                    InvariantSearchStatus.STATE_BUDGET_EXHAUSTED,
                    len(predecessors),
                    None,
                    None,
                )

            predecessors[successor] = (state, action_index)
            if not _invariant_holds(invariant, successor):
                return InvariantSearchResult(
                    InvariantSearchStatus.VIOLATION,
                    len(predecessors),
                    successor,
                    _reconstruct_action_indexes(successor, predecessors),
                )

            successor_depth = depth + 1
            if successor_depth == _MAX_DEPTH:
                reached_depth_limit = True
            else:
                frontier.append((successor, successor_depth))

    status = (
        InvariantSearchStatus.DEPTH_TRUNCATED if reached_depth_limit else InvariantSearchStatus.SAFE
    )
    return InvariantSearchResult(status, len(predecessors), None, None)


```

## Example

```python
def add_one(state: State) -> State | None:
    return (state[0] + 1,) if state[0] < 3 else None


def add_two(state: State) -> State | None:
    return (state[0] + 2,) if state[0] < 3 else None


result = find_shortest_invariant_violation(
    (0,),
    (add_one, add_two),
    lambda state: state[0] < 3,
)

assert result == InvariantSearchResult(
    status=InvariantSearchStatus.VIOLATION,
    admitted_state_count=4,
    violating_state=(3,),
    action_indexes=(0, 1),
)
```

## Trade-offs and Limitations

At most 50,000 unique states are admitted, including the initial state, and at
most 16 actions are called for each state below depth 32. Excluding callback
cost, tuple hashing and validation make the worst-case time
`O(states * actions * state_width)`; retained states and predecessor links use
`O(states * state_width)` memory. A candidate that would become state 50,001
ends the search before its invariant or any later action is called.

A non-violating state admitted at depth 32 makes the eventual result
`DEPTH_TRUNCATED`, but the remaining shallower frontier is still explored so a
later violation or state-budget exhaustion takes precedence. Actions are never
called from depth-32 states. `SAFE` therefore means only that the reachable
model closed within both fixed limits. Callback exceptions, malformed states,
and non-boolean invariant results propagate; no partial result is returned.
Callbacks are not sandboxed, and nondeterministic or effectful callbacks
invalidate the ordering and safety interpretation.

## Related Snippets

<!-- catalog:related:start -->
- [Shrink a Bounded Failing Sequence to a One-Deletion-Minimal Subsequence](shrink-a-bounded-failing-sequence-to-a-one-deletion-minimal-subsequence.md)
- [Enumerate Every Topological Order of a Tiny DAG for Schedule Tests](enumerate-every-topological-order-of-a-tiny-dag-for-schedule-tests.md)
- [Traverse a Parent Graph with Breadth-First Search](../algorithms-data-structures/traverse-a-parent-graph-with-breadth-first-search.md)
<!-- catalog:related:end -->
