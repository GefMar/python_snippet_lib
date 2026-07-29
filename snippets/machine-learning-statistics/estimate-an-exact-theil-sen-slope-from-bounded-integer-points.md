---
title: "Estimate an Exact Theil-Sen Slope from Bounded Integer Points"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-exact-squared-pearson-correlation-with-direction.md
  - compute-an-exact-integer-median-and-unscaled-median-absolute-deviation.md
  - calculate-a-symmetrically-trimmed-mean.md
---

# Estimate an Exact Theil-Sen Slope from Bounded Integer Points

## Idea and Problem

Estimate one robust line slope as the exact median of every pairwise slope between bounded integer points.

Strictly increasing `x` coordinates make every pairwise denominator positive
and non-zero. Constructing each slope as a `Fraction` preserves exact evidence,
and taking the median limits the influence of a minority of unusually steep or
shallow pairs without converting the result to binary floating point.

## When to Use

Use this estimator for a small, completely observed set of ordered integer
points when a robust descriptive slope is more useful than an ordinary
least-squares fit. It is a good fit when quadratic slope materialization is
acceptable and an exact rational result helps reproducible comparisons.

Use a statistical package when the analysis needs an intercept, confidence
intervals, hypothesis tests, repeated `x` coordinates, missing values, weights,
large data, or a complete regression model. Robustness to some outliers does
not make the result causal or inferential.

## Implementation

```python
from fractions import Fraction

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_POINT_COUNT = 256


def estimate_exact_theil_sen_slope(
    points: tuple[tuple[int, int], ...],
) -> Fraction:
    """Return the exact median of all pairwise slopes."""
    if type(points) is not tuple:
        raise TypeError("points must be an exact tuple")
    if not 2 <= len(points) <= _MAX_POINT_COUNT:
        raise ValueError("point count is outside the supported range")

    for index, point in enumerate(points):
        if type(point) is not tuple:
            raise TypeError(f"points[{index}] must be an exact tuple")
    for index, point in enumerate(points):
        if len(point) != 2:
            raise ValueError(f"points[{index}] must contain exactly two items")
    for index, (x_value, y_value) in enumerate(points):
        if type(x_value) is not int:
            raise TypeError(f"points[{index}].x must be an exact non-boolean integer")
        if type(y_value) is not int:
            raise TypeError(f"points[{index}].y must be an exact non-boolean integer")

    previous_x: int | None = None
    for index, (x_value, y_value) in enumerate(points):
        if not _MIN_INT64 <= x_value <= _MAX_INT64:
            raise ValueError(f"points[{index}].x is outside the signed 64-bit range")
        if not _MIN_INT64 <= y_value <= _MAX_INT64:
            raise ValueError(f"points[{index}].y is outside the signed 64-bit range")
        if previous_x is not None and x_value <= previous_x:
            raise ValueError("point x coordinates must be strictly increasing")
        previous_x = x_value

    slopes: list[Fraction] = []
    for left_index in range(len(points) - 1):
        left_x, left_y = points[left_index]
        for right_index in range(left_index + 1, len(points)):
            right_x, right_y = points[right_index]
            slopes.append(
                Fraction(
                    right_y - left_y,
                    right_x - left_x,
                )
            )

    slopes.sort()
    middle = len(slopes) // 2
    if len(slopes) % 2:
        return slopes[middle]
    return (slopes[middle - 1] + slopes[middle]) / 2
```

## Example

```python
def reference_pairwise_median_slope(
    points: tuple[tuple[int, int], ...],
) -> Fraction:
    from itertools import combinations
    from statistics import median

    return median(
        Fraction(right_y - left_y, right_x - left_x)
        for (left_x, left_y), (right_x, right_y) in combinations(points, 2)
    )


two_points = ((-2, 1), (2, 9))
negative = ((0, 3), (1, 1), (2, -1))
robust = ((0, 0), (1, 1), (2, 2), (3, 10))
extrema = ((_MIN_INT64, _MIN_INT64), (_MAX_INT64, _MAX_INT64))
cases = (two_points, negative, robust, extrema)
estimates = tuple(estimate_exact_theil_sen_slope(points) for points in cases)
for points in cases:
    assert estimate_exact_theil_sen_slope(points) == reference_pairwise_median_slope(points)

translated = tuple((x_value + 10, y_value - 20) for x_value, y_value in robust)
tilted = tuple((x_value, y_value + 3 * x_value) for x_value, y_value in robust)
translation_invariant = estimate_exact_theil_sen_slope(
    translated
) == estimate_exact_theil_sen_slope(robust)
tilt_equivariant = (
    estimate_exact_theil_sen_slope(tilted) == estimate_exact_theil_sen_slope(robust) + 3
)

try:
    estimate_exact_theil_sen_slope(((0, 0), (1, True)))
except TypeError:
    boolean_rejected = True
else:
    boolean_rejected = False

assert (
    estimates,
    translation_invariant,
    tilt_equivariant,
    boolean_rejected,
) == (
    (Fraction(2), Fraction(-2), Fraction(13, 6), Fraction(1)),
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

For `P = n * (n - 1) // 2` point pairs, slope construction takes `O(P)`
`Fraction` operations, sorting takes `O(P log P)` fraction comparisons, and the
slope list uses `O(P)` memory. Integer subtraction, fraction normalization by
greatest common divisor, comparison cross-products, and the final midpoint can
all exceed signed 64 bits; their cost depends on operand bit lengths rather
than being constant.

The estimator reports only the median pairwise slope. It deliberately omits an
intercept, fitted values, residuals, confidence intervals, hypothesis tests,
weights, repeated or unordered `x` coordinates, floats, missing values,
multivariate regression, and inference claims. Although the median is less
sensitive than a mean to some extreme pairwise slopes, it does not remove the
need to inspect data quality and modelling assumptions.

## Related Snippets

<!-- catalog:related:start -->
- [Compute Exact Squared Pearson Correlation with Direction](compute-exact-squared-pearson-correlation-with-direction.md)
- [Compute an Exact Integer Median and Unscaled Median Absolute Deviation](compute-an-exact-integer-median-and-unscaled-median-absolute-deviation.md)
- [Calculate a Symmetrically Trimmed Mean](calculate-a-symmetrically-trimmed-mean.md)
<!-- catalog:related:end -->
