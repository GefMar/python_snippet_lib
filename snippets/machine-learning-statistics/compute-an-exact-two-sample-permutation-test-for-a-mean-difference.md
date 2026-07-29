---
title: "Compute an Exact Two-Sample Permutation Test for a Mean Difference"
snippet_type: algorithm
use_cases:
  - data-transformation
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-an-exact-paired-two-sided-sign-test.md
  - compute-an-exact-two-sample-kolmogorov-smirnov-distance-and-witness-for-bounded-integer-samples.md
  - adjust-bounded-exact-p-values-with-holms-step-down-method.md
---

# Compute an Exact Two-Sample Permutation Test for a Mean Difference

## Idea and Problem

Enumerate every fixed-size reassignment of pooled observations to compute an exact two-sided permutation p-value.

The statistic is the absolute difference between two sample means. Because
every reassignment keeps the group sizes fixed, comparing
`abs(N * selected_sum - n_first * pooled_sum)` is equivalent and avoids
floating-point arithmetic inside the enumeration.

Pooled positions remain distinct even when their integer values are equal.
The inclusive tail counts the observed allocation, and the p-value is returned
as a `Fraction` without a Monte Carlo correction.

## When to Use

Use this exhaustive test for two small independent samples when labels are
exchangeable under the null hypothesis. That assumption commonly means the
observations would have the same joint distribution after any admitted label
permutation, not merely that two unrelated distributions have equal means.

The exact enumeration is useful for reproducible small-sample checks where a
Monte Carlo approximation is unnecessary and the absolute mean difference is
the preselected statistic.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction
from itertools import combinations
from math import comb

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_TOTAL_OBSERVATIONS = 20


@dataclass(frozen=True, slots=True)
class PermutationTestResult:
    observed_mean_difference: Fraction
    total_allocations: int
    extreme_allocations: int
    two_sided_p_value: Fraction


def _validate_sample(name: str, sample: object) -> tuple[int, ...]:
    if type(sample) is not tuple:
        raise TypeError(f"{name} must be an exact tuple")
    if not sample:
        raise ValueError(f"{name} must not be empty")
    return sample


def exact_mean_difference_permutation_test(
    first: tuple[int, ...],
    second: tuple[int, ...],
) -> PermutationTestResult:
    """Return an exhaustive two-sided permutation test under exchangeability."""
    first = _validate_sample("first", first)
    second = _validate_sample("second", second)
    total_observations = len(first) + len(second)
    if total_observations > _MAX_TOTAL_OBSERVATIONS:
        raise ValueError("combined sample size exceeds the supported limit")
    for name, sample in (("first", first), ("second", second)):
        for observation in sample:
            if type(observation) is not int:
                raise TypeError(f"{name} observations must be exact integers")
            if not _MIN_INT64 <= observation <= _MAX_INT64:
                raise ValueError(
                    f"{name} observation is outside the int64 range"
                )

    first_size = len(first)
    second_size = len(second)
    pooled = first + second
    pooled_sum = sum(pooled)
    first_sum = sum(first)

    observed_numerator = (
        total_observations * first_sum - first_size * pooled_sum
    )
    observed_difference = Fraction(
        observed_numerator,
        first_size * second_size,
    )
    observed_absolute_numerator = abs(observed_numerator)

    extreme_allocations = 0
    for selected_indices in combinations(range(total_observations), first_size):
        selected_sum = sum(pooled[index] for index in selected_indices)
        candidate_numerator = abs(
            total_observations * selected_sum - first_size * pooled_sum
        )
        if candidate_numerator >= observed_absolute_numerator:
            extreme_allocations += 1

    total_allocations = comb(total_observations, first_size)
    return PermutationTestResult(
        observed_mean_difference=observed_difference,
        total_allocations=total_allocations,
        extreme_allocations=extreme_allocations,
        two_sided_p_value=Fraction(extreme_allocations, total_allocations),
    )
```

## Example

```python
separated = exact_mean_difference_permutation_test((0, 0), (10, 10))
assert separated.observed_mean_difference == Fraction(-10)
assert separated.total_allocations == 6
assert separated.extreme_allocations == 2
assert separated.two_sided_p_value == Fraction(1, 3)

all_equal = exact_mean_difference_permutation_test((4, 4), (4, 4, 4))
assert all_equal.two_sided_p_value == 1

swapped = exact_mean_difference_permutation_test((10, 10), (0, 0))
assert swapped.observed_mean_difference == -separated.observed_mean_difference
assert swapped.two_sided_p_value == separated.two_sided_p_value

translated = exact_mean_difference_permutation_test((7, 7), (17, 17))
assert translated.two_sided_p_value == separated.two_sided_p_value
```

## Trade-offs and Limitations

With `N` pooled observations and first-group size `n`, direct enumeration uses
`O(n * C(N, n))` integer work and `O(N)` stored input plus combination state.
The largest supported allocation space is `C(20, 10) = 184,756`. This bound
does not make repeated high-volume use inexpensive.

Exactness is conditional on exchangeability and the chosen statistic. Equal
means alone do not make arbitrary labels exchangeable when distributions have
different shapes or variances. Observations must also be independent in the
way required by the study design; paired or clustered measurements need a
different permutation scheme.

The function provides no one-sided convention, studentization, stratification,
Monte Carlo sampling, confidence interval, multiple-testing adjustment, alpha
decision, covariate handling, or causal interpretation. Duplicate values are
not collapsed because allocations concern labelled observation positions.

## Related Snippets

<!-- catalog:related:start -->
- [Compute an Exact Paired Two-Sided Sign Test](compute-an-exact-paired-two-sided-sign-test.md)
- [Compute an Exact Two-Sample Kolmogorov-Smirnov Distance and Witness for Bounded Integer Samples](compute-an-exact-two-sample-kolmogorov-smirnov-distance-and-witness-for-bounded-integer-samples.md)
- [Adjust Bounded Exact P-Values with Holm's Step-Down Method](adjust-bounded-exact-p-values-with-holms-step-down-method.md)
<!-- catalog:related:end -->
