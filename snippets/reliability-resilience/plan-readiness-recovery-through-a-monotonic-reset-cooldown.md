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

Reduce bounded readiness observations into acknowledged reset requests without turning a transient or repeated unready report into a reset storm.

A running target must remain continuously unready for a grace interval before
one numbered request is planned. That request stays pending until a later
observation explicitly acknowledges its token, after which a monotonic cooldown
suppresses another request. Ready and stopped observations break the unready
streak and re-arm its grace rule, while a policy cap bounds the total number of
requests. Tokens are stable increasing counters carried in the returned state.

## When to Use

Use this pattern when readiness observations and reset acknowledgements have
already been collected from one non-decreasing monotonic clock. A separate
owner can consume the returned requests and feed each acknowledged token into a
later observation. The state must remain within the lifetime and epoch of that
clock.

This policy plans resets only for a target that is running but unready. A stopped
target is treated as a break in readiness continuity; starting it or deciding
why it stopped belongs to another policy.

## Implementation

```python
import math
from dataclasses import dataclass
from enum import StrEnum

_MAX_OBSERVATIONS = 1_024
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
    reset_requests: tuple[ResetRequest, ...]
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
            if cooldown_until - last_observed_at > _MAX_POLICY_SECONDS:
                raise ValueError("state cooldown exceeds the supported duration")

    return ReadinessRecoveryState(
        last_observed_at=last_observed_at,
        last_status=value.last_status,
        unready_since=unready_since,
        cooldown_until=cooldown_until,
        last_issued_token=issued,
        last_acknowledged_token=acknowledged,
        pending_requested_at=pending_requested_at,
    )


def _validated_observations(
    value: object,
    *,
    after: float | None,
) -> tuple[ReadinessObservation, ...]:
    if type(value) is not tuple:
        raise TypeError("observations must be an exact tuple")
    if len(value) > _MAX_OBSERVATIONS:
        raise ValueError(f"observations must contain at most {_MAX_OBSERVATIONS} records")

    previous = after
    observations: list[ReadinessObservation] = []
    for index, raw_observation in enumerate(value):
        field = f"observations[{index}]"
        if type(raw_observation) is not ReadinessObservation:
            raise TypeError(f"{field} must be an exact ReadinessObservation")
        observed_at = _finite_nonnegative_time(
            raw_observation.observed_at,
            field=f"{field}.observed_at",
        )
        if previous is not None and observed_at < previous:
            raise StaleReadinessInputError("observation times must be non-decreasing")
        if type(raw_observation.status) is not Readiness:
            raise TypeError(f"{field}.status must be an exact Readiness value")
        acknowledged_token = raw_observation.acknowledged_token
        if acknowledged_token is not None:
            acknowledged_token = _token(
                acknowledged_token,
                field=f"{field}.acknowledged_token",
                minimum=1,
            )
        observations.append(
            ReadinessObservation(
                observed_at=observed_at,
                status=raw_observation.status,
                acknowledged_token=acknowledged_token,
            )
        )
        previous = observed_at
    return tuple(observations)


def plan_readiness_recovery(
    state: ReadinessRecoveryState,
    observations: tuple[ReadinessObservation, ...],
    policy: ResetPolicy,
) -> ReadinessRecoveryPlan:
    """Reduce immutable observations into state and reset requests."""
    checked_policy = _validated_policy(policy)
    current = _validated_state(state, policy=checked_policy)
    checked_observations = _validated_observations(
        observations,
        after=current.last_observed_at,
    )

    requests: list[ResetRequest] = []
    action_limit_reached = False
    for observation in checked_observations:
        now = observation.observed_at
        cooldown_until = current.cooldown_until
        if cooldown_until is not None and now >= cooldown_until:
            cooldown_until = None

        issued = current.last_issued_token
        acknowledged = current.last_acknowledged_token
        pending_requested_at = current.pending_requested_at
        if observation.acknowledged_token is not None:
            if issued == acknowledged or pending_requested_at is None:
                raise StaleReadinessInputError("acknowledgement has no pending request")
            if observation.acknowledged_token != issued:
                raise StaleReadinessInputError("acknowledgement token is stale or unknown")
            if now <= pending_requested_at:
                raise StaleReadinessInputError("acknowledgement must be later than its request")
            acknowledged = issued
            pending_requested_at = None
            cooldown_until = _future_deadline(
                now,
                checked_policy.reset_cooldown_seconds,
            )

        if observation.status is Readiness.RUNNING_UNREADY:
            if (
                current.last_status is Readiness.RUNNING_UNREADY
                and current.unready_since is not None
            ):
                unready_since = current.unready_since
            else:
                unready_since = now
        else:
            unready_since = None

        has_pending_request = issued == acknowledged + 1
        if (
            observation.status is Readiness.RUNNING_UNREADY
            and not has_pending_request
            and cooldown_until is None
            and now - unready_since >= checked_policy.unready_grace_seconds
        ):
            if issued >= checked_policy.max_reset_requests:
                action_limit_reached = True
            else:
                issued += 1
                pending_requested_at = now
                requests.append(
                    ResetRequest(
                        token=issued,
                        requested_at=now,
                        continuously_unready_since=unready_since,
                    )
                )

        current = ReadinessRecoveryState(
            last_observed_at=now,
            last_status=observation.status,
            unready_since=unready_since,
            cooldown_until=cooldown_until,
            last_issued_token=issued,
            last_acknowledged_token=acknowledged,
            pending_requested_at=pending_requested_at,
        )

    return ReadinessRecoveryPlan(
        state=current,
        reset_requests=tuple(requests),
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
observations = (
    ReadinessObservation(0, Readiness.STOPPED),
    ReadinessObservation(1, Readiness.RUNNING_UNREADY),
    ReadinessObservation(5, Readiness.RUNNING_UNREADY),
    ReadinessObservation(6, Readiness.RUNNING_UNREADY),  # request token 1
    ReadinessObservation(6, Readiness.RUNNING_UNREADY),  # pending: no duplicate
    ReadinessObservation(7, Readiness.RUNNING_UNREADY, acknowledged_token=1),
    ReadinessObservation(10, Readiness.RUNNING_UNREADY),  # cooldown
    ReadinessObservation(11, Readiness.RUNNING_UNREADY),  # request token 2
    ReadinessObservation(12, Readiness.RUNNING_READY, acknowledged_token=2),
    ReadinessObservation(16, Readiness.RUNNING_UNREADY),  # fresh grace starts
    ReadinessObservation(21, Readiness.RUNNING_UNREADY),  # cap reached
)

plan = plan_readiness_recovery(ReadinessRecoveryState(), observations, policy)
try:
    plan_readiness_recovery(
        plan.state,
        (ReadinessObservation(20, Readiness.STOPPED),),
        policy,
    )
except StaleReadinessInputError:
    stale_rejected = True
else:
    stale_rejected = False

assert plan.reset_requests == (
    ResetRequest(1, 6.0, 1.0),
    ResetRequest(2, 11.0, 1.0),
)
assert plan.state == ReadinessRecoveryState(
    last_observed_at=21.0,
    last_status=Readiness.RUNNING_UNREADY,
    unready_since=16.0,
    cooldown_until=None,
    last_issued_token=2,
    last_acknowledged_token=2,
    pending_requested_at=None,
)
assert plan.action_limit_reached is True
assert stale_rejected is True
```

## Trade-offs and Limitations

Reduction is linear in at most 1,024 observations and can emit at most 256
requests over a state's lifetime. Equal timestamps are accepted, but an
acknowledgement must have a strictly later timestamp than its request. A pending
request suppresses all duplicates until acknowledged; if acknowledgement never
arrives, this planner deliberately remains pending.

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
