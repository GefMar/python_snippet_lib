---
title: "Build an Exact Nelson-Aalen Cumulative Hazard Table for Bounded Right-Censored Data"
snippet_type: algorithm
use_cases:
  - data-transformation
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - build-an-exact-kaplan-meier-curve-for-bounded-right-censored-data.md
  - compute-an-exact-poisson-binomial-pmf-from-bounded-rational-probabilities.md
  - fit-an-exact-unweighted-isotonic-regression-with-pool-adjacent-violators.md
---

# Build an Exact Nelson-Aalen Cumulative Hazard Table for Bounded Right-Censored Data

## Idea and Problem

Build an auditable Nelson-Aalen cumulative-hazard table from bounded integer event-or-censoring times using exact fractions.

At each distinct time, the hazard increment is the number of observed events
divided by the number of observations still at risk immediately before that
time. All events and censorings tied at the time remain in that risk set; the
whole group leaves only after the increment is recorded.

Summing `Fraction` increments avoids floating-point drift and makes tied-time
semantics visible. Censor-only times are retained with a zero increment so the
returned table accounts for every risk-set reduction.

## When to Use

Use this estimator when independent observations share a common origin and
each record contains either an observed event time or a last event-free
right-censoring time. It is useful for reproducible small-sample calculations,
teaching, fixtures, and audits that need exact life-table arithmetic.

Right censoring must be non-informative for the intended analysis. Use a
survival-analysis library for delayed entry, weights, strata, competing risks,
variance estimates, confidence bands, or model fitting.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction

_MAX_OBSERVATIONS = 4_096
_MAX_INT64 = (1 << 63) - 1


@dataclass(frozen=True, slots=True)
class HazardObservation:
    time: int
    event: bool


@dataclass(frozen=True, slots=True)
class NelsonAalenPoint:
    time: int
    at_risk: int
    events: int
    censored: int
    increment: Fraction
    cumulative_hazard: Fraction


def build_nelson_aalen_table(
    observations: tuple[HazardObservation, ...],
) -> tuple[NelsonAalenPoint, ...]:
    if type(observations) is not tuple:
        raise TypeError("observations must be an exact tuple")
    if not 1 <= len(observations) <= _MAX_OBSERVATIONS:
        raise ValueError("observation count is outside the supported range")

    for index, observation in enumerate(observations):
        if type(observation) is not HazardObservation:
            raise TypeError(
                f"observations[{index}] must be an exact HazardObservation"
            )
        if type(observation.time) is not int:
            raise TypeError(
                f"observations[{index}].time must be an exact integer"
            )
        if not 0 <= observation.time <= _MAX_INT64:
            raise ValueError(
                f"observations[{index}].time is outside non-negative signed-64"
            )
        if type(observation.event) is not bool:
            raise TypeError(
                f"observations[{index}].event must be an exact boolean"
            )

    ordered = sorted(observations, key=lambda observation: observation.time)
    at_risk = len(ordered)
    cumulative = Fraction(0, 1)
    points: list[NelsonAalenPoint] = []
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
        increment = Fraction(events, at_risk)
        cumulative += increment
        points.append(
            NelsonAalenPoint(
                time=time,
                at_risk=at_risk,
                events=events,
                censored=censored,
                increment=increment,
                cumulative_hazard=cumulative,
            )
        )

        at_risk -= group_size
        position = group_stop

    return tuple(points)
```

## Example

```python
observations = (
    HazardObservation(1, True),
    HazardObservation(2, True),
    HazardObservation(2, False),
    HazardObservation(3, False),
    HazardObservation(4, True),
)

table = build_nelson_aalen_table(observations)
permuted = build_nelson_aalen_table(tuple(reversed(observations)))
all_censored = build_nelson_aalen_table(
    (HazardObservation(0, False), HazardObservation(2, False))
)

assert table == (
    NelsonAalenPoint(1, 5, 1, 0, Fraction(1, 5), Fraction(1, 5)),
    NelsonAalenPoint(2, 4, 1, 1, Fraction(1, 4), Fraction(9, 20)),
    NelsonAalenPoint(3, 2, 0, 1, Fraction(0, 1), Fraction(9, 20)),
    NelsonAalenPoint(4, 1, 1, 0, Fraction(1, 1), Fraction(29, 20)),
)
assert permuted == table
assert all_censored == (
    NelsonAalenPoint(0, 2, 0, 1, Fraction(0, 1), Fraction(0, 1)),
    NelsonAalenPoint(2, 1, 0, 1, Fraction(0, 1), Fraction(0, 1)),
)
```

## Trade-offs and Limitations

Sorting `n` observations costs `O(n log n)` comparisons; the ordered copy and
at most `n` table rows use `O(n)` memory. Fraction addition and reduction have
costs that grow with integer bit length rather than remaining constant. Input
order among tied observations cannot affect the grouped result.

The Nelson-Aalen estimator adds event-to-risk ratios. It is not the
Kaplan-Meier product-limit calculation. Although `exp(-cumulative_hazard)` is
often used to derive a survival estimate, it must not be claimed equal to the
Kaplan-Meier sample curve; their formulas and finite-sample values differ.

The result describes an empirical estimate under common-origin, independent-
observation, and non-informative right-censoring assumptions. The function does
not handle delayed entry, left or interval censoring, weights, strata,
competing risks, variance or confidence intervals, smoothing, regression, or
population-level causal conclusions.

## Related Snippets

<!-- catalog:related:start -->
- [Build an Exact Kaplan-Meier Curve for Bounded Right-Censored Data](build-an-exact-kaplan-meier-curve-for-bounded-right-censored-data.md)
- [Compute an Exact Poisson-Binomial PMF from Bounded Rational Probabilities](compute-an-exact-poisson-binomial-pmf-from-bounded-rational-probabilities.md)
- [Fit an Exact Unweighted Isotonic Regression with Pool-Adjacent Violators](fit-an-exact-unweighted-isotonic-regression-with-pool-adjacent-violators.md)
<!-- catalog:related:end -->
