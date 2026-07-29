---
title: "Compute the Exact Gini Coefficient of Bounded Non-Negative Integers"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-an-exact-integer-median-and-unscaled-median-absolute-deviation.md
  - compute-exact-squared-pearson-correlation-with-direction.md
  - calculate-a-symmetrically-trimmed-mean.md
---

# Compute the Exact Gini Coefficient of Bounded Non-Negative Integers

## Idea and Problem

Measure inequality within bounded non-negative integer observations as an exact uncorrected population Gini coefficient.

The direct definition sums every pairwise absolute difference and divides by
twice the population size and total. Sorting yields the same numerator through
one position-weighted sum, reducing quadratic comparison work to a sort plus a
linear scan while `Fraction` preserves the exact result.

## When to Use

Use this calculation for one finite, equally weighted population of
non-negative integer observations when the uncorrected population definition
is the intended descriptive measure. Exact arithmetic is useful for stable
tests, small audits, and comparisons that must not depend on floating-point
rounding.

Choose a different estimator when observations carry weights or the analysis
requires a finite-sample correction. Use a statistical package for inference,
uncertainty estimates, streaming approximation, or domain-specific fairness
analysis.

## Implementation

```python
from fractions import Fraction

_MAX_INT64 = (1 << 63) - 1
_MAX_OBSERVATIONS = 65_536


def exact_gini_coefficient(values: tuple[int, ...]) -> Fraction:
    """Return the uncorrected population Gini coefficient exactly."""
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    if not 1 <= len(values) <= _MAX_OBSERVATIONS:
        raise ValueError("value count is outside the supported range")

    total = 0
    for index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"values[{index}] must be an exact integer")
        if not 0 <= value <= _MAX_INT64:
            raise ValueError(f"values[{index}] is outside the supported range")
        total += value
    if total == 0:
        raise ValueError("values must have a positive total")

    ordered = sorted(values)
    weighted_sum = sum(
        rank * value
        for rank, value in enumerate(ordered, start=1)
    )
    count = len(ordered)
    numerator = 2 * weighted_sum - (count + 1) * total
    return Fraction(numerator, count * total)
```

## Example

```python
def quadratic_gini(values: tuple[int, ...]) -> Fraction:
    pairwise_difference = sum(
        abs(left - right)
        for left in values
        for right in values
    )
    return Fraction(
        pairwise_difference,
        2 * len(values) * sum(values),
    )


def verify_small_populations() -> None:
    from itertools import product

    for length in range(1, 5):
        for sample in product(range(4), repeat=length):
            if sum(sample) == 0:
                continue
            result = exact_gini_coefficient(sample)
            assert result == quadratic_gini(sample)
            assert Fraction(0) <= result <= Fraction(length - 1, length)


verify_small_populations()

baseline = (0, 1, 3, 8)
assert (
    exact_gini_coefficient(baseline)
    == exact_gini_coefficient(tuple(reversed(baseline)))
    == exact_gini_coefficient(tuple(7 * value for value in baseline))
)
assert exact_gini_coefficient((7, 7, 7)) == Fraction(0)
assert exact_gini_coefficient((0, 0, 9, 0)) == Fraction(3, 4)
```

## Trade-offs and Limitations

For `n` observations, validation and accumulation take `O(n)` work, sorting
takes `O(n log n)` time, and the sorted copy uses `O(n)` memory. Although each
input fits a signed 64-bit non-negative range, totals and weighted products can
be wider. Python keeps them exact, but large-integer multiplication, division,
and fraction reduction are not constant-cost operations.

This is the uncorrected population coefficient
`sum(abs(x_i - x_j)) / (2 * n * sum(x))` over all ordered pairs. It lies from
zero through `(n - 1) / n`: equal positive observations produce zero, while
one positive value among otherwise zero observations reaches the upper bound.
The all-zero population is rejected because the definition's denominator is
zero.

The function rejects negative values, Booleans, arbitrary iterables, weights,
and more than 65,536 observations. It does not apply a finite-sample
normalization, approximate a stream, perform inference, or support fairness or
causal conclusions.

## Related Snippets

<!-- catalog:related:start -->
- [Compute an Exact Integer Median and Unscaled Median Absolute Deviation](compute-an-exact-integer-median-and-unscaled-median-absolute-deviation.md)
- [Compute Exact Squared Pearson Correlation with Direction](compute-exact-squared-pearson-correlation-with-direction.md)
- [Calculate a Symmetrically Trimmed Mean](calculate-a-symmetrically-trimmed-mean.md)
<!-- catalog:related:end -->
