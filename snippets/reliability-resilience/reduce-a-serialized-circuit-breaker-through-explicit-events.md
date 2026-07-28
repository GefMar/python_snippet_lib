---
title: "Reduce a Serialized Circuit Breaker Through Explicit Events"
snippet_type: pattern
use_cases:
  - lifecycle-management
  - retry-recovery
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - hold-a-switch-active-through-a-monotonic-cooldown.md
  - plan-readiness-recovery-through-a-monotonic-reset-cooldown.md
  - poll-a-remote-operation-within-deadline-and-failure-budgets.md
---

# Reduce a Serialized Circuit Breaker Through Explicit Events

## Idea and Problem

Represent one serialized, single-flight circuit breaker as validated frozen state and reduce explicit request and outcome events without owning a clock or invoking an operation.

Closed requests reserve the sole operation. Consecutive failures eventually
open the breaker, open requests remain denied until a declared tick, and the
first request at or after that tick becomes the single half-open probe. Because
the reducer is pure, invalid state, time regression, and deadline overflow are
rejected without partially changing caller-owned state.

## When to Use

Use this reducer when one caller already serializes operation admission and
completion, can supply nondecreasing ticks from one integer time domain, and
wants state-machine policy separated from I/O. The explicit reservation makes
it suitable only when every admitted request receives one success or failure
event before another operation is admitted.

Use a concurrency-aware circuit-breaker implementation when several operations
may overlap. Keep retries, timeouts, exception classification, persistence,
metrics, and the operation itself in the caller rather than hiding those
effects inside this transition function.

## Implementation

```python
from dataclasses import dataclass
from enum import Enum

_MAX_SIGNED_64 = (1 << 63) - 1
_MAX_FAILURE_THRESHOLD = 64


class CircuitPhase(Enum):
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half-open"


class CircuitEvent(Enum):
    REQUEST = "request"
    SUCCESS = "success"
    FAILURE = "failure"


@dataclass(frozen=True, slots=True)
class CircuitPolicy:
    failure_threshold: int
    open_duration: int


@dataclass(frozen=True, slots=True)
class CircuitState:
    phase: CircuitPhase
    consecutive_failures: int
    in_flight: bool
    reopen_at: int | None
    last_event_tick: int


@dataclass(frozen=True, slots=True)
class CircuitTransition:
    state: CircuitState
    admitted: bool | None


def _exact_signed_64(value: object, *, field: str) -> int:
    if type(value) is not int:
        raise TypeError(f"{field} must be an exact integer")
    if not 0 <= value <= _MAX_SIGNED_64:
        raise ValueError(f"{field} must be a non-negative signed 64-bit integer")
    return value


def _validate_policy(policy: object) -> CircuitPolicy:
    if type(policy) is not CircuitPolicy:
        raise TypeError("policy must be an exact CircuitPolicy")
    threshold = _exact_signed_64(
        policy.failure_threshold,
        field="policy.failure_threshold",
    )
    if not 1 <= threshold <= _MAX_FAILURE_THRESHOLD:
        raise ValueError("policy.failure_threshold is outside the supported range")
    duration = _exact_signed_64(
        policy.open_duration,
        field="policy.open_duration",
    )
    if duration == 0:
        raise ValueError("policy.open_duration must be positive")
    return policy


def _validate_state(state: object, policy: CircuitPolicy) -> CircuitState:
    if type(state) is not CircuitState:
        raise TypeError("state must be an exact CircuitState")
    if type(state.phase) is not CircuitPhase:
        raise TypeError("state.phase must be an exact CircuitPhase")
    failures = _exact_signed_64(
        state.consecutive_failures,
        field="state.consecutive_failures",
    )
    if type(state.in_flight) is not bool:
        raise TypeError("state.in_flight must be an exact boolean")
    last_tick = _exact_signed_64(
        state.last_event_tick,
        field="state.last_event_tick",
    )
    reopen_at = state.reopen_at
    if reopen_at is not None:
        reopen_at = _exact_signed_64(reopen_at, field="state.reopen_at")

    if state.phase is CircuitPhase.CLOSED:
        if reopen_at is not None:
            raise ValueError("closed state must not have a reopen tick")
        if not 0 <= failures < policy.failure_threshold:
            raise ValueError("closed failures must be below the policy threshold")
    elif state.phase is CircuitPhase.OPEN:
        if state.in_flight:
            raise ValueError("open state must not have a reservation")
        if failures != policy.failure_threshold:
            raise ValueError("open failures must equal the policy threshold")
        if reopen_at is None or reopen_at <= last_tick:
            raise ValueError("open reopen_at must be after the last event tick")
    else:
        if not state.in_flight:
            raise ValueError("half-open state requires one active probe")
        if failures != policy.failure_threshold:
            raise ValueError("half-open failures must equal the policy threshold")
        if reopen_at is not None:
            raise ValueError("half-open state must not have a reopen tick")
    return state


def _at_tick(state: CircuitState, tick: int) -> CircuitState:
    return CircuitState(
        phase=state.phase,
        consecutive_failures=state.consecutive_failures,
        in_flight=state.in_flight,
        reopen_at=state.reopen_at,
        last_event_tick=tick,
    )


def _reopen_tick(policy: CircuitPolicy, tick: int) -> int:
    if policy.open_duration > _MAX_SIGNED_64 - tick:
        raise OverflowError("reopen tick exceeds the signed 64-bit range")
    return tick + policy.open_duration


def reduce_circuit_breaker(
    policy: CircuitPolicy,
    state: CircuitState,
    event: CircuitEvent,
    tick: int,
) -> CircuitTransition:
    """Reduce one serialized circuit event into frozen state and admission."""
    checked_policy = _validate_policy(policy)
    checked_state = _validate_state(state, checked_policy)
    if type(event) is not CircuitEvent:
        raise TypeError("event must be an exact CircuitEvent")
    checked_tick = _exact_signed_64(tick, field="tick")
    if checked_tick < checked_state.last_event_tick:
        raise ValueError("tick must not precede the last event tick")

    if event is CircuitEvent.REQUEST:
        if checked_state.in_flight:
            return CircuitTransition(_at_tick(checked_state, checked_tick), False)
        if checked_state.phase is CircuitPhase.CLOSED:
            return CircuitTransition(
                CircuitState(
                    phase=CircuitPhase.CLOSED,
                    consecutive_failures=checked_state.consecutive_failures,
                    in_flight=True,
                    reopen_at=None,
                    last_event_tick=checked_tick,
                ),
                True,
            )
        if checked_tick < checked_state.reopen_at:
            return CircuitTransition(_at_tick(checked_state, checked_tick), False)
        return CircuitTransition(
            CircuitState(
                phase=CircuitPhase.HALF_OPEN,
                consecutive_failures=checked_policy.failure_threshold,
                in_flight=True,
                reopen_at=None,
                last_event_tick=checked_tick,
            ),
            True,
        )

    if not checked_state.in_flight:
        raise ValueError("an outcome requires an active reservation")

    if event is CircuitEvent.SUCCESS:
        return CircuitTransition(
            CircuitState(
                phase=CircuitPhase.CLOSED,
                consecutive_failures=0,
                in_flight=False,
                reopen_at=None,
                last_event_tick=checked_tick,
            ),
            None,
        )

    if checked_state.phase is CircuitPhase.CLOSED:
        failures = checked_state.consecutive_failures + 1
        if failures < checked_policy.failure_threshold:
            return CircuitTransition(
                CircuitState(
                    phase=CircuitPhase.CLOSED,
                    consecutive_failures=failures,
                    in_flight=False,
                    reopen_at=None,
                    last_event_tick=checked_tick,
                ),
                None,
            )
    else:
        failures = checked_policy.failure_threshold

    reopen_at = _reopen_tick(checked_policy, checked_tick)
    return CircuitTransition(
        CircuitState(
            phase=CircuitPhase.OPEN,
            consecutive_failures=failures,
            in_flight=False,
            reopen_at=reopen_at,
            last_event_tick=checked_tick,
        ),
        None,
    )
```

## Example

```python
policy = CircuitPolicy(failure_threshold=2, open_duration=5)
initial = CircuitState(CircuitPhase.CLOSED, 0, False, None, 0)

first_request = reduce_circuit_breaker(policy, initial, CircuitEvent.REQUEST, 0)
first_failure = reduce_circuit_breaker(
    policy,
    first_request.state,
    CircuitEvent.FAILURE,
    1,
)
second_request = reduce_circuit_breaker(
    policy,
    first_failure.state,
    CircuitEvent.REQUEST,
    2,
)
opened = reduce_circuit_breaker(
    policy,
    second_request.state,
    CircuitEvent.FAILURE,
    3,
)
denied = reduce_circuit_breaker(policy, opened.state, CircuitEvent.REQUEST, 7)
probe = reduce_circuit_breaker(policy, denied.state, CircuitEvent.REQUEST, 8)
duplicate_probe = reduce_circuit_breaker(
    policy,
    probe.state,
    CircuitEvent.REQUEST,
    8,
)
recovered = reduce_circuit_breaker(
    policy,
    duplicate_probe.state,
    CircuitEvent.SUCCESS,
    9,
)

try:
    reduce_circuit_breaker(policy, initial, CircuitEvent.SUCCESS, 0)
except ValueError:
    unreserved_outcome_rejected = True
else:
    unreserved_outcome_rejected = False

maximum_tick_state = CircuitState(
    CircuitPhase.CLOSED,
    0,
    True,
    None,
    _MAX_SIGNED_64,
)
try:
    reduce_circuit_breaker(
        CircuitPolicy(failure_threshold=1, open_duration=1),
        maximum_tick_state,
        CircuitEvent.FAILURE,
        _MAX_SIGNED_64,
    )
except OverflowError:
    overflow_rejected = True
else:
    overflow_rejected = False

assert (
    first_request.admitted,
    first_failure.state.consecutive_failures,
    opened.state.phase,
    opened.state.reopen_at,
    denied.admitted,
    denied.state.last_event_tick,
    probe.admitted,
    duplicate_probe.admitted,
    duplicate_probe.state.phase,
    recovered.state,
    unreserved_outcome_rejected,
    overflow_rejected,
) == (
    True,
    1,
    CircuitPhase.OPEN,
    8,
    False,
    7,
    True,
    False,
    CircuitPhase.HALF_OPEN,
    CircuitState(CircuitPhase.CLOSED, 0, False, None, 9),
    True,
    True,
)
```

## Trade-offs and Limitations

Validation and every transition take `O(1)` time and memory. Exact integer
ticks avoid floating-point boundary ambiguity, but they have no unit until the
caller consistently supplies one. The breaker does not advance on its own; an
open state becomes half-open only when a request arrives at or after its
reopen tick.

The model deliberately permits only one reserved operation. Denied requests
advance `last_event_tick` but otherwise preserve state, and every admitted
request must be paired with exactly one outcome by the caller. The returned
objects are frozen, so an overflow or validation error cannot partially mutate
the prior state. State must continue under a compatible policy and one
nondecreasing tick domain.

This reducer does not provide locks, concurrent admission, clocks, operation
invocation, exception classification, rolling windows, retries, backoff,
timeouts, cancellation, persistence, distribution, or metrics.

## Related Snippets

<!-- catalog:related:start -->
- [Hold a Switch Active Through a Monotonic Cooldown](hold-a-switch-active-through-a-monotonic-cooldown.md)
- [Plan Readiness Recovery Through a Monotonic Reset Cooldown](plan-readiness-recovery-through-a-monotonic-reset-cooldown.md)
- [Poll a Remote Operation Within Deadline and Failure Budgets](poll-a-remote-operation-within-deadline-and-failure-budgets.md)
<!-- catalog:related:end -->
