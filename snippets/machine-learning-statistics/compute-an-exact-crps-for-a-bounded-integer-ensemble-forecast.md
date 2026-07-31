---
title: "Compute an Exact CRPS for a Bounded Integer Ensemble Forecast"
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
  - compute-an-exact-binary-brier-score-from-integer-probability-ticks.md
  - compute-an-exact-normalized-one-dimensional-wasserstein-distance.md
  - select-a-forecast-vector-only-when-it-beats-a-frozen-baseline.md
---

# Compute an Exact CRPS for a Bounded Integer Ensemble Forecast

## Idea and Problem

Score one equally weighted integer ensemble forecast against an observed value with exact arithmetic.

The continuous ranked probability score for an empirical ensemble balances
two terms. Mean absolute error measures how far members are from the
observation, while a pairwise-distance term accounts for ensemble spread:

`CRPS = sum(abs(x_i - y)) / n - sum(i < j, abs(x_i - x_j)) / n**2`.

After sorting the members, every earlier value is no greater than the current
one. A running prefix sum therefore computes all pairwise distances involving
that member without constructing the quadratic set of pairs.

## When to Use

Use this algorithm for bounded reference calculations, regression tests, or
small forecast evaluations where ensemble members and observations are integer
ticks and a reproducible rational result is valuable. Duplicate members retain
their multiplicity and every member receives exactly the same probability
weight.

Use a maintained forecast-verification package for weighted ensembles,
floating-point and missing-value policies, continuous parametric
distributions, multivariate scores, large tabular workflows, calibration
analysis, inference, or model comparison. Choose those statistical policies
before treating any score as evidence about forecast quality.

## Implementation

```python
from fractions import Fraction

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_ENSEMBLE_SIZE = 4_096


def exact_ensemble_crps(
    ensemble: tuple[int, ...],
    observation: int,
) -> Fraction:
    """Return empirical equal-weight CRPS as a reduced Fraction."""
    if type(ensemble) is not tuple:
        raise TypeError("ensemble must be an exact tuple")
    if not 1 <= len(ensemble) <= _MAX_ENSEMBLE_SIZE:
        raise ValueError("ensemble size is outside the supported range")
    for index, member in enumerate(ensemble):
        if type(member) is not int:
            raise TypeError(f"ensemble[{index}] must be an exact integer")
        if not _MIN_INT64 <= member <= _MAX_INT64:
            raise ValueError(f"ensemble[{index}] is outside the signed 64-bit range")

    if type(observation) is not int:
        raise TypeError("observation must be an exact integer")
    if not _MIN_INT64 <= observation <= _MAX_INT64:
        raise ValueError("observation is outside the signed 64-bit range")

    ordered = sorted(ensemble)
    absolute_error_sum = sum(abs(member - observation) for member in ordered)

    pairwise_distance_sum = 0
    prefix_sum = 0
    for index, member in enumerate(ordered):
        pairwise_distance_sum += index * member - prefix_sum
        prefix_sum += member

    size = len(ordered)
    score = Fraction(absolute_error_sum, size) - Fraction(
        pairwise_distance_sum,
        size * size,
    )
    if score < 0:
        raise AssertionError("CRPS must be non-negative")
    return score
```

## Example

```python
def quadratic_crps_oracle(
    ensemble: tuple[int, ...],
    observation: int,
) -> Fraction:
    size = len(ensemble)
    first_term = Fraction(
        sum(abs(member - observation) for member in ensemble),
        size,
    )
    double_sum = sum(abs(first - second) for first in ensemble for second in ensemble)
    return first_term - Fraction(double_sum, 2 * size * size)


fixtures = (
    ((0, 2), 1, Fraction(1, 2)),
    ((0, 0, 2), 1, Fraction(5, 9)),
    ((7,), 2, Fraction(5)),
    ((5, 5, 5), 5, Fraction(0)),
    ((-(1 << 63), (1 << 63) - 1), 0, Fraction((1 << 64) - 1, 4)),
)

for members, observed, expected in fixtures:
    assert exact_ensemble_crps(members, observed) == expected
    assert exact_ensemble_crps(members, observed) == quadratic_crps_oracle(
        members,
        observed,
    )

original = (0, 0, 2)
assert exact_ensemble_crps(tuple(reversed(original)), 1) == Fraction(5, 9)
assert exact_ensemble_crps(tuple(value + 10 for value in original), 11) == Fraction(
    5,
    9,
)

invalid_calls = (
    ([], 0, TypeError),
    ((), 0, ValueError),
    ((0,) * (_MAX_ENSEMBLE_SIZE + 1), 0, ValueError),
    ((True,), 0, TypeError),
    ((_MIN_INT64 - 1,), 0, ValueError),
    ((0,), True, TypeError),
    ((0,), _MAX_INT64 + 1, ValueError),
)
rejected = 0
for members, observed, expected_error in invalid_calls:
    try:
        exact_ensemble_crps(members, observed)
    except expected_error:
        rejected += 1

assert rejected == len(invalid_calls)
```

## Trade-offs and Limitations

For `n` members, sorting takes `O(n log n)` time and `O(n)` memory; the two
summations are linear. Exact Python integers and `Fraction` avoid rounding, but
their arithmetic cost grows with operand magnitude. Input size is capped at
4,096 and every accepted value is in the signed-64 range.

The pairwise correction uses `n**2`, the empirical-distribution definition.
It is not the finite-ensemble fair-CRPS adjustment that uses `n * (n - 1)`.
Repeated values remain separate ensemble members, and member order is
irrelevant. A singleton score is simply its absolute error.

This is a univariate descriptive score. It does not accept member weights,
floats, missing values, quantile forecasts, continuous distributions, or
multivariate outcomes. It provides no threshold, p-value, confidence interval,
calibration diagnosis, model ranking policy, or claim that a lower score on
one observation generalizes.

## Related Snippets

<!-- catalog:related:start -->
- [Compute an Exact Binary Brier Score from Integer Probability Ticks](compute-an-exact-binary-brier-score-from-integer-probability-ticks.md)
- [Compute an Exact Normalized One-Dimensional Wasserstein Distance](compute-an-exact-normalized-one-dimensional-wasserstein-distance.md)
- [Select a Forecast Vector Only When It Beats a Frozen Baseline](select-a-forecast-vector-only-when-it-beats-a-frozen-baseline.md)
<!-- catalog:related:end -->
