---
title: "Summarize an Irregular Integer Gauge with Expiry"
snippet_type: algorithm
use_cases:
  - data-transformation
  - observability
tested_python:
  - "3.14"
dependencies: []
related:
  - integrate-regular-rate-samples-across-explicit-time-buckets.md
  - measure-time-in-a-state-within-a-half-open-window.md
---

# Summarize an Irregular Integer Gauge with Expiry

## Idea and Problem

Compute how much of a reporting window has a usable gauge value and its exact time-weighted average over that coverage.

An irregular gauge observation lasts until the next observation or until its
maximum age expires. Weight each value by the length of its usable interval.
This keeps an unobserved gap distinct from a genuine zero reading and prevents
a burst of closely spaced readings from dominating the average.

## When to Use

Use this for stepwise gauges on a single integer time axis, such as counts
from polls with occasional gaps. Provide the last observation before the window
if it might still be fresh at the window start. Supply trustworthy values in
strictly increasing timestamp order and choose an expiry that fits the polling
policy.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction


@dataclass(frozen=True, slots=True)
class GaugeWindow:
    covered_ticks: int
    integral: int

    @property
    def mean(self) -> Fraction | None:
        if self.covered_ticks == 0:
            return None
        return Fraction(self.integral, self.covered_ticks)


def summarize_gauge(
    samples: tuple[tuple[int, int], ...],
    start: int,
    end: int,
    max_age: int,
) -> GaugeWindow:
    """Integrate fresh observations over [start, end)."""
    if end < start:
        raise ValueError("end must not precede start")
    if max_age <= 0:
        raise ValueError("max_age must be positive")

    covered = 0
    integral = 0
    for index, (at, value) in enumerate(samples):
        if index + 1 < len(samples):
            next_at = samples[index + 1][0]
            if next_at <= at:
                raise ValueError("sample timestamps must be strictly increasing")
        else:
            next_at = end

        left = max(start, at)
        right = min(end, next_at, at + max_age)
        if left < right:
            duration = right - left
            covered += duration
            integral += value * duration

    return GaugeWindow(covered, integral)
```

## Example

```python
# A zero observation contributes to coverage. An expired interval does not.
samples = ((-2, 6), (2, 0), (7, 4), (10, 99))
report = summarize_gauge(samples, start=0, end=10, max_age=3)
gap = summarize_gauge(samples, start=5, end=7, max_age=3)

assert (report, report.mean, gap, gap.mean) == (
    GaugeWindow(7, 18),
    Fraction(18, 7),
    GaugeWindow(0, 0),
    None,
)
```

## Trade-offs and Limitations

The scan takes one pass over the supplied tuple and uses constant extra space
apart from integer sizes. The integral is in value-ticks. Read the coverage
alongside the mean: a short covered portion can have an exact mean that is
unrepresentative of the whole window.

This trusts the declared integer input types and does no interpolation.
Resolve duplicate timestamps before calling. For many queries on the same
history, consider indexing the usable intervals or precomputing prefix sums.

## Related Snippets

<!-- catalog:related:start -->
- [Integrate Regular Rate Samples Across Explicit Time Buckets](integrate-regular-rate-samples-across-explicit-time-buckets.md)
- [Measure Time in a State Within a Half-Open Window](measure-time-in-a-state-within-a-half-open-window.md)
<!-- catalog:related:end -->
