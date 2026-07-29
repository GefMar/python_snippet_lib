---
title: "Compute Exact Squared Pearson Correlation with Direction"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-exact-binary-roc-auc-with-tied-integer-scores.md
  - compute-exact-unweighted-cohens-kappa-from-a-confusion-matrix.md
  - compute-an-exact-integer-median-and-unscaled-median-absolute-deviation.md
---

# Compute Exact Squared Pearson Correlation with Direction

## Idea and Problem

Measure the squared linear correlation of paired bounded integers exactly while retaining the direction that squaring would otherwise discard.

Scaled centered sums avoid intermediate means: `n * sum(x**2) - sum(x)**2`
and its counterparts are integer multiples of the usual variance and
cross-deviation sums. Their common scale cancels in the correlation ratio, so
the squared magnitude is an exact `Fraction` and the cross term supplies its
negative, zero, or positive direction.

## When to Use

Use this calculation for a bounded, completely observed set of paired integer
measurements when exact reproducibility matters and squared Pearson correlation
is the chosen descriptive measure. It avoids binary floating-point rounding
without changing the usual linear-correlation definition.

Both coordinates must vary for Pearson correlation to be defined. Use a
statistical package when inputs are floating point, observations have weights
or missing values, or the analysis requires uncertainty estimates, hypothesis
tests, regression diagnostics, or robust treatment of outliers.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_OBSERVATIONS = 65_536


@dataclass(frozen=True, slots=True)
class SquaredPearsonCorrelation:
    count: int
    r_squared: Fraction | None
    direction: int | None


def _validate_signed_64_values(
    values: object,
    *,
    name: str,
) -> tuple[int, ...]:
    if type(values) is not tuple:
        raise TypeError(f"{name} must be an exact tuple")
    if not 2 <= len(values) <= _MAX_OBSERVATIONS:
        raise ValueError(f"{name} count is outside the supported range")
    for index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"{name}[{index}] must be an exact integer")
        if not _MIN_INT64 <= value <= _MAX_INT64:
            raise ValueError(f"{name}[{index}] is outside the signed 64-bit range")
    return values


def exact_squared_pearson_correlation(
    x_values: tuple[int, ...],
    y_values: tuple[int, ...],
) -> SquaredPearsonCorrelation:
    """Return exact squared Pearson correlation and covariance direction."""
    x = _validate_signed_64_values(x_values, name="x_values")
    y = _validate_signed_64_values(y_values, name="y_values")
    if len(x) != len(y):
        raise ValueError("x_values and y_values must have equal lengths")

    sum_x = 0
    sum_y = 0
    sum_x_squared = 0
    sum_y_squared = 0
    sum_products = 0
    for x_value, y_value in zip(x, y, strict=True):
        sum_x += x_value
        sum_y += y_value
        sum_x_squared += x_value * x_value
        sum_y_squared += y_value * y_value
        sum_products += x_value * y_value

    count = len(x)
    x_deviation = count * sum_x_squared - sum_x * sum_x
    y_deviation = count * sum_y_squared - sum_y * sum_y
    cross_deviation = count * sum_products - sum_x * sum_y

    if x_deviation == 0 or y_deviation == 0:
        return SquaredPearsonCorrelation(
            count=count,
            r_squared=None,
            direction=None,
        )

    direction = (cross_deviation > 0) - (cross_deviation < 0)
    return SquaredPearsonCorrelation(
        count=count,
        r_squared=Fraction(
            cross_deviation * cross_deviation,
            x_deviation * y_deviation,
        ),
        direction=direction,
    )
```

## Example

```python
positive = exact_squared_pearson_correlation((1, 2, 3), (1, 2, 4))
zero = exact_squared_pearson_correlation((-1, 0, 1), (1, -2, 1))
undefined = exact_squared_pearson_correlation((4, 4, 4), (1, 2, 3))

assert (positive, zero, undefined) == (
    SquaredPearsonCorrelation(3, Fraction(27, 28), 1),
    SquaredPearsonCorrelation(3, Fraction(0, 1), 0),
    SquaredPearsonCorrelation(3, None, None),
)
```

## Trade-offs and Limitations

Complete validation and accumulation take `O(n)` arithmetic work and use
`O(1)` auxiliary state. Python integers keep all sums and products exact, but
their bit lengths grow beyond the signed-64-bit input width, so multiplication,
division, and `Fraction` reduction are not constant-cost operations.

The returned fraction is the square of Pearson's product-moment correlation
and lies between zero and one. A direction of `-1`, `0`, or `1` records the
sign of the centered cross term; it is zero only for zero linear covariance.
If either coordinate is constant, the denominator is zero and both correlation
fields are returned as `None` rather than inventing a score.

Pearson correlation describes linear association and can be strongly affected
by outliers or a nonlinear relationship. Squaring removes the sign but does not
make the value causal, inferential, or a regression fit. This function does not
accept floats, missing values, weights, confidence intervals, hypothesis tests,
regression options, or causal assumptions.

## Related Snippets

<!-- catalog:related:start -->
- [Compute Exact Binary ROC AUC with Tied Integer Scores](compute-exact-binary-roc-auc-with-tied-integer-scores.md)
- [Compute Exact Unweighted Cohen's Kappa from a Confusion Matrix](compute-exact-unweighted-cohens-kappa-from-a-confusion-matrix.md)
- [Compute an Exact Integer Median and Unscaled Median Absolute Deviation](compute-an-exact-integer-median-and-unscaled-median-absolute-deviation.md)
<!-- catalog:related:end -->
