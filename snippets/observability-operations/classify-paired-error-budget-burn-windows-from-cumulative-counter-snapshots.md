---
title: "Classify Paired Error-Budget Burn Windows from Cumulative Counter Snapshots"
snippet_type: algorithm
use_cases:
  - observability
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - measure-cache-hit-ratios-from-monotonic-counter-snapshots.md
  - compute-a-validated-delta-between-cumulative-histogram-snapshots.md
  - compute-an-exact-apdex-score-from-already-classified-counts.md
---

# Classify Paired Error-Budget Burn Windows from Cumulative Counter Snapshots

## Idea and Problem

Compare exact error-budget burn in nested long and short counter windows without hiding resets, empty traffic, or threshold ties behind floating-point arithmetic.

Three coherent cumulative snapshots define both windows: a long-window start, a
later short-window start, and one shared end. Each error ratio is divided by the
error budget derived from the caller-supplied SLO target. Requiring both burn
rates to meet their own thresholds lets sustained and recent evidence
contribute to one explicit classification.

## When to Use

Use this calculation after a metrics system has captured coherent monotonic
request and failure counters for two nested observation windows. Exact
`Fraction` inputs are useful in tests, alert-rule evaluation, and configuration
validation where equality at a threshold must be deterministic.

Keep collection, window duration, threshold selection, multi-instance
aggregation, and notification outside this function. Counter resets or
incomparable snapshots must be separated by epoch metadata rather than repaired
by guessing.

## Implementation

```python
from dataclasses import dataclass
from enum import StrEnum
from fractions import Fraction

_MAX_COUNTER = (1 << 63) - 1
_MAX_FRACTION_COMPONENT_BITS = 256


class BurnDecision(StrEnum):
    NO_TRAFFIC = "no-traffic"
    BELOW_THRESHOLD = "below-threshold"
    ALERT = "alert"


def _counter(value: object, *, name: str) -> int:
    if type(value) is not int:
        raise TypeError(f"{name} must be an exact integer")
    if not 0 <= value <= _MAX_COUNTER:
        raise ValueError(f"{name} is outside the supported range")
    return value


def _bounded_fraction(value: object, *, name: str) -> Fraction:
    if type(value) is not Fraction:
        raise TypeError(f"{name} must be an exact Fraction")
    if (
        abs(value.numerator).bit_length() > _MAX_FRACTION_COMPONENT_BITS
        or value.denominator.bit_length() > _MAX_FRACTION_COMPONENT_BITS
    ):
        raise ValueError(f"{name} components exceed the supported size")
    return value


@dataclass(frozen=True, slots=True)
class ErrorCounterSnapshot:
    requests: int
    failed: int

    def __post_init__(self) -> None:
        _counter(self.requests, name="requests")
        _counter(self.failed, name="failed")
        if self.failed > self.requests:
            raise ValueError("failed must not exceed requests")


@dataclass(frozen=True, slots=True)
class BurnWindow:
    requests: int
    failed: int
    error_ratio: Fraction | None
    burn_rate: Fraction | None


@dataclass(frozen=True, slots=True)
class PairedBurnEvaluation:
    long_window: BurnWindow
    short_window: BurnWindow
    decision: BurnDecision


def _counter_delta(
    start: ErrorCounterSnapshot,
    end: ErrorCounterSnapshot,
) -> tuple[int, int]:
    requests = end.requests - start.requests
    failed = end.failed - start.failed
    if requests < 0 or failed < 0 or failed > requests:
        raise ValueError(
            "counter snapshots do not form a valid monotonic window"
        )
    return requests, failed


def _measure_window(
    start: ErrorCounterSnapshot,
    end: ErrorCounterSnapshot,
    *,
    error_budget: Fraction,
) -> BurnWindow:
    requests, failed = _counter_delta(start, end)
    if requests == 0:
        return BurnWindow(
            requests=0,
            failed=0,
            error_ratio=None,
            burn_rate=None,
        )
    error_ratio = Fraction(failed, requests)
    return BurnWindow(
        requests=requests,
        failed=failed,
        error_ratio=error_ratio,
        burn_rate=error_ratio / error_budget,
    )


def classify_paired_error_budget_burn(
    long_start: ErrorCounterSnapshot,
    short_start: ErrorCounterSnapshot,
    end: ErrorCounterSnapshot,
    *,
    slo_target: Fraction,
    long_threshold: Fraction,
    short_threshold: Fraction,
) -> PairedBurnEvaluation:
    if (
        type(long_start) is not ErrorCounterSnapshot
        or type(short_start) is not ErrorCounterSnapshot
        or type(end) is not ErrorCounterSnapshot
    ):
        raise TypeError("snapshots must be exact ErrorCounterSnapshot values")

    target = _bounded_fraction(slo_target, name="slo_target")
    long_limit = _bounded_fraction(long_threshold, name="long_threshold")
    short_limit = _bounded_fraction(short_threshold, name="short_threshold")
    if not 0 <= target < 1:
        raise ValueError("slo_target must be in the half-open interval [0, 1)")
    if long_limit <= 0 or short_limit <= 0:
        raise ValueError("burn thresholds must be positive")

    _counter_delta(long_start, short_start)
    _counter_delta(short_start, end)
    error_budget = 1 - target
    long_window = _measure_window(
        long_start,
        end,
        error_budget=error_budget,
    )
    short_window = _measure_window(
        short_start,
        end,
        error_budget=error_budget,
    )

    if long_window.burn_rate is None or short_window.burn_rate is None:
        decision = BurnDecision.NO_TRAFFIC
    elif (
        long_window.burn_rate >= long_limit
        and short_window.burn_rate >= short_limit
    ):
        decision = BurnDecision.ALERT
    else:
        decision = BurnDecision.BELOW_THRESHOLD

    return PairedBurnEvaluation(
        long_window=long_window,
        short_window=short_window,
        decision=decision,
    )
```

## Example

```python
evaluation = classify_paired_error_budget_burn(
    ErrorCounterSnapshot(requests=0, failed=0),
    ErrorCounterSnapshot(requests=800, failed=20),
    ErrorCounterSnapshot(requests=1000, failed=30),
    slo_target=Fraction(99, 100),
    long_threshold=Fraction(3, 1),
    short_threshold=Fraction(5, 1),
)

no_recent_traffic = classify_paired_error_budget_burn(
    ErrorCounterSnapshot(requests=0, failed=0),
    ErrorCounterSnapshot(requests=100, failed=1),
    ErrorCounterSnapshot(requests=100, failed=1),
    slo_target=Fraction(99, 100),
    long_threshold=Fraction(1, 1),
    short_threshold=Fraction(1, 1),
)

try:
    classify_paired_error_budget_burn(
        ErrorCounterSnapshot(requests=100, failed=4),
        ErrorCounterSnapshot(requests=90, failed=3),
        ErrorCounterSnapshot(requests=120, failed=5),
        slo_target=Fraction(99, 100),
        long_threshold=Fraction(1, 1),
        short_threshold=Fraction(1, 1),
    )
except ValueError:
    reset_rejected = True
else:
    reset_rejected = False

assert (
    evaluation.long_window.burn_rate,
    evaluation.short_window.burn_rate,
    evaluation.decision,
    no_recent_traffic.short_window,
    no_recent_traffic.decision,
    reset_rejected,
) == (
    Fraction(3, 1),
    Fraction(5, 1),
    BurnDecision.ALERT,
    BurnWindow(0, 0, None, None),
    BurnDecision.NO_TRAFFIC,
    True,
)
```

## Trade-offs and Limitations

The calculation uses constant space and a fixed amount of exact rational
arithmetic. Counter values are capped at the non-negative signed 64-bit range;
each caller-supplied fraction is capped to 256-bit numerator and denominator
components. `Fraction` avoids threshold rounding but costs more than native
floating-point operations.

A zero-request window has neither an error ratio nor a burn rate, so both
fields are `None` and the pair is classified as `no-traffic`. The inclusive
threshold comparison is a local policy, not a recommendation for particular
numbers or durations. The function assumes one coherent counter epoch and does
not capture timestamps, establish window nesting by time, detect delayed
samples, forecast budget exhaustion, or deliver an alert.

## Related Snippets

<!-- catalog:related:start -->
- [Measure Cache Hit Ratios from Monotonic Counter Snapshots](measure-cache-hit-ratios-from-monotonic-counter-snapshots.md)
- [Compute a Validated Delta Between Cumulative Histogram Snapshots](compute-a-validated-delta-between-cumulative-histogram-snapshots.md)
- [Compute an Exact Apdex Score from Already Classified Counts](compute-an-exact-apdex-score-from-already-classified-counts.md)
<!-- catalog:related:end -->
