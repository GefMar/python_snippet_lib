---
title: "Fit an Exact Ordinary Least-Squares Line to Bounded Integer Points"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - estimate-an-exact-theil-sen-slope-from-bounded-integer-points.md
  - compute-exact-squared-pearson-correlation-with-direction.md
  - fit-an-exact-unweighted-isotonic-regression-with-pool-adjacent-violators.md
---

# Fit an Exact Ordinary Least-Squares Line to Bounded Integer Points

## Idea and Problem

Fit the unique vertical-residual least-squares line through bounded integer points when the x-coordinates vary, while identifying constant-x input as undefined.

For `n` observations, five integer statistics are sufficient: `n`, `sum(x)`,
`sum(y)`, `sum(x * x)`, and `sum(x * y)`. If
`D = n * sum(x * x) - sum(x) ** 2` is non-zero, the normal equations give
`slope = (n * sum(x * y) - sum(x) * sum(y)) / D`; the intercept then makes
the mean residual zero. `Fraction` keeps both coefficients exact.

## When to Use

Use this calculation for a completely observed collection of bounded integer
points when an exact, reproducible descriptive straight-line fit is useful.
Input order is irrelevant, while duplicate points and repeated `x` values keep
their multiplicity, so repeated observations intentionally influence the fit.
The residuals are vertical: `x` is treated as the predictor rather than as a
second interchangeable coordinate.

No probabilistic assumptions are needed to compute the observed-sample
sum-of-squared-error minimizer. Use a statistical package when the work needs
inference, prediction assessment, weights, missing or floating-point values,
diagnostics, robust fitting, or a model with more than one predictor.

## Implementation

```python
from fractions import Fraction

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_OLS_POINTS = 100_000


def fit_exact_ordinary_least_squares_line(
    points: tuple[tuple[int, int], ...],
) -> tuple[Fraction, Fraction] | None:
    """Return exact slope and intercept, or None for constant x."""
    if type(points) is not tuple:
        raise TypeError("points must be an exact tuple")
    if not 2 <= len(points) <= _MAX_OLS_POINTS:
        raise ValueError("point count is outside the supported range")

    for index, point in enumerate(points):
        if type(point) is not tuple:
            raise TypeError(f"points[{index}] must be an exact tuple")
        if len(point) != 2:
            raise ValueError(f"points[{index}] must contain exactly two items")
        x_value, y_value = point
        if type(x_value) is not int:
            raise TypeError(f"points[{index}].x must be an exact integer")
        if type(y_value) is not int:
            raise TypeError(f"points[{index}].y must be an exact integer")
        if not _MIN_INT64 <= x_value <= _MAX_INT64:
            raise ValueError(f"points[{index}].x is outside the signed 64-bit range")
        if not _MIN_INT64 <= y_value <= _MAX_INT64:
            raise ValueError(f"points[{index}].y is outside the signed 64-bit range")

    sum_x = 0
    sum_y = 0
    sum_x_squared = 0
    sum_xy = 0
    for x_value, y_value in points:
        sum_x += x_value
        sum_y += y_value
        sum_x_squared += x_value * x_value
        sum_xy += x_value * y_value

    count = len(points)
    denominator = count * sum_x_squared - sum_x * sum_x
    if denominator == 0:
        return None

    slope_numerator = count * sum_xy - sum_x * sum_y
    slope = Fraction(slope_numerator, denominator)
    intercept = Fraction(sum_y, count) - slope * Fraction(sum_x, count)
    return slope, intercept
```

## Example

```python
def centered_fraction_oracle(
    points: tuple[tuple[int, int], ...],
) -> tuple[Fraction, Fraction] | None:
    count = Fraction(len(points))
    mean_x = sum((Fraction(x_value) for x_value, _ in points), Fraction()) / count
    mean_y = sum((Fraction(y_value) for _, y_value in points), Fraction()) / count

    covariance = Fraction()
    x_variance = Fraction()
    for x_value, y_value in points:
        centered_x = Fraction(x_value) - mean_x
        centered_y = Fraction(y_value) - mean_y
        covariance += centered_x * centered_y
        x_variance += centered_x * centered_x

    if x_variance == 0:
        return None
    slope = covariance / x_variance
    return slope, mean_y - slope * mean_x


def exhaustive_short_point_sequences():
    from itertools import product

    coordinate_points = tuple(product((-1, 0, 1), repeat=2))
    for length in range(2, 5):
        yield from product(coordinate_points, repeat=length)


exhaustive_matches = True
exhaustive_count = 0
for candidate in exhaustive_short_point_sequences():
    if fit_exact_ordinary_least_squares_line(candidate) != centered_fraction_oracle(candidate):
        exhaustive_matches = False
    exhaustive_count += 1

fractional = ((0, 0), (1, 1), (2, 1))
zero_slope = ((0, 2), (1, 1), (2, 2))
duplicates = ((0, 0), (1, 2), (2, 1), (2, 1))
repeated_x = ((0, 0), (1, 2), (1, 4), (2, 3))
constant_x = ((5, -1), (5, 0), (5, 9))
extrema = ((_MIN_INT64, _MAX_INT64), (0, 0), (_MAX_INT64, _MIN_INT64))

order_independent = fit_exact_ordinary_least_squares_line(
    tuple(reversed(repeated_x))
) == fit_exact_ordinary_least_squares_line(repeated_x)
extrema_matches = fit_exact_ordinary_least_squares_line(extrema) == centered_fraction_oracle(
    extrema
)

large_points = tuple((x_value, 3 * x_value + 7) for x_value in range(-50_000, 50_000))
large_fit = fit_exact_ordinary_least_squares_line(large_points)

invalid_inputs: tuple[tuple[object, type[Exception]], ...] = (
    ([(0, 0), (1, 1)], TypeError),
    (((0, 0),), ValueError),
    (((0, 0),) * (_MAX_OLS_POINTS + 1), ValueError),
    (((0, 0), [1, 1]), TypeError),
    (((0, 0), (1,)), ValueError),
    (((0, 0), (1, True)), TypeError),
    (((0, 0), (1 << 63, 1)), ValueError),
)
validation_rejections = []
for invalid_input, expected_exception in invalid_inputs:
    try:
        fit_exact_ordinary_least_squares_line(invalid_input)
    except expected_exception:
        validation_rejections.append(True)
    else:
        validation_rejections.append(False)

assert (
    exhaustive_matches,
    exhaustive_count,
    fit_exact_ordinary_least_squares_line(fractional),
    fit_exact_ordinary_least_squares_line(zero_slope),
    fit_exact_ordinary_least_squares_line(duplicates),
    fit_exact_ordinary_least_squares_line(repeated_x) == centered_fraction_oracle(repeated_x),
    fit_exact_ordinary_least_squares_line(constant_x),
    order_independent,
    extrema_matches,
    large_fit,
    all(validation_rejections),
) == (
    True,
    7_371,
    (Fraction(1, 2), Fraction(1, 6)),
    (Fraction(0), Fraction(5, 3)),
    (Fraction(4, 11), Fraction(6, 11)),
    True,
    None,
    True,
    True,
    (Fraction(3), Fraction(7)),
    True,
)
```

## Trade-offs and Limitations

Validation and accumulation each make one pass, for `O(n)` big-integer
operations and `O(1)` auxiliary state beyond the already materialized input.
Signed-64-bit products need roughly twice the input width, summation adds bits
with the point count, and the normal-equation products can grow further. Thus
the linear operation count is not a constant-time arithmetic claim. Constructing
the final `Fraction` values also pays greatest-common-divisor costs that depend
on the accumulated operands' bit lengths.

For nonconstant `x`, the two returned coefficients uniquely minimize the sum
of squared vertical residuals over the observed sample. `None` is returned only
when all `x` values are equal and the slope is undefined. Exact arithmetic does
not establish predictive validity, causal meaning, unbiasedness, uncertainty,
or any other inferential conclusion; those require an appropriate error model
and separate analysis.

This intentionally excludes weights, missing or floating-point values,
multivariate and robust fitting, regularization, diagnostics, fitted or residual
output, hypothesis tests, and confidence or prediction intervals. Outliers,
nonlinearity, dependence, and measurement error in `x` can still make the
descriptive line misleading.

## Related Snippets

<!-- catalog:related:start -->
- [Estimate an Exact Theil-Sen Slope from Bounded Integer Points](estimate-an-exact-theil-sen-slope-from-bounded-integer-points.md)
- [Compute Exact Squared Pearson Correlation with Direction](compute-exact-squared-pearson-correlation-with-direction.md)
- [Fit an Exact Unweighted Isotonic Regression with Pool-Adjacent Violators](fit-an-exact-unweighted-isotonic-regression-with-pool-adjacent-violators.md)
<!-- catalog:related:end -->
