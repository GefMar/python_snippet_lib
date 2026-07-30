---
title: "Plan Full-Jitter Exponential Retry Delays Under a Cumulative Wait Budget"
snippet_type: algorithm
use_cases:
  - resource-management
  - retry-recovery
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - poll-with-deterministic-capped-backoff-under-one-monotonic-deadline.md
  - plan-one-retry-admission-from-a-success-coupled-budget-snapshot.md
  - resolve-a-guarded-retry-decision-from-operation-and-worker-policies.md
---

# Plan Full-Jitter Exponential Retry Delays Under a Cumulative Wait Budget

## Idea and Problem

Choose bounded discrete full-jitter delays without allowing their cumulative wait to exceed one explicit budget.

Each zero-based retry starts with an exponentially growing ceiling capped by
policy. The remaining cumulative budget caps it again, and one injected
`randbelow` call selects an integer delay from zero through that ceiling. The
returned immutable plan separates schedule selection from sleeping and retry
execution.

## When to Use

Use this planner after an operation has already been classified as safe and
eligible to retry. It suits bounded clients, workers, and tests that need
per-retry jitter, an auditable total wait allowance, and deterministic random
injection.

Use a deadline-aware retry library when attempts have variable duration or
must be cancelled. Apply server hints, rate-limit policy, idempotency checks,
and retryable-error classification before calling this function; a randomized
delay cannot make an unsafe retry safe.

## Implementation

```python
from collections.abc import Callable
from dataclasses import dataclass

_MAX_BASE_DELAY_MS = 60_000
_MAX_CAP_DELAY_MS = 3_600_000
_MAX_RETRY_COUNT = 32
_MAX_WAIT_BUDGET_MS = 3_600_000


@dataclass(frozen=True, slots=True)
class RetryDelay:
    retry_index: int
    effective_ceiling_ms: int
    delay_ms: int


@dataclass(frozen=True, slots=True)
class RetryDelayPlan:
    delays: tuple[RetryDelay, ...]
    total_wait_ms: int
    remaining_wait_budget_ms: int
    omitted_retry_count: int


def _bounded_integer(
    value: object,
    *,
    name: str,
    minimum: int,
    maximum: int,
) -> int:
    if type(value) is not int:
        raise TypeError(f"{name} must be an exact integer")
    if not minimum <= value <= maximum:
        raise ValueError(f"{name} is outside the supported range")
    return value


def plan_full_jitter_retry_delays(
    *,
    base_delay_ms: int,
    cap_delay_ms: int,
    retry_count: int,
    wait_budget_ms: int,
    randbelow: Callable[[int], int],
) -> RetryDelayPlan:
    """Return discrete full-jitter delays without exceeding the wait budget."""
    base_delay_ms = _bounded_integer(
        base_delay_ms,
        name="base_delay_ms",
        minimum=1,
        maximum=_MAX_BASE_DELAY_MS,
    )
    cap_delay_ms = _bounded_integer(
        cap_delay_ms,
        name="cap_delay_ms",
        minimum=1,
        maximum=_MAX_CAP_DELAY_MS,
    )
    if cap_delay_ms < base_delay_ms:
        raise ValueError("cap_delay_ms must be at least base_delay_ms")
    retry_count = _bounded_integer(
        retry_count,
        name="retry_count",
        minimum=0,
        maximum=_MAX_RETRY_COUNT,
    )
    wait_budget_ms = _bounded_integer(
        wait_budget_ms,
        name="wait_budget_ms",
        minimum=0,
        maximum=_MAX_WAIT_BUDGET_MS,
    )
    if not callable(randbelow):
        raise TypeError("randbelow must be callable")

    remaining = wait_budget_ms
    exponential_ceiling = base_delay_ms
    delays: list[RetryDelay] = []

    for retry_index in range(retry_count):
        if remaining == 0:
            break

        ceiling = min(exponential_ceiling, remaining)
        delay = randbelow(ceiling + 1)
        if type(delay) is not int:
            raise TypeError("randbelow result must be an exact integer")
        if not 0 <= delay <= ceiling:
            raise ValueError("randbelow result is outside the requested range")

        delays.append(RetryDelay(retry_index, ceiling, delay))
        remaining -= delay
        exponential_ceiling = min(
            cap_delay_ms,
            exponential_ceiling * 2,
        )

    return RetryDelayPlan(
        delays=tuple(delays),
        total_wait_ms=wait_budget_ms - remaining,
        remaining_wait_budget_ms=remaining,
        omitted_retry_count=retry_count - len(delays),
    )


```

## Example

```python
class ScriptedRandbelow:
    def __init__(self, values: tuple[int, ...]) -> None:
        self.values = iter(values)
        self.stops: list[int] = []

    def __call__(self, stop: int) -> int:
        self.stops.append(stop)
        value = next(self.values)
        assert 0 <= value < stop
        return value


draw = ScriptedRandbelow((100, 0, 150))
budget_exhausted = plan_full_jitter_retry_delays(
    base_delay_ms=100,
    cap_delay_ms=400,
    retry_count=5,
    wait_budget_ms=250,
    randbelow=draw,
)

zero_draws = ScriptedRandbelow((0, 0, 0, 0))
all_retries = plan_full_jitter_retry_delays(
    base_delay_ms=3,
    cap_delay_ms=5,
    retry_count=4,
    wait_budget_ms=100,
    randbelow=zero_draws,
)

never_called = ScriptedRandbelow(())
no_budget = plan_full_jitter_retry_delays(
    base_delay_ms=1,
    cap_delay_ms=1,
    retry_count=3,
    wait_budget_ms=0,
    randbelow=never_called,
)

assert budget_exhausted == RetryDelayPlan(
    delays=(
        RetryDelay(0, 100, 100),
        RetryDelay(1, 150, 0),
        RetryDelay(2, 150, 150),
    ),
    total_wait_ms=250,
    remaining_wait_budget_ms=0,
    omitted_retry_count=2,
)
assert draw.stops == [101, 151, 151]
assert tuple(delay.effective_ceiling_ms for delay in all_retries.delays) == (
    3,
    5,
    5,
    5,
)
assert zero_draws.stops == [4, 6, 6, 6]
assert no_budget.omitted_retry_count == 3 and never_called.stops == []
```

## Trade-offs and Limitations

Planning takes `O(r)` time and output space for at most 32 requested retries.
Saturating the exponential ceiling after every record avoids constructing an
unbounded power. Integer milliseconds and the inclusive draw rule make every
boundary deterministic and keep the cumulative total at or below the budget.

A zero delay is valid full jitter. It does not consume budget, but the explicit
retry count still bounds the loop. `omitted_retry_count` counts only requested
records not emitted because the budget was already zero; exhausting the retry
count with budget left produces zero omissions.

The injected callable is trusted local code. It receives exactly one positive
exclusive stop for every emitted record; its exceptions propagate, while a
boolean, non-integer, negative, or too-large result is rejected. The planner
cannot test whether accepted draws are uniform, so that property belongs to the
callback contract. It does not provide cryptographic randomness, fairness,
sleeps, attempts, deadlines, server-hint handling, persistence, or concurrency
coordination.

## Related Snippets

<!-- catalog:related:start -->
- [Poll with Deterministic Capped Backoff Under One Monotonic Deadline](poll-with-deterministic-capped-backoff-under-one-monotonic-deadline.md)
- [Plan One Retry Admission from a Success-Coupled Budget Snapshot](plan-one-retry-admission-from-a-success-coupled-budget-snapshot.md)
- [Resolve a Guarded Retry Decision from Operation and Worker Policies](resolve-a-guarded-retry-decision-from-operation-and-worker-policies.md)
<!-- catalog:related:end -->
