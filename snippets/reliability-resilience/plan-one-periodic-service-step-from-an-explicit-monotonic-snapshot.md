---
title: "Plan One Periodic-Service Step from an Explicit Monotonic Snapshot"
snippet_type: algorithm
use_cases:
  - lifecycle-management
  - retry-recovery
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - plan-a-versioned-transition-for-the-current-workflow-attempt.md
  - plan-readiness-recovery-through-a-monotonic-reset-cooldown.md
  - hold-a-switch-active-through-a-monotonic-cooldown.md
---

# Plan One Periodic-Service Step from an Explicit Monotonic Snapshot

## Idea and Problem

Reduce one immutable periodic-service snapshot to one revision-bound piece of advice without reading a clock or starting work.

The lifecycle is closed to `IDLE`, `ACTIVE`, and `STOPPED`, and the result is
closed to `WAIT`, `START`, `STOP`, and `EXPIRED`. A successful `START` plan
contains the complete successor state, including its next due time and active
deadline, so that state can be conditionally accepted before external work is
started.

## When to Use

Use this algorithm when a caller already has one coherent service snapshot and
one finite reading from the same monotonic clock epoch. The interval is the
maximum active window and also the delay from an accepted start to the next due
boundary. A completion path outside this planner may replace an `ACTIVE` state
with an `IDLE` state while preserving that `next_due` value.

Advice has exact precedence: an already stopped state returns `STOP`; otherwise
a stop request returns `STOP`; otherwise an active state at or beyond its
deadline returns `EXPIRED`; otherwise an active state returns `WAIT`; otherwise
an idle state at or beyond `next_due` returns `START`; the remaining idle case
returns `WAIT`. Thus equality is due for `IDLE` and expired for `ACTIVE`, while
a stop request wins both boundaries.

## Implementation

```python
import math
from dataclasses import dataclass
from enum import StrEnum

_LAST_REVISION = 2**63 - 1


class ServicePhase(StrEnum):
    IDLE = "idle"
    ACTIVE = "active"
    STOPPED = "stopped"


class StepAdvice(StrEnum):
    WAIT = "wait"
    START = "start"
    STOP = "stop"
    EXPIRED = "expired"


class StepReason(StrEnum):
    ALREADY_STOPPED = "already-stopped"
    STOP_REQUESTED = "stop-requested"
    ACTIVE_DEADLINE_REACHED = "active-deadline-reached"
    ACTIVE_BEFORE_DEADLINE = "active-before-deadline"
    NEXT_DUE_REACHED = "next-due-reached"
    BEFORE_NEXT_DUE = "before-next-due"


@dataclass(frozen=True, slots=True)
class PeriodicServiceState:
    phase: ServicePhase
    revision: int
    next_due: int | float | None
    active_deadline: int | float | None


@dataclass(frozen=True, slots=True)
class PeriodicStepPlan:
    expected_revision: int
    advice: StepAdvice
    reason: StepReason
    successor: PeriodicServiceState | None


class StalePeriodicServiceSnapshotError(ValueError):
    pass


def _revision(value: object, *, field: str) -> int:
    if type(value) is not int:
        raise TypeError(f"{field} must be an exact integer")
    if not 0 <= value <= _LAST_REVISION:
        raise ValueError(f"{field} is outside the supported revision range")
    return value


def _finite_number(value: object, *, field: str) -> int | float:
    if type(value) not in (int, float):
        raise TypeError(f"{field} must be an exact integer or float")
    if type(value) is float and not math.isfinite(value):
        raise ValueError(f"{field} must be finite")
    return value


def _positive_interval(value: object) -> int | float:
    interval = _finite_number(value, field="interval")
    if interval <= 0.0:
        raise ValueError("interval must be positive")
    return interval


def _optional_time(value: object, *, field: str) -> int | float | None:
    if value is None:
        return None
    return _finite_number(value, field=field)


def _state(value: object) -> PeriodicServiceState:
    if type(value) is not PeriodicServiceState:
        raise TypeError("snapshot must be an exact PeriodicServiceState")
    if type(value.phase) is not ServicePhase:
        raise TypeError("snapshot.phase must be an exact ServicePhase")

    revision = _revision(value.revision, field="snapshot.revision")
    next_due = _optional_time(value.next_due, field="snapshot.next_due")
    active_deadline = _optional_time(
        value.active_deadline,
        field="snapshot.active_deadline",
    )

    if value.phase is ServicePhase.IDLE:
        if next_due is None or active_deadline is not None:
            raise ValueError("IDLE requires next_due and forbids active_deadline")
    elif value.phase is ServicePhase.ACTIVE:
        if next_due is None or active_deadline is None:
            raise ValueError("ACTIVE requires next_due and active_deadline")
        if active_deadline > next_due:
            raise ValueError("active_deadline must not follow next_due")
    elif next_due is not None or active_deadline is not None:
        raise ValueError("STOPPED forbids next_due and active_deadline")

    return PeriodicServiceState(
        phase=value.phase,
        revision=revision,
        next_due=next_due,
        active_deadline=active_deadline,
    )


def _advanced_revision(revision: int) -> int:
    if revision == _LAST_REVISION:
        raise OverflowError("snapshot revision cannot be advanced")
    return revision + 1


def _future_boundary(
    now: int | float,
    interval: int | float,
) -> int | float:
    try:
        boundary = now + interval
    except OverflowError:
        raise OverflowError("now plus interval is not a later finite boundary") from None
    if (type(boundary) is float and not math.isfinite(boundary)) or boundary <= now:
        raise OverflowError("now plus interval is not a later finite boundary")
    return boundary


def plan_periodic_service_step(
    snapshot: PeriodicServiceState,
    *,
    expected_revision: int,
    now: int | float,
    interval: int | float,
    stop_requested: bool,
) -> PeriodicStepPlan:
    state = _state(snapshot)
    expected = _revision(expected_revision, field="expected_revision")
    instant = _finite_number(now, field="now")
    cadence = _positive_interval(interval)
    if type(instant) is not type(cadence):
        raise TypeError("now and interval must use the same exact numeric type")
    if type(stop_requested) is not bool:
        raise TypeError("stop_requested must be an exact boolean")
    if state.revision != expected:
        raise StalePeriodicServiceSnapshotError(
            "snapshot revision does not match expected_revision"
        )

    def result(
        advice: StepAdvice,
        reason: StepReason,
        successor: PeriodicServiceState | None = None,
    ) -> PeriodicStepPlan:
        return PeriodicStepPlan(expected, advice, reason, successor)

    if state.phase is ServicePhase.STOPPED:
        return result(StepAdvice.STOP, StepReason.ALREADY_STOPPED)

    if stop_requested:
        stopped = PeriodicServiceState(
            ServicePhase.STOPPED,
            _advanced_revision(state.revision),
            None,
            None,
        )
        return result(StepAdvice.STOP, StepReason.STOP_REQUESTED, stopped)

    if state.phase is ServicePhase.ACTIVE:
        assert state.active_deadline is not None
        if instant >= state.active_deadline:
            return result(
                StepAdvice.EXPIRED,
                StepReason.ACTIVE_DEADLINE_REACHED,
            )
        return result(StepAdvice.WAIT, StepReason.ACTIVE_BEFORE_DEADLINE)

    assert state.next_due is not None
    if instant >= state.next_due:
        boundary = _future_boundary(instant, cadence)
        active = PeriodicServiceState(
            ServicePhase.ACTIVE,
            _advanced_revision(state.revision),
            boundary,
            boundary,
        )
        return result(StepAdvice.START, StepReason.NEXT_DUE_REACHED, active)
    return result(StepAdvice.WAIT, StepReason.BEFORE_NEXT_DUE)
```

## Example

```python
idle = PeriodicServiceState(ServicePhase.IDLE, 8, 40.0, None)
start = plan_periodic_service_step(
    idle,
    expected_revision=8,
    now=40.0,
    interval=10.0,
    stop_requested=False,
)
active = start.successor
assert active is not None

waiting = plan_periodic_service_step(
    active,
    expected_revision=9,
    now=49.5,
    interval=10.0,
    stop_requested=False,
)
expired = plan_periodic_service_step(
    active,
    expected_revision=9,
    now=50.0,
    interval=10.0,
    stop_requested=False,
)
stopping = plan_periodic_service_step(
    active,
    expected_revision=9,
    now=50.0,
    interval=10.0,
    stop_requested=True,
)
large_integer_wait = plan_periodic_service_step(
    PeriodicServiceState(ServicePhase.IDLE, 3, 2**60 + 1, None),
    expected_revision=3,
    now=2**60,
    interval=1,
    stop_requested=False,
)

assert (
    start,
    waiting.advice,
    expired.advice,
    stopping.advice,
    stopping.successor,
    large_integer_wait.advice,
) == (
    PeriodicStepPlan(
        8,
        StepAdvice.START,
        StepReason.NEXT_DUE_REACHED,
        PeriodicServiceState(ServicePhase.ACTIVE, 9, 50.0, 50.0),
    ),
    StepAdvice.WAIT,
    StepAdvice.EXPIRED,
    StepAdvice.STOP,
    PeriodicServiceState(ServicePhase.STOPPED, 10, None, None),
    StepAdvice.WAIT,
)
```

## Trade-offs and Limitations

The state matrix is strict: `IDLE` carries only `next_due`, `ACTIVE` carries
both times with `active_deadline <= next_due`, and `STOPPED` carries neither.
Only exact built-in integers, floats, booleans, enums, and data classes at their
respective boundaries are accepted, so booleans cannot masquerade as times or
revisions. Integer times remain integers instead of being rounded through a
float conversion. `now` and `interval` must use the same exact numeric type, so
successor arithmetic cannot silently mix an integer cadence with float
rounding. Float times and derived float boundaries must be finite; a positive
interval that cannot produce a later finite boundary is rejected. Revisions
occupy the nonnegative signed 64-bit range and never wrap.

Every result is advice tied to `expected_revision`, not permission by itself.
The caller must conditionally accept the successor only while authoritative
state still has that revision, and must accept a `START` successor before doing
external work. A rejected condition makes the whole plan stale. This helper has
no hierarchy, loop, callback, clock read, thread, persistence, or ownership
protocol. Completing active work, resolving `EXPIRED`, storing state, and
arbitrating concurrent callers remain outside it.

## Related Snippets

<!-- catalog:related:start -->
- [Plan a Versioned Transition for the Current Workflow Attempt](plan-a-versioned-transition-for-the-current-workflow-attempt.md)
- [Plan Readiness Recovery Through a Monotonic Reset Cooldown](plan-readiness-recovery-through-a-monotonic-reset-cooldown.md)
- [Hold a Switch Active Through a Monotonic Cooldown](hold-a-switch-active-through-a-monotonic-cooldown.md)
<!-- catalog:related:end -->
