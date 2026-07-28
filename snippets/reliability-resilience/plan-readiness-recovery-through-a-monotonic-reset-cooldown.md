---
title: "Plan Readiness Recovery Through a Monotonic Reset Cooldown"
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
  - plan-a-versioned-transition-for-the-current-workflow-attempt.md
  - reduce-bounded-acknowledgements-into-exactly-once-completions.md
---

# Plan Readiness Recovery Through a Monotonic Reset Cooldown

## Idea and Problem

Reduce one readiness observation into an acknowledged reset plan without turning a transient or repeated unready report into a reset storm.

A running target must remain continuously unready for a grace interval before
one numbered request is planned. That request stays pending until a later
observation explicitly acknowledges its token, after which a monotonic cooldown
suppresses another request. Ready and stopped observations break the unready
streak and re-arm its grace rule, while a policy cap bounds the total number of
requests. Tokens are stable increasing counters carried in the returned state.

## When to Use

Use this pattern one observation at a time when readiness and reset
acknowledgements come from one non-decreasing monotonic clock. A separate owner
must consume a returned request before feeding its acknowledged token into a
later transition. The state must remain within the lifetime and epoch of that
clock.

This policy plans resets only for a target that is running but unready. A stopped
target is treated as a break in readiness continuity; starting it or deciding
why it stopped belongs to another policy.

## Implementation

```python
import math
from dataclasses import dataclass
from enum import StrEnum

_MAX_POLICY_SECONDS = 30 * 24 * 60 * 60
_MAX_RESET_REQUESTS = 256
_MAX_TOKEN = 2**63 - 1


class Readiness(StrEnum):
    STOPPED = "stopped"
    RUNNING_READY = "running-ready"
    RUNNING_UNREADY = "running-unready"


@dataclass(frozen=True, slots=True)
class ResetPolicy:
    unready_grace_seconds: float
    reset_cooldown_seconds: float
    max_reset_requests: int


@dataclass(frozen=True, slots=True)
class ReadinessObservation:
    observed_at: float
    status: Readiness
    acknowledged_token: int | None = None


@dataclass(frozen=True, slots=True)
class ReadinessRecoveryState:
    last_observed_at: float | None = None
    last_status: Readiness | None = None
    unready_since: float | None = None
    cooldown_until: float | None = None
    last_issued_token: int = 0
    last_acknowledged_token: int = 0
    pending_requested_at: float | None = None


@dataclass(frozen=True, slots=True)
class ResetRequest:
    token: int
    requested_at: float
    continuously_unready_since: float


@dataclass(frozen=True, slots=True)
class ReadinessRecoveryPlan:
    state: ReadinessRecoveryState
    reset_request: ResetRequest | None
    action_limit_reached: bool


class StaleReadinessInputError(ValueError):
    pass


def _finite_nonnegative_time(value: object, *, field: str) -> float:
    if type(value) not in (int, float):
        raise TypeError(f"{field} must be an exact integer or float")
    try:
        converted = float(value)
    except OverflowError:
        raise ValueError(f"{field} must be representable as a finite float") from None
    if not math.isfinite(converted) or converted < 0:
        raise ValueError(f"{field} must be finite and non-negative")
    return converted


def _positive_duration(value: object, *, field: str) -> float:
    duration = _finite_nonnegative_time(value, field=field)
    if not 0 < duration <= _MAX_POLICY_SECONDS:
        raise ValueError(f"{field} must be positive and at most 30 days")
    return duration


def _token(value: object, *, field: str, minimum: int = 0) -> int:
    if type(value) is not int:
        raise TypeError(f"{field} must be an exact integer")
    if not minimum <= value <= _MAX_TOKEN:
        raise ValueError(f"{field} is outside the supported token range")
    return value


def _future_deadline(now: float, duration: float) -> float:
    deadline = now + duration
    if not math.isfinite(deadline) or deadline <= now:
        raise OverflowError("the cooldown deadline is not representable")
    return deadline


def _validated_policy(value: object) -> ResetPolicy:
    if type(value) is not ResetPolicy:
        raise TypeError("policy must be an exact ResetPolicy")
    if type(value.max_reset_requests) is not int:
        raise TypeError("policy.max_reset_requests must be an exact integer")
    if not 1 <= value.max_reset_requests <= _MAX_RESET_REQUESTS:
        raise ValueError("policy.max_reset_requests must be between 1 and 256")
    return ResetPolicy(
        unready_grace_seconds=_positive_duration(
            value.unready_grace_seconds,
            field="policy.unready_grace_seconds",
        ),
        reset_cooldown_seconds=_positive_duration(
            value.reset_cooldown_seconds,
            field="policy.reset_cooldown_seconds",
        ),
        max_reset_requests=value.max_reset_requests,
    )


def _validated_state(
    value: object,
    *,
    policy: ResetPolicy,
) -> ReadinessRecoveryState:
    if type(value) is not ReadinessRecoveryState:
        raise TypeError("state must be an exact ReadinessRecoveryState")
    issued = _token(value.last_issued_token, field="state.last_issued_token")
    acknowledged = _token(
        value.last_acknowledged_token,
        field="state.last_acknowledged_token",
    )
    if issued > policy.max_reset_requests:
        raise ValueError("state has issued more requests than the policy permits")
    if acknowledged > issued or issued - acknowledged > 1:
        raise ValueError("state token counts describe an impossible acknowledgement state")

    if value.last_observed_at is None:
        if (
            value.last_status is not None
            or value.unready_since is not None
            or value.cooldown_until is not None
            or value.pending_requested_at is not None
            or issued != 0
            or acknowledged != 0
        ):
            raise ValueError("an unobserved state cannot contain readiness history")
        return ReadinessRecoveryState()

    last_observed_at = _finite_nonnegative_time(
        value.last_observed_at,
        field="state.last_observed_at",
    )
    if type(value.last_status) is not Readiness:
        raise TypeError("state.last_status must be an exact Readiness value")

    if value.last_status is Readiness.RUNNING_UNREADY:
        if value.unready_since is None:
            raise ValueError("running-unready state requires unready_since")
        unready_since = _finite_nonnegative_time(
            value.unready_since,
            field="state.unready_since",
        )
        if unready_since > last_observed_at:
            raise ValueError("state.unready_since cannot follow its last observation")
    else:
        if value.unready_since is not None:
            raise ValueError("only running-unready state can have unready_since")
        unready_since = None

    has_pending_request = issued == acknowledged + 1
    if has_pending_request:
        if value.pending_requested_at is None:
            raise ValueError("an unacknowledged token requires pending_requested_at")
        pending_requested_at = _finite_nonnegative_time(
            value.pending_requested_at,
            field="state.pending_requested_at",
        )
        if pending_requested_at > last_observed_at:
            raise ValueError("state.pending_requested_at cannot be in the future")
        if value.cooldown_until is not None:
            raise ValueError("a pending request and cooldown cannot coexist")
        cooldown_until = None
    else:
        if value.pending_requested_at is not None:
            raise ValueError("state has a pending time without an unacknowledged token")
        pending_requested_at = None
        if value.cooldown_until is None:
            cooldown_until = None
        else:
            cooldown_until = _finite_nonnegative_time(
                value.cooldown_until,
                field="state.cooldown_until",
            )
            if acknowledged == 0:
                raise ValueError("a cooldown requires an acknowledged request")
            if cooldown_until <= last_observed_at:
                raise ValueError("an expired cooldown must not remain in state")
            maximum_cooldown_until = _future_deadline(
                last_observed_at,
                policy.reset_cooldown_seconds,
            )
            if cooldown_until > maximum_cooldown_until:
                raise ValueError("state cooldown exceeds the current policy duration")

    return ReadinessRecoveryState(
        last_observed_at=last_observed_at,
        last_status=value.last_status,
        unready_since=unready_since,
        cooldown_until=cooldown_until,
        last_issued_token=issued,
        last_acknowledged_token=acknowledged,
        pending_requested_at=pending_requested_at,
    )


def _validated_observation(
    value: object,
    *,
    after: float | None,
) -> ReadinessObservation:
    if type(value) is not ReadinessObservation:
        raise TypeError("observation must be an exact ReadinessObservation")
    observed_at = _finite_nonnegative_time(
        value.observed_at,
        field="observation.observed_at",
    )
    if after is not None and observed_at < after:
        raise StaleReadinessInputError("observation time must be non-decreasing")
    if type(value.status) is not Readiness:
        raise TypeError("observation.status must be an exact Readiness value")
    acknowledged_token = value.acknowledged_token
    if acknowledged_token is not None:
        acknowledged_token = _token(
            acknowledged_token,
            field="observation.acknowledged_token",
            minimum=1,
        )
    return ReadinessObservation(
        observed_at=observed_at,
        status=value.status,
        acknowledged_token=acknowledged_token,
    )


def plan_readiness_recovery(
    state: ReadinessRecoveryState,
    observation: ReadinessObservation,
    policy: ResetPolicy,
) -> ReadinessRecoveryPlan:
    """Reduce one immutable observation into state and at most one request."""
    checked_policy = _validated_policy(policy)
    current = _validated_state(state, policy=checked_policy)
    checked_observation = _validated_observation(
        observation,
        after=current.last_observed_at,
    )

    now = checked_observation.observed_at
    cooldown_until = current.cooldown_until
    if cooldown_until is not None and now >= cooldown_until:
        cooldown_until = None

    issued = current.last_issued_token
    acknowledged = current.last_acknowledged_token
    pending_requested_at = current.pending_requested_at
    if checked_observation.acknowledged_token is not None:
        if issued == acknowledged or pending_requested_at is None:
            raise StaleReadinessInputError("acknowledgement has no pending request")
        if checked_observation.acknowledged_token != issued:
            raise StaleReadinessInputError("acknowledgement token is stale or unknown")
        if now <= pending_requested_at:
            raise StaleReadinessInputError("acknowledgement must be later than its request")
        acknowledged = issued
        pending_requested_at = None
        cooldown_until = _future_deadline(
            now,
            checked_policy.reset_cooldown_seconds,
        )

    if checked_observation.status is Readiness.RUNNING_UNREADY:
        if current.last_status is Readiness.RUNNING_UNREADY and current.unready_since is not None:
            unready_since = current.unready_since
        else:
            unready_since = now
    else:
        unready_since = None

    reset_request = None
    action_limit_reached = False
    has_pending_request = issued == acknowledged + 1
    if (
        checked_observation.status is Readiness.RUNNING_UNREADY
        and not has_pending_request
        and cooldown_until is None
        and now - unready_since >= checked_policy.unready_grace_seconds
    ):
        if issued >= checked_policy.max_reset_requests:
            action_limit_reached = True
        else:
            issued += 1
            pending_requested_at = now
            reset_request = ResetRequest(
                token=issued,
                requested_at=now,
                continuously_unready_since=unready_since,
            )

    current = ReadinessRecoveryState(
        last_observed_at=now,
        last_status=checked_observation.status,
        unready_since=unready_since,
        cooldown_until=cooldown_until,
        last_issued_token=issued,
        last_acknowledged_token=acknowledged,
        pending_requested_at=pending_requested_at,
    )

    return ReadinessRecoveryPlan(
        state=current,
        reset_request=reset_request,
        action_limit_reached=action_limit_reached,
    )
```

## Example

```python
policy = ResetPolicy(
    unready_grace_seconds=5,
    reset_cooldown_seconds=4,
    max_reset_requests=2,
)
state = ReadinessRecoveryState()
for observation in (
    ReadinessObservation(0, Readiness.STOPPED),
    ReadinessObservation(1, Readiness.RUNNING_UNREADY),
    ReadinessObservation(5, Readiness.RUNNING_UNREADY),
):
    transition = plan_readiness_recovery(state, observation, policy)
    assert transition.reset_request is None
    state = transition.state

first = plan_readiness_recovery(
    state,
    ReadinessObservation(6, Readiness.RUNNING_UNREADY),
    policy,
)
pending = plan_readiness_recovery(
    first.state,
    ReadinessObservation(6, Readiness.RUNNING_UNREADY),
    policy,
)
acknowledged_first = plan_readiness_recovery(
    pending.state,
    ReadinessObservation(7, Readiness.RUNNING_UNREADY, acknowledged_token=1),
    policy,
)
cooling_down = plan_readiness_recovery(
    acknowledged_first.state,
    ReadinessObservation(10, Readiness.RUNNING_UNREADY),
    policy,
)
second = plan_readiness_recovery(
    cooling_down.state,
    ReadinessObservation(11, Readiness.RUNNING_UNREADY),
    policy,
)
acknowledged_second = plan_readiness_recovery(
    second.state,
    ReadinessObservation(12, Readiness.RUNNING_READY, acknowledged_token=2),
    policy,
)
fresh_grace = plan_readiness_recovery(
    acknowledged_second.state,
    ReadinessObservation(16, Readiness.RUNNING_UNREADY),
    policy,
)
capped = plan_readiness_recovery(
    fresh_grace.state,
    ReadinessObservation(21, Readiness.RUNNING_UNREADY),
    policy,
)
try:
    plan_readiness_recovery(
        capped.state,
        ReadinessObservation(20, Readiness.STOPPED),
        policy,
    )
except StaleReadinessInputError:
    stale_rejected = True
else:
    stale_rejected = False

try:
    plan_readiness_recovery(
        ReadinessRecoveryState(
            last_observed_at=10,
            last_status=Readiness.RUNNING_READY,
            cooldown_until=20,
            last_issued_token=1,
            last_acknowledged_token=1,
        ),
        ReadinessObservation(10, Readiness.RUNNING_READY),
        policy,
    )
except ValueError:
    impossible_cooldown_rejected = True
else:
    impossible_cooldown_rejected = False

assert (first.reset_request, pending.reset_request, second.reset_request) == (
    ResetRequest(1, 6.0, 1.0),
    None,
    ResetRequest(2, 11.0, 1.0),
)
assert capped.state == ReadinessRecoveryState(
    last_observed_at=21.0,
    last_status=Readiness.RUNNING_UNREADY,
    unready_since=16.0,
    cooldown_until=None,
    last_issued_token=2,
    last_acknowledged_token=2,
    pending_requested_at=None,
)
assert capped.reset_request is None
assert capped.action_limit_reached is True
assert stale_rejected is True
assert impossible_cooldown_rejected is True
```

## Trade-offs and Limitations

Each transition consumes one observation and emits at most one request; a state
can emit at most 256 requests over its lifetime. Equal observation timestamps
are accepted, but an acknowledgement must arrive in a later transition with a
strictly later timestamp than its request. A pending request suppresses all
duplicates until acknowledged; if acknowledgement never arrives, this planner
deliberately remains pending.

Readiness is observation-driven, so grace and cooldown boundaries do nothing
until another observation is supplied. Acknowledging a token confirms only that
the request was consumed; it does not prove that a reset succeeded or that the
target became ready. Raw monotonic times generally cannot cross process restarts.
The reducer invokes no callback, performs no reset, retry, persistence, clock
read, or other effect, and its requests are plans rather than success reports.

## Related Snippets

<!-- catalog:related:start -->
- [Hold a Switch Active Through a Monotonic Cooldown](hold-a-switch-active-through-a-monotonic-cooldown.md)
- [Plan a Versioned Transition for the Current Workflow Attempt](plan-a-versioned-transition-for-the-current-workflow-attempt.md)
- [Reduce Bounded Acknowledgements into Exactly-Once Completions](reduce-bounded-acknowledgements-into-exactly-once-completions.md)
<!-- catalog:related:end -->
