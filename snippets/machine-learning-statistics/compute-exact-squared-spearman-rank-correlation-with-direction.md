---
title: "Compute Exact Squared Spearman Rank Correlation with Direction"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-exact-squared-pearson-correlation-with-direction.md
  - compute-exact-binary-roc-auc-with-tied-integer-scores.md
  - estimate-an-exact-theil-sen-slope-from-bounded-integer-points.md
---

# Compute Exact Squared Spearman Rank Correlation with Direction

## Idea and Problem

Measure monotonic association between paired bounded integers exactly while handling ties and retaining the direction hidden by squaring.

Spearman correlation is Pearson correlation applied to ranks. Each tied value
receives its group's average rank; storing twice that midrank keeps every rank
integer. Centering those doubled ranks around their known mean produces exact
integer cross-deviation and squared-deviation sums, so rho-squared is a
`Fraction` and the cross term supplies its sign.

## When to Use

Use this calculation for a bounded, completely observed set of paired ordinal
or integer measurements when monotonic association matters more than the
original value spacing. It is useful when ties are meaningful and an exact,
reproducible squared coefficient is preferable to a floating-point estimate.

Both coordinates must vary for rank correlation to be defined. Use a
statistical package when inputs are floating point, observations have missing
values or weights, uncertainty estimates or p-values are required, or a
different tie convention or correlation statistic is part of the analysis.

## Implementation

```python
from fractions import Fraction

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_RANK_OBSERVATIONS = 100_000


def _validated_rank_values(value: object, *, name: str) -> tuple[int, ...]:
    if type(value) is not tuple:
        raise TypeError(f"{name} must be an exact tuple")
    if not 2 <= len(value) <= _MAX_RANK_OBSERVATIONS:
        raise ValueError(f"{name} count is outside the supported range")
    for index, item in enumerate(value):
        if type(item) is not int:
            raise TypeError(f"{name}[{index}] must be an exact non-boolean integer")
        if not _MIN_INT64 <= item <= _MAX_INT64:
            raise ValueError(f"{name}[{index}] is outside the signed 64-bit range")
    return value


def _doubled_midranks(values: tuple[int, ...]) -> list[int]:
    ordered_indexes = sorted(
        range(len(values)),
        key=lambda index: (values[index], index),
    )
    ranks = [0] * len(values)
    start = 0
    while start < len(values):
        stop = start + 1
        while (
            stop < len(values) and values[ordered_indexes[stop]] == values[ordered_indexes[start]]
        ):
            stop += 1

        doubled_midrank = start + 1 + stop
        for position in range(start, stop):
            ranks[ordered_indexes[position]] = doubled_midrank
        start = stop
    return ranks


def exact_squared_spearman_correlation(
    x_values: tuple[int, ...],
    y_values: tuple[int, ...],
) -> tuple[Fraction | None, int | None]:
    """Return exact rho-squared and its direction, or two None values."""
    x = _validated_rank_values(x_values, name="x_values")
    y = _validated_rank_values(y_values, name="y_values")
    if len(x) != len(y):
        raise ValueError("x_values and y_values must have equal lengths")

    x_ranks = _doubled_midranks(x)
    y_ranks = _doubled_midranks(y)
    rank_center = len(x) + 1
    x_squared_deviation = 0
    y_squared_deviation = 0
    cross_deviation = 0

    for x_rank, y_rank in zip(x_ranks, y_ranks, strict=True):
        x_deviation = x_rank - rank_center
        y_deviation = y_rank - rank_center
        x_squared_deviation += x_deviation * x_deviation
        y_squared_deviation += y_deviation * y_deviation
        cross_deviation += x_deviation * y_deviation

    if x_squared_deviation == 0 or y_squared_deviation == 0:
        return None, None

    direction = (cross_deviation > 0) - (cross_deviation < 0)
    return (
        Fraction(
            cross_deviation * cross_deviation,
            x_squared_deviation * y_squared_deviation,
        ),
        direction,
    )
```

## Example

```python
def fraction_midranks(values: tuple[int, ...]) -> tuple[Fraction, ...]:
    return tuple(
        Fraction(
            2 * sum(other < value for other in values)
            + sum(other == value for other in values)
            + 1,
            2,
        )
        for value in values
    )


def spearman_from_independent_ranks(
    x_values: tuple[int, ...],
    y_values: tuple[int, ...],
) -> tuple[Fraction | None, int | None]:
    x_ranks = fraction_midranks(x_values)
    y_ranks = fraction_midranks(y_values)
    x_mean = sum(x_ranks, Fraction()) / len(x_ranks)
    y_mean = sum(y_ranks, Fraction()) / len(y_ranks)
    x_deviations = tuple(rank - x_mean for rank in x_ranks)
    y_deviations = tuple(rank - y_mean for rank in y_ranks)
    x_squared = sum(value * value for value in x_deviations)
    y_squared = sum(value * value for value in y_deviations)
    if x_squared == 0 or y_squared == 0:
        return None, None
    cross = sum(
        x_value * y_value for x_value, y_value in zip(x_deviations, y_deviations, strict=True)
    )
    return cross * cross / (x_squared * y_squared), (cross > 0) - (cross < 0)


def exercise_tied_short_sequences() -> int:
    from itertools import product

    checked = 0
    for length in range(2, 6):
        sequences = tuple(product((-1, 0, 2), repeat=length))
        for x_values in sequences:
            for y_values in sequences:
                assert exact_squared_spearman_correlation(
                    x_values, y_values
                ) == spearman_from_independent_ranks(x_values, y_values)
                checked += 1
    return checked


def raises(error_type: type[Exception], operation: object) -> bool:
    try:
        operation()  # type: ignore[operator]
    except error_type:
        return True
    return False


positive = exact_squared_spearman_correlation((1, 2, 2, 4), (10, 20, 20, 30))
negative = exact_squared_spearman_correlation((1, 2, 2, 4), (30, 20, 20, 10))
zero = exact_squared_spearman_correlation((-1, 0, 1), (1, -2, 1))
undefined = exact_squared_spearman_correlation((4, 4, 4), (1, 2, 3))

paired_x = (10, 10, 20, 30, 30, 40)
paired_y = (3, 1, 2, 2, 5, 4)
pair_order = (5, 0, 3, 1, 4, 2)
paired = exact_squared_spearman_correlation(paired_x, paired_y)
permuted_pairs = exact_squared_spearman_correlation(
    tuple(paired_x[index] for index in pair_order),
    tuple(paired_y[index] for index in pair_order),
)

maximum_values = tuple(range(_MAX_RANK_OBSERVATIONS))
maximum_reverse = tuple(reversed(maximum_values))
maximum_positive = exact_squared_spearman_correlation(maximum_values, maximum_values)
maximum_negative = exact_squared_spearman_correlation(maximum_values, maximum_reverse)
maximum_constant = exact_squared_spearman_correlation(
    (0,) * _MAX_RANK_OBSERVATIONS,
    maximum_values,
)

assert (
    exercise_tied_short_sequences(),
    positive,
    negative,
    zero,
    undefined,
    paired == spearman_from_independent_ranks(paired_x, paired_y),
    permuted_pairs == paired,
    maximum_positive,
    maximum_negative,
    maximum_constant,
    exact_squared_spearman_correlation(
        (_MIN_INT64, 0, _MAX_INT64),
        (_MAX_INT64, 0, _MIN_INT64),
    ),
    raises(TypeError, lambda: exact_squared_spearman_correlation((1, True), (1, 2))),
    raises(ValueError, lambda: exact_squared_spearman_correlation((1, 2), (1, 2, 3))),
) == (
    66_420,
    (Fraction(1), 1),
    (Fraction(1), -1),
    (Fraction(0), 0),
    (None, None),
    True,
    True,
    (Fraction(1), 1),
    (Fraction(1), -1),
    (None, None),
    (Fraction(1), -1),
    True,
    True,
)
```

## Trade-offs and Limitations

Validation and centered accumulation take `O(N)` arithmetic work. Building two
stable value/index orders dominates at `O(N log N)` time; the orders, doubled
rank arrays, and sorting state use `O(N)` auxiliary memory. Python integers
keep every rank sum and product exact, but their bit lengths grow with the
bounded observation count, and final `Fraction` reduction is not a unit-cost
operation.

The returned fraction is Spearman's rho squared and lies between zero and one.
The accompanying `-1`, `0`, or `1` is the sign of rank covariance; it is zero
only when the rank covariance is zero. When either coordinate is constant, its
rank variance is zero and both fields are `None`. Average ranks give every
member of a tied group the same doubled midrank without disturbing pair
alignment.

Rank correlation ignores numeric distance and describes monotonic association,
not causation or calibration. Squaring discards signed magnitude, so direction
is reported separately. The function does not return signed irrational rho,
accept floats, missing values or weights, calculate p-values or confidence
intervals, choose another tie policy, or compute Pearson or Kendall
correlation.

## Related Snippets

<!-- catalog:related:start -->
- [Compute Exact Squared Pearson Correlation with Direction](compute-exact-squared-pearson-correlation-with-direction.md)
- [Compute Exact Binary ROC AUC with Tied Integer Scores](compute-exact-binary-roc-auc-with-tied-integer-scores.md)
- [Estimate an Exact Theil-Sen Slope from Bounded Integer Points](estimate-an-exact-theil-sen-slope-from-bounded-integer-points.md)
<!-- catalog:related:end -->
