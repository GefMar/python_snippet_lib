---
title: "Detect Wall Clock Divergence from Bracketed Monotonic Readings"
snippet_type: recipe
use_cases:
  - observability
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../reliability-resilience/plan-one-periodic-service-step-from-an-explicit-monotonic-snapshot.md
  - ../reliability-resilience/propagate-a-monotonic-deadline-with-contextvar.md
---

# Detect Wall Clock Divergence from Bracketed Monotonic Readings

## Idea and Problem

Compare elapsed wall time against the range allowed by two bracketed monotonic readings.

Read the monotonic clock immediately before and after each wall-clock
sample. The true monotonic instant of each wall read lies between its
brackets. Compare the resulting possible elapsed times rather than treating
each wall read as simultaneous with one monotonic read.

## When to Use

Use this to diagnose divergence between clocks on the same running system.
Keep the readings close together within each bracket and allow for normal
drift and resolution when choosing a tolerance. Use a monotonic clock
directly for deadlines.

## Implementation

```python
from dataclasses import dataclass
from time import monotonic_ns, time_ns
from typing import Literal


@dataclass(frozen=True, slots=True)
class ClockSample:
    before_ns: int
    wall_ns: int
    after_ns: int


def sample_clocks() -> ClockSample:
    before = monotonic_ns()
    wall = time_ns()
    return ClockSample(before, wall, monotonic_ns())


def compare_elapsed(
    first: ClockSample, second: ClockSample, *, tolerance_ns: int
) -> Literal["wall-ahead", "wall-behind", "compatible"]:
    if not first.before_ns <= first.after_ns <= second.before_ns <= second.after_ns:
        raise ValueError("expected ordered, non-overlapping brackets")
    if tolerance_ns < 0:
        raise ValueError("tolerance must be non-negative")

    minimum = second.before_ns - first.after_ns
    maximum = second.after_ns - first.before_ns
    wall_elapsed = second.wall_ns - first.wall_ns
    if wall_elapsed > maximum + tolerance_ns:
        return "wall-ahead"
    if wall_elapsed < minimum - tolerance_ns:
        return "wall-behind"
    return "compatible"
```

## Example

```python
first = ClockSample(1_000, 50_000, 1_004)
# The possible monotonic interval is 96 through 104 ns.
results = tuple(
    compare_elapsed(first, ClockSample(1_100, 50_000 + delta, 1_104), tolerance_ns=3)
    for delta in (108, 107, 93, 92)
)
assert results == ("wall-ahead", "compatible", "compatible", "wall-behind")
```

## Trade-offs and Limitations

A compatible reading only means that the measurements could agree within
the bounds. A long scheduling pause makes a bracket wide and can hide a
real difference. Repeat samples with narrow brackets when resolution matters.

The result describes a measured difference, without identifying its cause.
Clock adjustments, drift and platform differences during suspend can affect
the comparison. Never subtract monotonic readings from separate systems.

## Related Snippets

<!-- catalog:related:start -->
- [Plan One Periodic-Service Step from an Explicit Monotonic Snapshot](../reliability-resilience/plan-one-periodic-service-step-from-an-explicit-monotonic-snapshot.md)
- [Propagate a Monotonic Deadline with ContextVar](../reliability-resilience/propagate-a-monotonic-deadline-with-contextvar.md)
<!-- catalog:related:end -->
