---
title: "Plan One Retry Admission from a Success-Coupled Budget Snapshot"
snippet_type: algorithm
use_cases:
  - retry-recovery
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - resolve-a-guarded-retry-decision-from-operation-and-worker-policies.md
  - plan-one-discrete-token-bucket-admission-from-an-explicit-tick-snapshot.md
  - ../storage-databases/compare-and-swap-a-versioned-sqlite-setting-with-one-conditional-update.md
---

# Plan One Retry Admission from a Success-Coupled Budget Snapshot

## Idea and Problem

Limit retry amplification by admitting retries only from an allowance earned by successful original attempts in one explicit epoch.

The policy adds a fixed base allowance to an exact rational fraction of primary
successes. Retries already admitted are counted permanently within the epoch,
so a retry success cannot fund another retry. An admitted plan carries the
snapshot revision and a single successor for conditional storage before work
starts.

## When to Use

Use this planner after another policy has already established that one failed
operation is safe and useful to retry. The caller must read a coherent snapshot
for one named epoch and compare-and-swap the returned successor against the
expected revision before launching the retry.

This budget answers only an aggregate amplification question. It does not
classify failures, establish idempotency, limit attempts per operation, schedule
backoff, enforce deadlines, deduplicate events, or coordinate distributed
writers. Epoch creation, expiry, reset, and observation lag remain explicit
caller responsibilities.

## Implementation

```python
import re
from dataclasses import dataclass
from enum import StrEnum

_MAX_INT64 = 2**63 - 1
_MAX_RATIO_TERM = 1_000
_EPOCH = re.compile(r"[A-Za-z0-9._:-]{1,64}", re.ASCII).fullmatch


class RetryBudgetOutcome(StrEnum):
    ADMIT = "admit"
    REJECT_BUDGET_EXHAUSTED = "reject-budget-exhausted"
    REJECT_REVISION_EXHAUSTED = "reject-revision-exhausted"


@dataclass(frozen=True, slots=True)
class RetryBudgetSnapshot:
    epoch: str
    primary_successes: int
    retries_admitted: int
    revision: int


@dataclass(frozen=True, slots=True)
class RetryBudgetPolicy:
    base_allowance: int
    numerator: int
    denominator: int


@dataclass(frozen=True, slots=True)
class RetryAdmissionPlan:
    outcome: RetryBudgetOutcome
    allowance: int
    remaining_before: int
    expected_revision: int
    successor: RetryBudgetSnapshot | None


def _nonnegative_int64(value: object, *, field: str) -> int:
    if type(value) is not int:
        raise TypeError(f"{field} must be an exact integer")
    if not 0 <= value <= _MAX_INT64:
        raise ValueError(f"{field} is outside 0..2**63-1")
    return value


def _validated_snapshot(value: object) -> RetryBudgetSnapshot:
    if type(value) is not RetryBudgetSnapshot:
        raise TypeError("snapshot must be an exact RetryBudgetSnapshot")
    if type(value.epoch) is not str:
        raise TypeError("snapshot.epoch must be an exact string")
    if _EPOCH(value.epoch) is None:
        raise ValueError("snapshot.epoch is outside the closed token grammar")
    return RetryBudgetSnapshot(
        value.epoch,
        _nonnegative_int64(
            value.primary_successes,
            field="snapshot.primary_successes",
        ),
        _nonnegative_int64(
            value.retries_admitted,
            field="snapshot.retries_admitted",
        ),
        _nonnegative_int64(value.revision, field="snapshot.revision"),
    )


def _validated_policy(value: object) -> RetryBudgetPolicy:
    if type(value) is not RetryBudgetPolicy:
        raise TypeError("policy must be an exact RetryBudgetPolicy")
    base = _nonnegative_int64(value.base_allowance, field="policy.base_allowance")
    if type(value.numerator) is not int:
        raise TypeError("policy.numerator must be an exact integer")
    if type(value.denominator) is not int:
        raise TypeError("policy.denominator must be an exact integer")
    if not 0 <= value.numerator <= _MAX_RATIO_TERM:
        raise ValueError("policy.numerator is outside 0..1000")
    if not 1 <= value.denominator <= _MAX_RATIO_TERM:
        raise ValueError("policy.denominator is outside 1..1000")
    if value.numerator > value.denominator:
        raise ValueError("policy.numerator must not exceed its denominator")
    return RetryBudgetPolicy(base, value.numerator, value.denominator)


def plan_retry_admission(
    snapshot: RetryBudgetSnapshot,
    *,
    policy: RetryBudgetPolicy,
) -> RetryAdmissionPlan:
    state = _validated_snapshot(snapshot)
    rules = _validated_policy(policy)
    earned = state.primary_successes * rules.numerator // rules.denominator
    allowance = min(_MAX_INT64, rules.base_allowance + earned)
    remaining = max(0, allowance - state.retries_admitted)

    if remaining == 0:
        return RetryAdmissionPlan(
            RetryBudgetOutcome.REJECT_BUDGET_EXHAUSTED,
            allowance,
            0,
            state.revision,
            None,
        )
    if state.revision == _MAX_INT64:
        return RetryAdmissionPlan(
            RetryBudgetOutcome.REJECT_REVISION_EXHAUSTED,
            allowance,
            remaining,
            state.revision,
            None,
        )

    successor = RetryBudgetSnapshot(
        state.epoch,
        state.primary_successes,
        state.retries_admitted + 1,
        state.revision + 1,
    )
    return RetryAdmissionPlan(
        RetryBudgetOutcome.ADMIT,
        allowance,
        remaining,
        state.revision,
        successor,
    )
```

## Example

```python
checked = 0
for primary_successes in range(5):
    for retries_admitted in range(5):
        for base_allowance in range(3):
            for denominator in range(1, 5):
                for numerator in range(denominator + 1):
                    snapshot = RetryBudgetSnapshot(
                        "window-1",
                        primary_successes,
                        retries_admitted,
                        7,
                    )
                    policy = RetryBudgetPolicy(
                        base_allowance,
                        numerator,
                        denominator,
                    )
                    plan = plan_retry_admission(snapshot, policy=policy)
                    expected_allowance = (
                        base_allowance
                        + primary_successes * numerator // denominator
                    )
                    assert plan.allowance == expected_allowance
                    assert plan.remaining_before == max(
                        0,
                        expected_allowance - retries_admitted,
                    )
                    assert (plan.outcome is RetryBudgetOutcome.ADMIT) == (
                        retries_admitted < expected_allowance
                    )
                    if plan.successor is not None:
                        assert plan.successor == RetryBudgetSnapshot(
                            "window-1",
                            primary_successes,
                            retries_admitted + 1,
                            8,
                        )
                    checked += 1

boundary = plan_retry_admission(
    RetryBudgetSnapshot("window-max", _MAX_INT64, 0, 9),
    policy=RetryBudgetPolicy(_MAX_INT64, 1, 1),
)
revision_exhausted = plan_retry_admission(
    RetryBudgetSnapshot("window-max", 10, 0, _MAX_INT64),
    policy=RetryBudgetPolicy(0, 1, 2),
)
same_snapshot = RetryBudgetSnapshot("race", 10, 1, 20)
first_plan = plan_retry_admission(
    same_snapshot,
    policy=RetryBudgetPolicy(0, 1, 2),
)
second_plan = plan_retry_admission(
    same_snapshot,
    policy=RetryBudgetPolicy(0, 1, 2),
)

assert (
    checked == 1_050
    and boundary.allowance == _MAX_INT64
    and boundary.outcome is RetryBudgetOutcome.ADMIT
    and revision_exhausted.outcome
    is RetryBudgetOutcome.REJECT_REVISION_EXHAUSTED
    and revision_exhausted.successor is None
    and first_plan == second_plan
    and first_plan.expected_revision == 20
)
```

## Trade-offs and Limitations

Planning takes constant time and memory. Python's unbounded intermediate
integer arithmetic computes the fraction exactly, after which the allowance is
saturated into the declared signed-64-bit domain. A maximum revision fails
closed instead of saturating, because a saturated revision could not protect a
conditional update from reapplication.

Two callers can compute identical plans from the same snapshot; only one may
win the required compare-and-swap. The snapshot counts admissions, not retry
completions, because reserving budget before execution prevents concurrent
callers from overspending it. Changing policy or resetting an epoch can reduce
the current allowance below the historical admitted count, in which case the
planner simply rejects further retries.

## Related Snippets

<!-- catalog:related:start -->
- [Resolve a Guarded Retry Decision from Operation and Worker Policies](resolve-a-guarded-retry-decision-from-operation-and-worker-policies.md)
- [Plan One Discrete Token-Bucket Admission from an Explicit Tick Snapshot](plan-one-discrete-token-bucket-admission-from-an-explicit-tick-snapshot.md)
- [Compare and Swap a Versioned SQLite Setting with One Conditional Update](../storage-databases/compare-and-swap-a-versioned-sqlite-setting-with-one-conditional-update.md)
<!-- catalog:related:end -->
