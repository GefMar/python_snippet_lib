---
title: "Hold a Switch Active Through a Monotonic Cooldown"
snippet_type: pattern
use_cases:
  - observability
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - wait-for-a-predicate-until-a-monotonic-deadline.md
  - ../observability-operations/count-values-in-fixed-upper-bound-bins.md
---

# Hold a Switch Active Through a Monotonic Cooldown

## Idea and Problem

Keep a threshold-activated switch on until a full quiet cooldown has elapsed, without embedding clocks or side effects in the state transition.

A high observation activates the switch or extends its deadline. A low
observation leaves it active before that deadline and deactivates it at or
after the boundary. Returning an explicit action lets the caller perform each
external transition once while tests use synthetic monotonic timestamps.

## When to Use

Use this pattern when noisy measurements control a protective or degraded mode
and turning it off too quickly is harmful. Supply timestamps from one
non-decreasing monotonic clock and persist the returned state only within that
clock's lifetime. Use dual thresholds when genuine hysteresis is required, or a
scheduler when deactivation must happen without a new observation.

## Implementation

```python
import math
from dataclasses import dataclass
from enum import Enum


class SwitchAction(Enum):
    NONE = "none"
    ACTIVATE = "activate"
    DEACTIVATE = "deactivate"


@dataclass(frozen=True, slots=True)
class CooldownState:
    active_until: float | None = None
    last_observed_at: float | None = None


@dataclass(frozen=True, slots=True)
class CooldownUpdate:
    state: CooldownState
    action: SwitchAction


def _finite_number(value: int | float, *, name: str) -> float:
    if isinstance(value, bool) or not isinstance(value, (int, float)):
        raise TypeError(f"{name} must be an integer or float")
    try:
        converted = float(value)
    except OverflowError as error:
        raise ValueError(f"{name} must be representable as a finite float") from error
    if not math.isfinite(converted):
        raise ValueError(f"{name} must be finite")
    return converted


def update_cooldown_switch(
    state: CooldownState,
    *,
    observed: int | float,
    threshold: int | float,
    cooldown: int | float,
    now: int | float,
) -> CooldownUpdate:
    if not isinstance(state, CooldownState):
        raise TypeError("state must be a CooldownState")
    observed_value = _finite_number(observed, name="observed")
    threshold_value = _finite_number(threshold, name="threshold")
    cooldown_value = _finite_number(cooldown, name="cooldown")
    now_value = _finite_number(now, name="now")
    if cooldown_value <= 0:
        raise ValueError("cooldown must be positive")
    if now_value < 0:
        raise ValueError("now must be non-negative")

    last_observed_at = state.last_observed_at
    if last_observed_at is not None:
        last_observed_at = _finite_number(
            last_observed_at,
            name="state.last_observed_at",
        )
        if last_observed_at < 0 or now_value < last_observed_at:
            raise ValueError("now must not precede the previous observation")

    active_until = state.active_until
    if active_until is not None:
        active_until = _finite_number(active_until, name="state.active_until")
        if last_observed_at is None:
            raise ValueError("active state requires a previous observation")
        if active_until < 0:
            raise ValueError("state.active_until must be non-negative")
        if active_until <= last_observed_at:
            raise ValueError("active state must end after its previous observation")

    if observed_value >= threshold_value:
        candidate_deadline = now_value + cooldown_value
        if not math.isfinite(candidate_deadline) or candidate_deadline <= now_value:
            raise OverflowError("cooldown deadline is not representable")
        new_deadline = (
            candidate_deadline
            if active_until is None
            else max(active_until, candidate_deadline)
        )
        return CooldownUpdate(
            state=CooldownState(new_deadline, now_value),
            action=(
                SwitchAction.NONE
                if active_until is not None
                else SwitchAction.ACTIVATE
            ),
        )

    if active_until is not None and now_value < active_until:
        return CooldownUpdate(
            state=CooldownState(active_until, now_value),
            action=SwitchAction.NONE,
        )

    return CooldownUpdate(
        state=CooldownState(None, now_value),
        action=(
            SwitchAction.DEACTIVATE
            if active_until is not None
            else SwitchAction.NONE
        ),
    )
```

## Example

```python
state = CooldownState()
first = update_cooldown_switch(
    state,
    observed=10,
    threshold=10,
    cooldown=5,
    now=0,
)
not_shortened = update_cooldown_switch(
    first.state,
    observed=12,
    threshold=10,
    cooldown=1,
    now=1,
)
during = update_cooldown_switch(
    not_shortened.state,
    observed=2,
    threshold=10,
    cooldown=5,
    now=2,
)
extended = update_cooldown_switch(
    during.state,
    observed=12,
    threshold=10,
    cooldown=5,
    now=4,
)
expired = update_cooldown_switch(
    extended.state,
    observed=2,
    threshold=10,
    cooldown=5,
    now=9,
)
steady = update_cooldown_switch(
    expired.state,
    observed=2,
    threshold=10,
    cooldown=5,
    now=10,
)

try:
    update_cooldown_switch(
        CooldownState(active_until=5),
        observed=0,
        threshold=10,
        cooldown=5,
        now=1,
    )
except ValueError:
    invalid_state_rejected = True
else:
    invalid_state_rejected = False

assert (
    first.action,
    first.state.active_until,
    not_shortened.action,
    not_shortened.state.active_until,
    during.action,
    extended.action,
    extended.state.active_until,
    expired.action,
    expired.state.active_until,
    steady.action,
    invalid_state_rejected,
) == (
    SwitchAction.ACTIVATE,
    5.0,
    SwitchAction.NONE,
    5.0,
    SwitchAction.NONE,
    SwitchAction.NONE,
    9.0,
    SwitchAction.DEACTIVATE,
    None,
    SwitchAction.NONE,
    True,
)
```

## Trade-offs and Limitations

This is a hold-down interval, not classical two-threshold hysteresis. Expiry is
observation-driven: no transition occurs until the next call, and a high value
extends an already active state even if its old deadline has passed. The state
does not execute, retry, or roll back side effects; callers own persistence,
concurrency control, and exactly-once action handling. Monotonic timestamps are
process-local on typical systems, so raw deadlines usually cannot survive a
restart. The rule also reacts to one value at a time and performs no smoothing.

## Related Snippets

<!-- catalog:related:start -->
- [Wait for a Predicate Until a Monotonic Deadline](wait-for-a-predicate-until-a-monotonic-deadline.md)
- [Count Values in Fixed Upper-Bound Bins](../observability-operations/count-values-in-fixed-upper-bound-bins.md)
<!-- catalog:related:end -->
