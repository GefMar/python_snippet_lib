---
title: "Check a Bounded Complete History for Linearizability Against a Deterministic Model"
snippet_type: testing-technique
use_cases:
  - concurrency-control
  - testing
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - find-a-shortest-invariant-violation-in-a-bounded-deterministic-state-model.md
  - enumerate-every-topological-order-of-a-tiny-dag-for-schedule-tests.md
  - ../storage-databases/audit-a-bounded-transaction-schedule-for-conflict-serializability.md
---

# Check a Bounded Complete History for Linearizability Against a Deterministic Model

## Idea and Problem

Decide whether one small completed concurrent history can be explained by a legal sequential execution that respects real-time precedence.

Operations whose time intervals overlap may be placed in either order. An
operation that finishes before or exactly when another starts must come first.
Depth-first search tries currently eligible records by input index and applies
each to a trusted deterministic state model. Matching a model result against
the recorded result makes that sequential prefix feasible.

Different prefixes can reach the same completed-operation mask and behavioral
state. Memoizing only fully explored failures avoids repeating their suffix
search without hiding a possible witness. A separate admission budget makes
state-space truncation explicit instead of misreporting it as a proof of
nonlinearizability.

## When to Use

Use this checker for a tiny, fully completed operation history and a compact
pure reference model, such as a register, counter, queue, or bounded state
machine. It is useful in deterministic concurrency tests when a Boolean answer
is insufficient: a positive result includes the lexicographically first valid
sequence of original record indexes, and budget exhaustion is distinguishable
from exhaustive rejection.

Treat the callback and its states as trusted test code. State hash and equality
must be stable behavioral identity, and the callback must be pure,
deterministic, quick, and independent of the candidate prefix. Use a dedicated
linearizability tool for pending calls, large histories, richer return values,
randomized exploration, or production-scale stress traces.

## Implementation

```python
import re
from collections.abc import Callable
from dataclasses import dataclass
from enum import StrEnum

_MAX_OPERATIONS = 10
_MAX_UNIQUE_STATES = 100_000
_MIN_I64 = -(1 << 63)
_MAX_I64 = (1 << 63) - 1
_NAME = re.compile(r"[A-Za-z][A-Za-z0-9_.-]{0,63}", re.ASCII).fullmatch

StateModel = Callable[
    [object, str, int | None],
    tuple[object, int | None],
]


class LinearizabilityStatus(StrEnum):
    LINEARIZABLE = "LINEARIZABLE"
    NONLINEARIZABLE = "NONLINEARIZABLE"
    INCONCLUSIVE = "INCONCLUSIVE"


@dataclass(frozen=True, slots=True)
class CompletedOperation:
    name: str
    argument: int | None
    observed_result: int | None
    started: int
    finished: int


@dataclass(frozen=True, slots=True)
class LinearizabilityCheck:
    status: LinearizabilityStatus
    linearization: tuple[int, ...] | None
    admitted_unique_states: int


def _validate_optional_i64(value: object, *, field: str) -> int | None:
    if value is None:
        return None
    if type(value) is not int:
        raise TypeError(f"{field} must be an exact integer or None")
    if not _MIN_I64 <= value <= _MAX_I64:
        raise ValueError(f"{field} is outside the signed 64-bit range")
    return value


def _require_hashable(state: object, *, field: str) -> None:
    try:
        hash(state)
    except TypeError as error:
        raise TypeError(f"{field} must be hashable") from error


def _apply_model(
    model: StateModel,
    state: object,
    operation: CompletedOperation,
) -> tuple[object, int | None]:
    step = model(state, operation.name, operation.argument)
    if type(step) is not tuple:
        raise TypeError("model must return an exact tuple")
    if len(step) != 2:
        raise ValueError("model must return exactly two values")

    next_state, raw_result = step
    _require_hashable(next_state, field="model next state")
    model_result = _validate_optional_i64(raw_result, field="model result")
    return next_state, model_result


def check_history_linearizability(
    history: tuple[CompletedOperation, ...],
    initial_state: object,
    model: StateModel,
    *,
    max_unique_states: int,
) -> LinearizabilityCheck:
    """Return a witness, an exhaustive rejection, or a bounded-search result."""
    if type(history) is not tuple:
        raise TypeError("history must be an exact tuple")
    if not 1 <= len(history) <= _MAX_OPERATIONS:
        raise ValueError("history length is outside 1..10")
    if not callable(model):
        raise TypeError("model must be callable")
    if type(max_unique_states) is not int:
        raise TypeError("max_unique_states must be an exact integer")
    if not 1 <= max_unique_states <= _MAX_UNIQUE_STATES:
        raise ValueError("max_unique_states is outside 1..100000")

    for index, operation in enumerate(history):
        if type(operation) is not CompletedOperation:
            raise TypeError(f"history[{index}] must be an exact CompletedOperation")
        if type(operation.name) is not str:
            raise TypeError(f"history[{index}].name must be an exact string")
        if _NAME(operation.name) is None:
            raise ValueError(f"history[{index}].name is not a valid ASCII identifier")
        _validate_optional_i64(
            operation.argument,
            field=f"history[{index}].argument",
        )
        _validate_optional_i64(
            operation.observed_result,
            field=f"history[{index}].observed_result",
        )
        if type(operation.started) is not int:
            raise TypeError(f"history[{index}].started must be an exact integer")
        if type(operation.finished) is not int:
            raise TypeError(f"history[{index}].finished must be an exact integer")
        if not 0 <= operation.started < operation.finished <= _MAX_I64:
            raise ValueError(f"history[{index}] has an invalid time interval")

    _require_hashable(initial_state, field="initial_state")

    predecessor_masks = tuple(
        sum(
            1 << before_index
            for before_index, before in enumerate(history)
            if before_index != operation_index and before.finished <= operation.started
        )
        for operation_index, operation in enumerate(history)
    )
    full_mask = (1 << len(history)) - 1
    root = (0, initial_state)
    admitted: set[tuple[int, object]] = {root}
    dead: set[tuple[int, object]] = set()

    def visit(
        mask: int,
        state: object,
        prefix: tuple[int, ...],
    ) -> tuple[int, ...] | LinearizabilityStatus | None:
        if mask == full_mask:
            return prefix

        key = (mask, state)
        if key in dead:
            return None

        remaining_mask = full_mask ^ mask
        for operation_index, operation in enumerate(history):
            operation_bit = 1 << operation_index
            if not remaining_mask & operation_bit:
                continue
            if predecessor_masks[operation_index] & remaining_mask:
                continue

            next_state, model_result = _apply_model(model, state, operation)
            if (
                type(model_result) is not type(operation.observed_result)
                or model_result != operation.observed_result
            ):
                continue

            next_mask = mask | operation_bit
            next_key = (next_mask, next_state)
            if next_key not in admitted:
                if len(admitted) >= max_unique_states:
                    return LinearizabilityStatus.INCONCLUSIVE
                admitted.add(next_key)

            outcome = visit(
                next_mask,
                next_state,
                (*prefix, operation_index),
            )
            if outcome is LinearizabilityStatus.INCONCLUSIVE:
                return outcome
            if type(outcome) is tuple:
                return outcome

        dead.add(key)
        return None

    outcome = visit(0, initial_state, ())
    if outcome is LinearizabilityStatus.INCONCLUSIVE:
        return LinearizabilityCheck(
            LinearizabilityStatus.INCONCLUSIVE,
            None,
            len(admitted),
        )
    if type(outcome) is tuple:
        return LinearizabilityCheck(
            LinearizabilityStatus.LINEARIZABLE,
            outcome,
            len(admitted),
        )
    return LinearizabilityCheck(
        LinearizabilityStatus.NONLINEARIZABLE,
        None,
        len(admitted),
    )
```

## Example

```python

def register_model(
    state: object,
    name: str,
    argument: int | None,
) -> tuple[object, int | None]:
    if type(state) is not int:
        raise TypeError("register state must be an exact integer")
    if name == "write":
        if argument is None:
            raise ValueError("write requires an argument")
        return argument, None
    if name == "read":
        if argument is not None:
            raise ValueError("read does not accept an argument")
        return state, state
    raise ValueError("unknown operation")


def permutation_oracle(
    history: tuple[CompletedOperation, ...],
    initial_state: object,
    model: StateModel,
) -> tuple[int, ...] | None:
    from itertools import permutations

    for order in permutations(range(len(history))):
        positions = {operation_index: rank for rank, operation_index in enumerate(order)}
        if any(
            positions[before_index] >= positions[after_index]
            for before_index, before in enumerate(history)
            for after_index, after in enumerate(history)
            if before_index != after_index and before.finished <= after.started
        ):
            continue

        state = initial_state
        for operation_index in order:
            operation = history[operation_index]
            state, result = model(state, operation.name, operation.argument)
            if (
                type(result) is not type(operation.observed_result)
                or result != operation.observed_result
            ):
                break
        else:
            return order
    return None


linearizable_history = (
    CompletedOperation("read", None, 1, 1, 4),
    CompletedOperation("write", 1, None, 0, 3),
)
nonlinearizable_history = (
    CompletedOperation("write", 1, None, 0, 2),
    CompletedOperation("read", None, 0, 2, 3),
)

positive = check_history_linearizability(
    linearizable_history,
    0,
    register_model,
    max_unique_states=32,
)
negative = check_history_linearizability(
    nonlinearizable_history,
    0,
    register_model,
    max_unique_states=32,
)
limited = check_history_linearizability(
    linearizable_history,
    0,
    register_model,
    max_unique_states=1,
)


def exercise_tiny_histories() -> int:
    checked = 0
    for observed in range(4):
        history = (
            CompletedOperation("write", 1, None, 0, 4),
            CompletedOperation("write", 2, None, 1, 5),
            CompletedOperation("read", None, observed, 2, 6),
        )
        expected = permutation_oracle(history, 0, register_model)
        actual = check_history_linearizability(
            history,
            0,
            register_model,
            max_unique_states=128,
        )
        if expected is None:
            assert actual.status is LinearizabilityStatus.NONLINEARIZABLE
            assert actual.linearization is None
        else:
            assert actual.status is LinearizabilityStatus.LINEARIZABLE
            assert actual.linearization == expected
        checked += 1
    return checked


def malformed_model(
    state: object,
    name: str,
    argument: int | None,
) -> object:
    return [state, None]


def unhashable_state_model(
    state: object,
    name: str,
    argument: int | None,
) -> tuple[object, int | None]:
    return [], None


def boolean_result_model(
    state: object,
    name: str,
    argument: int | None,
) -> tuple[object, object]:
    return state, False


class ModelFailure(RuntimeError):
    pass


def failing_model(
    state: object,
    name: str,
    argument: int | None,
) -> tuple[object, int | None]:
    raise ModelFailure("model failure")


def raises(expected: type[BaseException], callback: Callable[[], object]) -> bool:
    try:
        callback()
    except expected:
        return True
    return False


one_write = (CompletedOperation("write", 1, None, 0, 1),)
invalid_checks = (
    raises(
        ValueError,
        lambda: check_history_linearizability(
            (CompletedOperation("bad name", None, None, 0, 1),),
            0,
            register_model,
            max_unique_states=8,
        ),
    ),
    raises(
        TypeError,
        lambda: check_history_linearizability(
            (CompletedOperation("write", True, None, 0, 1),),
            0,
            register_model,
            max_unique_states=8,
        ),
    ),
    raises(
        TypeError,
        lambda: check_history_linearizability(
            (CompletedOperation("write", 1, None, False, 1),),
            0,
            register_model,
            max_unique_states=8,
        ),
    ),
    raises(
        TypeError,
        lambda: check_history_linearizability(
            one_write,
            0,
            malformed_model,
            max_unique_states=8,
        ),
    ),
    raises(
        TypeError,
        lambda: check_history_linearizability(
            one_write,
            0,
            unhashable_state_model,
            max_unique_states=8,
        ),
    ),
    raises(
        TypeError,
        lambda: check_history_linearizability(
            one_write,
            [],
            register_model,
            max_unique_states=8,
        ),
    ),
    raises(
        TypeError,
        lambda: check_history_linearizability(
            one_write,
            0,
            boolean_result_model,
            max_unique_states=8,
        ),
    ),
    raises(
        ModelFailure,
        lambda: check_history_linearizability(
            one_write,
            0,
            failing_model,
            max_unique_states=8,
        ),
    ),
)

assert (
    positive,
    permutation_oracle(linearizable_history, 0, register_model),
    negative,
    permutation_oracle(nonlinearizable_history, 0, register_model),
    limited,
    exercise_tiny_histories(),
    invalid_checks,
) == (
    LinearizabilityCheck(LinearizabilityStatus.LINEARIZABLE, (1, 0), 3),
    (1, 0),
    LinearizabilityCheck(LinearizabilityStatus.NONLINEARIZABLE, None, 2),
    None,
    LinearizabilityCheck(LinearizabilityStatus.INCONCLUSIVE, None, 1),
    4,
    (True,) * 8,
)
```

## Trade-offs and Limitations

For `N` completed operations and `B` admitted unique `(mask, state)` keys,
precedence construction takes `O(N**2)` time. The search performs at most
`O(B * N)` eligibility checks and model calls and retains `O(B)` keys, excluding
the callback's cost and the bit complexity of state hashing and equality. The
fixed limits are `N <= 10` and `B <= 100,000`; recursion depth cannot exceed
ten.

The root counts as the first admitted key. Before admitting key `B + 1`, the
whole search returns `INCONCLUSIVE`, even if a different unexplored branch might
contain a witness. A key enters the dead set only after every permitted suffix
has failed, so `NONLINEARIZABLE` means exhaustive failure under the model and
not budget truncation. Increasing the budget can change `INCONCLUSIVE` to
either conclusive status, but cannot invalidate an existing witness.

This checker models one complete history of successful calls with optional
signed-64 integer arguments and results. It does not support pending
operations, exceptions as recorded outcomes, nondeterministic or effectful
models, mutable state identity, weak-memory semantics, streaming histories, or
partial-order reduction. A positive bounded example does not prove that an
implementation is thread-safe, while a negative result is meaningful only to
the extent that the supplied sequential model is correct.

## Related Snippets

<!-- catalog:related:start -->
- [Find a Shortest Invariant Violation in a Bounded Deterministic State Model](find-a-shortest-invariant-violation-in-a-bounded-deterministic-state-model.md)
- [Enumerate Every Topological Order of a Tiny DAG for Schedule Tests](enumerate-every-topological-order-of-a-tiny-dag-for-schedule-tests.md)
- [Audit a Bounded Transaction Schedule for Conflict Serializability](../storage-databases/audit-a-bounded-transaction-schedule-for-conflict-serializability.md)
<!-- catalog:related:end -->
