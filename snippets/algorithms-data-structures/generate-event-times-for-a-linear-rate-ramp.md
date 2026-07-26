---
title: "Generate Event Times for a Linear Rate Ramp"
snippet_type: algorithm
use_cases:
  - automation
  - performance-optimization
tested_python:
  - "3.14"
dependencies: []
related:
  - ../reliability-resilience/assign-stable-schedule-slots-with-a-digest.md
  - ../observability-operations/count-values-in-fixed-upper-bound-bins.md
---

# Generate Event Times for a Linear Rate Ramp

## Idea and Problem

Generate deterministic event times by inverting the cumulative integral of a rate that changes linearly over a fixed duration.

For each one-based event ordinal, the algorithm finds the time at which the
integrated rate reaches that integer. It emits ordinals through the floor of
the total area under the rate curve, so there is no artificial event at time
zero. A rationalized quadratic root avoids the cancellation in directly
subtracting nearly equal floating-point values.

## When to Use

Use this schedule to construct a finite ramp-up or ramp-down plan before a
simulation, benchmark, or deterministic replay. Rates are events per second
and time offsets are seconds. This is a fluid-rate plan, not a stochastic
arrival process; use a Poisson or domain-specific model when random arrival
variance matters.

## Implementation

```python
import math


def _finite_number(value: object, *, name: str, positive: bool) -> float:
    if isinstance(value, bool) or not isinstance(value, (int, float)):
        raise TypeError(f"{name} must be a real number")
    try:
        number = float(value)
    except OverflowError as error:
        raise ValueError(f"{name} must be finite") from error
    if not math.isfinite(number):
        raise ValueError(f"{name} must be finite")
    if positive and number <= 0.0:
        raise ValueError(f"{name} must be positive")
    if not positive and number < 0.0:
        raise ValueError(f"{name} must be non-negative")
    return number


def linear_rate_event_times(
    start_rate: float,
    end_rate: float,
    duration: float,
    *,
    max_events: int = 100_000,
) -> tuple[float, ...]:
    start = _finite_number(start_rate, name="start_rate", positive=False)
    end = _finite_number(end_rate, name="end_rate", positive=False)
    span = _finite_number(duration, name="duration", positive=True)
    if isinstance(max_events, bool) or not isinstance(max_events, int):
        raise TypeError("max_events must be an integer")
    if max_events <= 0:
        raise ValueError("max_events must be positive")

    total = (start + end) * span / 2.0
    if not math.isfinite(total):
        raise ValueError("the integrated event count is too large")
    event_count = math.floor(total)
    if event_count > max_events:
        raise ValueError("the schedule exceeds max_events")
    if event_count == 0:
        return ()

    slope = (end - start) / span
    if slope == 0.0:
        return tuple(ordinal / start for ordinal in range(1, event_count + 1))

    times: list[float] = []
    for ordinal in range(1, event_count + 1):
        discriminant = start * start + 2.0 * slope * ordinal
        scale = max(start * start, abs(2.0 * slope * ordinal), 1.0)
        if discriminant < 0.0 and math.isclose(
            discriminant,
            0.0,
            rel_tol=0.0,
            abs_tol=1e-12 * scale,
        ):
            discriminant = 0.0
        if discriminant < 0.0:
            raise ArithmeticError("event threshold has no real time")

        denominator = start + math.sqrt(discriminant)
        if denominator <= 0.0:
            raise ArithmeticError("event threshold has no non-negative time")
        event_time = 2.0 * ordinal / denominator
        if event_time > span and math.isclose(event_time, span, rel_tol=1e-12):
            event_time = span
        if not 0.0 <= event_time <= span:
            raise ArithmeticError("event time fell outside the schedule")
        times.append(event_time)

    return tuple(times)
```

## Example

```python
constant = linear_rate_event_times(2.0, 2.0, 2.0)
ramp_up = linear_rate_event_times(0.0, 4.0, 2.0)
ramp_down = linear_rate_event_times(4.0, 0.0, 2.0)
below_one = linear_rate_event_times(0.0, 0.5, 2.0)


def rounded(values: tuple[float, ...]) -> tuple[float, ...]:
    return tuple(round(value, 6) for value in values)


assert (
    rounded(constant),
    rounded(ramp_up),
    rounded(ramp_down),
    below_one,
    all(0.0 <= left < right <= 2.0 for left, right in zip((0.0,) + ramp_up, ramp_up)),
) == (
    (0.5, 1.0, 1.5, 2.0),
    (1.0, 1.414214, 1.732051, 2.0),
    (0.267949, 0.585786, 1.0, 2.0),
    (),
    True,
)
```

## Trade-offs and Limitations

The function materializes one float per event and rejects plans above
`max_events`; stream or aggregate much larger schedules. Floating-point area
near an integer can change the final `floor`, and very large or ill-scaled
rates can lose precision despite the stable root formula. Returned offsets do
not account for scheduler resolution, work duration, backpressure, or clock
drift. The plan says when events should be due; it never sleeps, rate-limits,
or guarantees that a runtime can execute them on time.

## Related Snippets

<!-- catalog:related:start -->
- [Assign Stable Schedule Slots with a Digest](../reliability-resilience/assign-stable-schedule-slots-with-a-digest.md)
- [Count Values in Fixed Upper-Bound Bins](../observability-operations/count-values-in-fixed-upper-bound-bins.md)
<!-- catalog:related:end -->
