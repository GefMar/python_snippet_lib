---
title: "Plan One Discrete Token-Bucket Admission from an Explicit Tick Snapshot"
snippet_type: algorithm
use_cases:
  - concurrency-control
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - plan-one-periodic-service-step-from-an-explicit-monotonic-snapshot.md
  - plan-a-versioned-transition-for-the-current-workflow-attempt.md
  - ../storage-databases/compare-and-swap-a-versioned-sqlite-setting-with-one-conditional-update.md
---

# Plan One Discrete Token-Bucket Admission from an Explicit Tick Snapshot

## Idea and Problem

Reduce one immutable integer token-bucket snapshot and one explicit tick into a closed admission outcome and a revision-bound successor state.

Refill saturates at capacity without multiplying a very large elapsed interval.
Every outcome fixes the state at `now_tick` and advances its revision, allowing
the caller to conditionally store the successor before starting admitted work.

## When to Use

Use this algorithm when one token represents one indivisible unit of capacity
and a caller can read and conditionally replace a coherent bucket snapshot.
All quantities use exact nonnegative integers in a signed 64-bit range. Ticks
must share one caller-defined monotonic epoch; wall-clock timestamps and mixed
epochs are unsuitable.

Admission is closed to `ADMIT`, `REJECT_INSUFFICIENT`, and
`REJECT_COST_EXCEEDS_CAPACITY`. A cost exactly equal to the refilled token count
is admitted. A zero refill rate is valid and adds no tokens, and the model has
no floats, fractional rate, or fractional carry.

## Implementation

```python
from dataclasses import dataclass
from enum import StrEnum

_MAX_INT64 = 2**63 - 1


class AdmissionOutcome(StrEnum):
    ADMIT = "admit"
    REJECT_INSUFFICIENT = "reject-insufficient"
    REJECT_COST_EXCEEDS_CAPACITY = "reject-cost-exceeds-capacity"


@dataclass(frozen=True, slots=True)
class TokenBucketSnapshot:
    capacity: int
    tokens: int
    refill_per_tick: int
    last_tick: int
    revision: int


@dataclass(frozen=True, slots=True)
class TokenBucketAdmissionPlan:
    expected_revision: int
    outcome: AdmissionOutcome
    successor: TokenBucketSnapshot


class StaleTokenBucketTickError(ValueError):
    pass


def _nonnegative_int64(value: object, *, field: str) -> int:
    if type(value) is not int:
        raise TypeError(f"{field} must be an exact integer")
    if not 0 <= value <= _MAX_INT64:
        raise ValueError(f"{field} is outside the supported range")
    return value


def _validated_snapshot(value: object) -> TokenBucketSnapshot:
    if type(value) is not TokenBucketSnapshot:
        raise TypeError("snapshot must be an exact TokenBucketSnapshot")

    capacity = _nonnegative_int64(value.capacity, field="snapshot.capacity")
    tokens = _nonnegative_int64(value.tokens, field="snapshot.tokens")
    refill = _nonnegative_int64(
        value.refill_per_tick,
        field="snapshot.refill_per_tick",
    )
    last_tick = _nonnegative_int64(value.last_tick, field="snapshot.last_tick")
    revision = _nonnegative_int64(value.revision, field="snapshot.revision")
    if tokens > capacity:
        raise ValueError("snapshot.tokens must not exceed snapshot.capacity")
    return TokenBucketSnapshot(capacity, tokens, refill, last_tick, revision)


def _positive_cost(value: object) -> int:
    cost = _nonnegative_int64(value, field="cost")
    if cost == 0:
        raise ValueError("cost must be positive")
    return cost


def _saturating_refill(
    *,
    capacity: int,
    tokens: int,
    refill_per_tick: int,
    elapsed: int,
) -> int:
    room = capacity - tokens
    if room == 0 or refill_per_tick == 0 or elapsed == 0:
        return tokens

    ticks_to_fill, remainder = divmod(room, refill_per_tick)
    if remainder != 0:
        ticks_to_fill += 1
    if elapsed >= ticks_to_fill:
        return capacity

    # This branch proves elapsed * refill_per_tick is smaller than room.
    return tokens + elapsed * refill_per_tick


def plan_one_token_bucket_admission(
    snapshot: TokenBucketSnapshot,
    *,
    now_tick: int,
    cost: int,
) -> TokenBucketAdmissionPlan:
    state = _validated_snapshot(snapshot)
    now = _nonnegative_int64(now_tick, field="now_tick")
    requested = _positive_cost(cost)
    if now < state.last_tick:
        raise StaleTokenBucketTickError("now_tick precedes snapshot.last_tick")
    if state.revision == _MAX_INT64:
        raise OverflowError("snapshot revision cannot be advanced")

    available = _saturating_refill(
        capacity=state.capacity,
        tokens=state.tokens,
        refill_per_tick=state.refill_per_tick,
        elapsed=now - state.last_tick,
    )
    if requested > state.capacity:
        outcome = AdmissionOutcome.REJECT_COST_EXCEEDS_CAPACITY
        remaining = available
    elif requested <= available:
        outcome = AdmissionOutcome.ADMIT
        remaining = available - requested
    else:
        outcome = AdmissionOutcome.REJECT_INSUFFICIENT
        remaining = available

    successor = TokenBucketSnapshot(
        capacity=state.capacity,
        tokens=remaining,
        refill_per_tick=state.refill_per_tick,
        last_tick=now,
        revision=state.revision + 1,
    )
    return TokenBucketAdmissionPlan(state.revision, outcome, successor)
```

## Example

```python
equal = plan_one_token_bucket_admission(
    TokenBucketSnapshot(8, 3, 0, 10, 4),
    now_tick=10,
    cost=3,
)
insufficient = plan_one_token_bucket_admission(
    TokenBucketSnapshot(8, 2, 0, 10, 7),
    now_tick=12,
    cost=3,
)
oversized = plan_one_token_bucket_admission(
    TokenBucketSnapshot(8, 2, 2, 0, 9),
    now_tick=_MAX_INT64,
    cost=9,
)

try:
    plan_one_token_bucket_admission(
        TokenBucketSnapshot(8, 3, 1, 10, 2),
        now_tick=9,
        cost=1,
    )
except StaleTokenBucketTickError:
    stale_rejected = True
else:
    stale_rejected = False

assert (
    equal,
    insufficient.outcome,
    insufficient.successor,
    oversized.outcome,
    oversized.successor.tokens,
    oversized.expected_revision,
    stale_rejected,
) == (
    TokenBucketAdmissionPlan(
        4,
        AdmissionOutcome.ADMIT,
        TokenBucketSnapshot(8, 0, 0, 10, 5),
    ),
    AdmissionOutcome.REJECT_INSUFFICIENT,
    TokenBucketSnapshot(8, 2, 0, 12, 8),
    AdmissionOutcome.REJECT_COST_EXCEEDS_CAPACITY,
    8,
    9,
    True,
)
```

## Trade-offs and Limitations

Validation rejects booleans, floats, negative values, values above
`2**63 - 1`, over-capacity token counts, stale ticks, zero costs, and revisions
that cannot advance. Saturation first divides the remaining capacity by the
refill rate; multiplication occurs only after proving the product is smaller
than that bounded capacity gap. A zero refill rate bypasses division.

Every result advances the revision and moves `last_tick` to `now_tick`, even
when admission is rejected, so the successor materializes the refill decision
for that snapshot. The function reads no clock and performs no sleep, lock,
I/O, or mutation. The caller owns concurrency: it must apply the successor only
if stored revision still equals `expected_revision`, discard a stale plan, and
start external work only after an `ADMIT` successor is accepted. This protocol
does not make the external work transactional or retry-safe after a crash.

## Related Snippets

<!-- catalog:related:start -->
- [Plan One Periodic-Service Step from an Explicit Monotonic Snapshot](plan-one-periodic-service-step-from-an-explicit-monotonic-snapshot.md)
- [Plan a Versioned Transition for the Current Workflow Attempt](plan-a-versioned-transition-for-the-current-workflow-attempt.md)
- [Compare and Swap a Versioned SQLite Setting with One Conditional Update](../storage-databases/compare-and-swap-a-versioned-sqlite-setting-with-one-conditional-update.md)
<!-- catalog:related:end -->
