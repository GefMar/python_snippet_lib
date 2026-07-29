---
title: "Build an Exact Kaplan-Meier Curve for Bounded Right-Censored Data"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-an-exact-integer-median-and-unscaled-median-absolute-deviation.md
  - fit-an-exact-unweighted-isotonic-regression-with-pool-adjacent-violators.md
  - ../data-processing/measure-time-in-a-state-within-a-half-open-window.md
---

# Build an Exact Kaplan-Meier Curve for Bounded Right-Censored Data

## Idea and Problem

Build a Kaplan-Meier product-limit curve from bounded integer event-or-censoring times while preserving every sample survival level as an exact fraction.

At each distinct time, every observation still under study forms the risk set.
Observed events multiply survival by the fraction that remains event-free;
right-censored observations leave the risk set only after that same-time update.
Grouping ties makes the result independent of input order and records enough
life-table evidence to audit each step.

## When to Use

Use this estimator when each independent observation starts at a common origin
and supplies either an observed event time or a last event-free censoring time.
Right censoring must be non-informative for the intended analysis: after
accounting for the modeled information, censoring must not reveal the otherwise
unobserved event time.

This compact implementation is useful when exact reproducible sample-curve
arithmetic matters more than inferential features. Use a survival-analysis
library for delayed entry, clustered observations, competing risks, weights,
strata, confidence bands, variance estimates, or model-based comparisons.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction

_MAX_INT64 = (1 << 63) - 1
_MAX_OBSERVATIONS = 4_096


@dataclass(frozen=True, slots=True)
class SurvivalObservation:
    time: int
    event: bool


@dataclass(frozen=True, slots=True)
class KaplanMeierPoint:
    time: int
    at_risk: int
    events: int
    censored: int
    survival: Fraction


def build_kaplan_meier_curve(
    observations: tuple[SurvivalObservation, ...],
) -> tuple[KaplanMeierPoint, ...]:
    """Return exact life-table points for bounded right-censored data."""
    if type(observations) is not tuple:
        raise TypeError("observations must be an exact tuple")
    if not 1 <= len(observations) <= _MAX_OBSERVATIONS:
        raise ValueError("observation count is outside the supported range")

    for index, observation in enumerate(observations):
        if type(observation) is not SurvivalObservation:
            raise TypeError(f"observations[{index}] must be an exact SurvivalObservation")
        if type(observation.time) is not int:
            raise TypeError(f"observations[{index}].time must be an exact integer")
        if not 0 <= observation.time <= _MAX_INT64:
            raise ValueError(
                f"observations[{index}].time is outside the non-negative signed 64-bit range"
            )
        if type(observation.event) is not bool:
            raise TypeError(f"observations[{index}].event must be an exact boolean")

    ordered = sorted(observations, key=lambda observation: observation.time)
    at_risk = len(ordered)
    survival = Fraction(1, 1)
    points: list[KaplanMeierPoint] = []
    position = 0

    while position < len(ordered):
        time = ordered[position].time
        group_stop = position
        events = 0
        while group_stop < len(ordered) and ordered[group_stop].time == time:
            if ordered[group_stop].event:
                events += 1
            group_stop += 1

        group_size = group_stop - position
        censored = group_size - events
        if events:
            survival *= Fraction(at_risk - events, at_risk)
        points.append(
            KaplanMeierPoint(
                time=time,
                at_risk=at_risk,
                events=events,
                censored=censored,
                survival=survival,
            )
        )

        at_risk -= group_size
        position = group_stop

    return tuple(points)
```

## Example

```python
observations = (
    SurvivalObservation(1, True),
    SurvivalObservation(2, True),
    SurvivalObservation(2, False),
    SurvivalObservation(3, False),
    SurvivalObservation(4, True),
)

curve = build_kaplan_meier_curve(observations)
permuted = build_kaplan_meier_curve(tuple(reversed(observations)))
all_censored = build_kaplan_meier_curve(
    (SurvivalObservation(0, False), SurvivalObservation(2, False))
)

assert (curve, permuted == curve, all_censored) == (
    (
        KaplanMeierPoint(1, 5, 1, 0, Fraction(4, 5)),
        KaplanMeierPoint(2, 4, 1, 1, Fraction(3, 5)),
        KaplanMeierPoint(3, 2, 0, 1, Fraction(3, 5)),
        KaplanMeierPoint(4, 1, 1, 0, Fraction(0, 1)),
    ),
    True,
    (
        KaplanMeierPoint(0, 2, 0, 1, Fraction(1, 1)),
        KaplanMeierPoint(2, 1, 0, 1, Fraction(1, 1)),
    ),
)
```

## Trade-offs and Limitations

Sorting `n` observations takes `O(n log n)` comparisons; the ordered copy and
at most `n` returned points use `O(n)` memory. Python integer, gcd, and
`Fraction` operations are not constant-cost, and their work grows with the bit
length of intermediate numerators and denominators.

The exact fractions describe the empirical product-limit calculation, not the
unknown population survival function or its uncertainty. Censor-only times are
included with unchanged survival for an auditable life table, although the
curve itself falls only at observed event times. When events and censorings
share a time, all remain in the risk set for that time's event update.

The function assumes a common observation origin and non-informative right
censoring. It does not support delayed entry, left or interval censoring,
competing risks, weights, strata, confidence intervals, Greenwood variance,
hazards, or median-survival inference. For complete uncensored observations, a
simple descriptive statistic may be a better fit.

## Related Snippets

<!-- catalog:related:start -->
- [Compute an Exact Integer Median and Unscaled Median Absolute Deviation](compute-an-exact-integer-median-and-unscaled-median-absolute-deviation.md)
- [Fit an Exact Unweighted Isotonic Regression with Pool-Adjacent Violators](fit-an-exact-unweighted-isotonic-regression-with-pool-adjacent-violators.md)
- [Measure Time in a State Within a Half-Open Window](../data-processing/measure-time-in-a-state-within-a-half-open-window.md)
<!-- catalog:related:end -->
