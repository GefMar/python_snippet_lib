---
title: "Compute an Exact Two-Group Log-Rank Statistic from Bounded Right-Censored Data"
snippet_type: algorithm
use_cases:
  - data-transformation
  - testing
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - build-an-exact-kaplan-meier-curve-for-bounded-right-censored-data.md
  - build-an-exact-nelson-aalen-cumulative-hazard-table-for-bounded-right-censored-data.md
  - compute-an-exact-two-sample-kolmogorov-smirnov-distance-and-witness-for-bounded-integer-samples.md
---

# Compute an Exact Two-Group Log-Rank Statistic from Bounded Right-Censored Data

## Idea and Problem

Compare two bounded right-censored groups by accumulating the log-rank score and variance with exact rational arithmetic.

At each observed event time, the null calculation treats the number of group-1
events as a hypergeometric draw from the complete risk set. If `n1` and `n0`
subjects are at risk and `d` events occur, the expected group-1 count is
`d*n1/n`. Its tied-event variance contribution is
`n1*n0*d*(n-d)/(n**2*(n-1))` when the denominator is meaningful.

Summing observed minus expected counts gives the score. Dividing its square by
the summed variance gives the usual one-degree-of-freedom statistic, while
`Fraction` preserves the complete finite-table arithmetic.

## When to Use

Use this calculation for a small, auditable comparison of two independent
groups observed from a common origin when right censoring is non-informative.
Integer time ties are supported: subjects censored at an event time remain in
that time's risk set and leave only after its event contribution.

Use Kaplan-Meier or Nelson-Aalen when the goal is to estimate one group's curve
rather than compare two groups. Use a survival-analysis package when p-values,
confidence intervals, delayed entry, stratification, weights, clustering,
competing risks, or regression are required.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction
from itertools import product

_MAX_LOG_RANK_OBSERVATIONS = 4_096
_MAX_SURVIVAL_TIME = (1 << 63) - 1


@dataclass(frozen=True, slots=True)
class GroupedSurvivalObservation:
    time: int
    event: bool
    group: int


@dataclass(frozen=True, slots=True)
class LogRankResult:
    event_times: int
    observed_group_one: int
    expected_group_one: Fraction
    score: Fraction
    variance: Fraction
    statistic: Fraction | None


def exact_two_group_log_rank(
    observations: tuple[GroupedSurvivalObservation, ...],
) -> LogRankResult:
    """Return exact score evidence and the variance-standardized statistic."""
    if type(observations) is not tuple:
        raise TypeError("observations must be an exact tuple")
    if not 2 <= len(observations) <= _MAX_LOG_RANK_OBSERVATIONS:
        raise ValueError("observation count is outside 2..4,096")

    group_sizes = [0, 0]
    for index, observation in enumerate(observations):
        if type(observation) is not GroupedSurvivalObservation:
            raise TypeError(
                f"observations[{index}] must be an exact GroupedSurvivalObservation"
            )
        if type(observation.time) is not int:
            raise TypeError(f"observations[{index}].time must be an exact integer")
        if not 0 <= observation.time <= _MAX_SURVIVAL_TIME:
            raise ValueError(f"observations[{index}].time is outside signed 64-bit range")
        if type(observation.event) is not bool:
            raise TypeError(f"observations[{index}].event must be an exact boolean")
        if type(observation.group) is not int:
            raise TypeError(f"observations[{index}].group must be an exact integer")
        if observation.group not in (0, 1):
            raise ValueError(f"observations[{index}].group must be zero or one")
        group_sizes[observation.group] += 1
    if 0 in group_sizes:
        raise ValueError("both groups must contain at least one observation")

    ordered = sorted(observations, key=lambda observation: observation.time)
    at_risk = group_sizes.copy()
    observed_group_one = 0
    expected_group_one = Fraction(0, 1)
    variance = Fraction(0, 1)
    event_times = 0
    position = 0

    while position < len(ordered):
        time = ordered[position].time
        stop = position
        removed = [0, 0]
        events = [0, 0]
        while stop < len(ordered) and ordered[stop].time == time:
            observation = ordered[stop]
            removed[observation.group] += 1
            if observation.event:
                events[observation.group] += 1
            stop += 1

        total_events = events[0] + events[1]
        if total_events:
            total_at_risk = at_risk[0] + at_risk[1]
            expected = Fraction(total_events * at_risk[1], total_at_risk)
            expected_group_one += expected
            observed_group_one += events[1]
            event_times += 1
            if total_at_risk > 1:
                variance += Fraction(
                    at_risk[0]
                    * at_risk[1]
                    * total_events
                    * (total_at_risk - total_events),
                    total_at_risk * total_at_risk * (total_at_risk - 1),
                )

        at_risk[0] -= removed[0]
        at_risk[1] -= removed[1]
        position = stop

    score = Fraction(observed_group_one, 1) - expected_group_one
    statistic = None if variance == 0 else score * score / variance
    return LogRankResult(
        event_times=event_times,
        observed_group_one=observed_group_one,
        expected_group_one=expected_group_one,
        score=score,
        variance=variance,
        statistic=statistic,
    )
```

## Example

```python
def rescan_log_rank(
    observations: tuple[GroupedSurvivalObservation, ...],
) -> LogRankResult:
    event_times = sorted({item.time for item in observations if item.event})
    observed = 0
    expected = Fraction(0, 1)
    variance = Fraction(0, 1)
    for time in event_times:
        risk = [
            sum(item.group == group and item.time >= time for item in observations)
            for group in (0, 1)
        ]
        events = [
            sum(
                item.group == group and item.time == time and item.event
                for item in observations
            )
            for group in (0, 1)
        ]
        total_risk = sum(risk)
        total_events = sum(events)
        observed += events[1]
        expected += Fraction(total_events * risk[1], total_risk)
        if total_risk > 1:
            variance += Fraction(
                risk[0] * risk[1] * total_events * (total_risk - total_events),
                total_risk * total_risk * (total_risk - 1),
            )
    score = Fraction(observed, 1) - expected
    return LogRankResult(
        event_times=len(event_times),
        observed_group_one=observed,
        expected_group_one=expected,
        score=score,
        variance=variance,
        statistic=None if variance == 0 else score * score / variance,
    )


fixture = (
    GroupedSurvivalObservation(1, True, 0),
    GroupedSurvivalObservation(1, False, 1),
    GroupedSurvivalObservation(2, False, 0),
    GroupedSurvivalObservation(2, True, 1),
    GroupedSurvivalObservation(3, True, 1),
    GroupedSurvivalObservation(4, True, 0),
)
fixture_result = exact_two_group_log_rank(fixture)
swapped = exact_two_group_log_rank(
    tuple(
        GroupedSurvivalObservation(item.time, item.event, 1 - item.group)
        for item in fixture
    )
)
all_censored = exact_two_group_log_rank(
    (
        GroupedSurvivalObservation(1, False, 0),
        GroupedSurvivalObservation(2, False, 1),
    )
)

observation_kinds = tuple(
    GroupedSurvivalObservation(time, event, group)
    for time in range(3)
    for event in (False, True)
    for group in (0, 1)
)
exhaustive_samples = 0
for length in range(2, 5):
    for observations in product(observation_kinds, repeat=length):
        if {item.group for item in observations} != {0, 1}:
            continue
        actual = exact_two_group_log_rank(observations)
        assert actual == rescan_log_rank(observations)
        assert exact_two_group_log_rank(tuple(reversed(observations))) == actual
        exhaustive_samples += 1

maximum_observations = tuple(
    GroupedSurvivalObservation(index, index % 3 != 0, index % 2)
    for index in range(_MAX_LOG_RANK_OBSERVATIONS)
)
maximum_result = exact_two_group_log_rank(maximum_observations)

assert fixture_result == LogRankResult(
    event_times=4,
    observed_group_one=2,
    expected_group_one=Fraction(3, 2),
    score=Fraction(1, 2),
    variance=Fraction(3, 4),
    statistic=Fraction(1, 3),
)
assert (swapped.score, swapped.variance, swapped.statistic) == (
    -fixture_result.score,
    fixture_result.variance,
    fixture_result.statistic,
)
assert all_censored == LogRankResult(0, 0, Fraction(0), Fraction(0), Fraction(0), None)
assert (
    exhaustive_samples,
    maximum_result.event_times,
    maximum_result.variance > 0,
) == (19_512, 2_730, True)
```

## Trade-offs and Limitations

Sorting costs `O(N log N)` comparisons; the ordered copy uses `O(N)` state.
Fraction addition and reduction are exact but their arithmetic cost grows with
intermediate numerator and denominator bit length. The independent rescan in
the Example is deliberately quadratic and is not part of the implementation.

“Exact” describes the arithmetic score, variance, and ratio only. The function
does not compute an exact finite-sample null distribution or a p-value. When
all variance contributions vanish, the standardized statistic is unavailable
rather than zero; a numeric zero would incorrectly suggest informative
evidence of equality.

The comparison assumes independent observations, a common time origin, and
non-informative right censoring. It does not handle delayed entry, left or
interval censoring, weights, strata, competing risks, clustered observations,
confidence intervals, effect-size estimation, causal conclusions, or tests of
the proportional-hazards assumption.

## Related Snippets

<!-- catalog:related:start -->
- [Build an Exact Kaplan-Meier Curve for Bounded Right-Censored Data](build-an-exact-kaplan-meier-curve-for-bounded-right-censored-data.md)
- [Build an Exact Nelson-Aalen Cumulative Hazard Table for Bounded Right-Censored Data](build-an-exact-nelson-aalen-cumulative-hazard-table-for-bounded-right-censored-data.md)
- [Compute an Exact Two-Sample Kolmogorov-Smirnov Distance and Witness for Bounded Integer Samples](compute-an-exact-two-sample-kolmogorov-smirnov-distance-and-witness-for-bounded-integer-samples.md)
<!-- catalog:related:end -->
