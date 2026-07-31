---
title: "Compute Exact Squared Energy Distance Between Two Bounded Integer Samples"
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
  - compute-an-exact-normalized-one-dimensional-wasserstein-distance.md
  - compute-an-exact-crps-for-a-bounded-integer-ensemble-forecast.md
  - compute-an-exact-two-sample-kolmogorov-smirnov-distance-and-witness-for-bounded-integer-samples.md
---

# Compute Exact Squared Energy Distance Between Two Bounded Integer Samples

## Idea and Problem

Compare two empirical one-dimensional distributions with exact pairwise-distance evidence.

For first and second sample sizes `n` and `m`, the squared empirical energy
distance is the biased V-statistic

`2 * sum(abs(x - y)) / (n * m)
- sum(abs(x - x_prime)) / n**2
- sum(abs(y - y_prime)) / m**2`.

The cross term visits every pair from different samples. Each within-sample
term visits every ordered pair, including the zero-distance diagonal. Sorting
turns those quadratic sums into prefix-sum scans while `Fraction` preserves the
exact normalized result.

## When to Use

Use this calculation for two bounded, equally weighted samples of integer
observations when a reproducible descriptive distance is more useful than a
floating-point approximation. It is suitable for reference calculations,
distribution-regression fixtures, and small drift checks where duplicates
must retain their empirical multiplicity.

Use Wasserstein distance when transport work is the intended geometry, or a
Kolmogorov-Smirnov statistic when the largest cumulative-distribution gap is
the desired summary. Choose a statistics library when observations carry
weights, points are multidimensional, or the work requires an estimator
variant, permutation procedure, threshold, confidence statement, or
hypothesis test.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_SAMPLE_SIZE = 2_048
_MAX_COMBINED_SAMPLE_SIZE = 4_096


@dataclass(frozen=True, slots=True)
class SquaredEnergyDistanceEvidence:
    first_size: int
    second_size: int
    cross_absolute_distance_sum: int
    first_ordered_within_absolute_distance_sum: int
    second_ordered_within_absolute_distance_sum: int
    squared_energy_distance: Fraction


def _validated_sorted_sample(
    values: object,
    *,
    field: str,
) -> tuple[int, ...]:
    if type(values) is not tuple:
        raise TypeError(f"{field} must be an exact tuple")
    if not 1 <= len(values) <= _MAX_SAMPLE_SIZE:
        raise ValueError(f"{field} size is outside the supported range")

    for index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"{field}[{index}] must be an exact integer")
        if not _MIN_INT64 <= value <= _MAX_INT64:
            raise ValueError(f"{field}[{index}] is outside the signed 64-bit range")
    return tuple(sorted(values))


def _ordered_within_absolute_distance_sum(values: tuple[int, ...]) -> int:
    prefix_sum = 0
    unordered_sum = 0
    for index, value in enumerate(values):
        unordered_sum += index * value - prefix_sum
        prefix_sum += value
    return 2 * unordered_sum


def _cross_absolute_distance_sum(
    first: tuple[int, ...],
    second: tuple[int, ...],
) -> int:
    second_total = sum(second)
    second_prefix = 0
    second_index = 0
    cross_sum = 0

    for value in first:
        while second_index < len(second) and second[second_index] <= value:
            second_prefix += second[second_index]
            second_index += 1

        lower_sum = second_index * value - second_prefix
        upper_count = len(second) - second_index
        upper_sum = second_total - second_prefix - upper_count * value
        cross_sum += lower_sum + upper_sum

    return cross_sum


def exact_squared_energy_distance(
    first_values: tuple[int, ...],
    second_values: tuple[int, ...],
) -> SquaredEnergyDistanceEvidence:
    """Return exact biased empirical squared-energy-distance evidence."""
    first = _validated_sorted_sample(first_values, field="first_values")
    second = _validated_sorted_sample(second_values, field="second_values")
    if len(first) + len(second) > _MAX_COMBINED_SAMPLE_SIZE:
        raise ValueError("combined sample size exceeds the supported limit")

    first_within = _ordered_within_absolute_distance_sum(first)
    second_within = _ordered_within_absolute_distance_sum(second)
    cross = _cross_absolute_distance_sum(first, second)
    squared_distance = (
        Fraction(2 * cross, len(first) * len(second))
        - Fraction(first_within, len(first) ** 2)
        - Fraction(second_within, len(second) ** 2)
    )
    if squared_distance < 0:
        raise AssertionError("squared energy distance must be non-negative")

    return SquaredEnergyDistanceEvidence(
        first_size=len(first),
        second_size=len(second),
        cross_absolute_distance_sum=cross,
        first_ordered_within_absolute_distance_sum=first_within,
        second_ordered_within_absolute_distance_sum=second_within,
        squared_energy_distance=squared_distance,
    )
```

## Example

```python
def direct_quadratic_energy_evidence(
    first: tuple[int, ...],
    second: tuple[int, ...],
) -> SquaredEnergyDistanceEvidence:
    cross = sum(abs(left - right) for left in first for right in second)
    first_within = sum(abs(left - right) for left in first for right in first)
    second_within = sum(abs(left - right) for left in second for right in second)
    squared_distance = (
        Fraction(2 * cross, len(first) * len(second))
        - Fraction(first_within, len(first) ** 2)
        - Fraction(second_within, len(second) ** 2)
    )
    return SquaredEnergyDistanceEvidence(
        first_size=len(first),
        second_size=len(second),
        cross_absolute_distance_sum=cross,
        first_ordered_within_absolute_distance_sum=first_within,
        second_ordered_within_absolute_distance_sum=second_within,
        squared_energy_distance=squared_distance,
    )


def tiny_samples() -> tuple[tuple[int, ...], ...]:
    from itertools import product

    alphabet = (-2, 0, 3)
    return tuple(sample for length in range(1, 4) for sample in product(alphabet, repeat=length))


checked_pairs = 0
for first in tiny_samples():
    for second in tiny_samples():
        assert exact_squared_energy_distance(
            first,
            second,
        ) == direct_quadratic_energy_evidence(first, second)
        checked_pairs += 1

first = (-8, -1, -1, 7)
second = (-3, 2, 11)
baseline = exact_squared_energy_distance(first, second)
swapped = exact_squared_energy_distance(second, first)
permuted = exact_squared_energy_distance(tuple(reversed(first)), (11, -3, 2))
translated = exact_squared_energy_distance(
    tuple(value + 13 for value in first),
    tuple(value + 13 for value in second),
)
reflected = exact_squared_energy_distance(
    tuple(-value for value in first),
    tuple(-value for value in second),
)
replicated = exact_squared_energy_distance(
    tuple(value for value in first for _ in range(2)),
    tuple(value for value in second for _ in range(3)),
)
identity = exact_squared_energy_distance((-4, -4, 9), (9, -4, -4))
extremes = exact_squared_energy_distance((_MIN_INT64,), (_MAX_INT64,))
boundary = exact_squared_energy_distance(
    (0,) * _MAX_SAMPLE_SIZE,
    (0,) * _MAX_SAMPLE_SIZE,
)


def rejected(
    first_candidate: object,
    second_candidate: object,
    expected: type[Exception],
) -> bool:
    try:
        exact_squared_energy_distance(first_candidate, second_candidate)
    except expected:
        return True
    return False


invalid_calls = (
    ([], (0,), TypeError),
    ((), (0,), ValueError),
    ((0,) * (_MAX_SAMPLE_SIZE + 1), (0,), ValueError),
    ((True,), (0,), TypeError),
    ((_MIN_INT64 - 1,), (0,), ValueError),
    ((0,), (_MAX_INT64 + 1,), ValueError),
)
rejected_count = sum(
    rejected(first_candidate, second_candidate, expected)
    for first_candidate, second_candidate, expected in invalid_calls
)

extreme_distance = 2 * (_MAX_INT64 - _MIN_INT64)
assert (
    checked_pairs,
    baseline.squared_energy_distance,
    swapped.squared_energy_distance,
    permuted.squared_energy_distance,
    translated.squared_energy_distance,
    reflected.squared_energy_distance,
    replicated.squared_energy_distance,
    identity.squared_energy_distance,
    extremes,
    boundary.squared_energy_distance,
    rejected_count,
) == (
    len(tiny_samples()) ** 2,
    baseline.squared_energy_distance,
    baseline.squared_energy_distance,
    baseline.squared_energy_distance,
    baseline.squared_energy_distance,
    baseline.squared_energy_distance,
    baseline.squared_energy_distance,
    Fraction(0),
    SquaredEnergyDistanceEvidence(
        first_size=1,
        second_size=1,
        cross_absolute_distance_sum=_MAX_INT64 - _MIN_INT64,
        first_ordered_within_absolute_distance_sum=0,
        second_ordered_within_absolute_distance_sum=0,
        squared_energy_distance=Fraction(extreme_distance),
    ),
    Fraction(0),
    len(invalid_calls),
)
```

## Trade-offs and Limitations

For sample sizes `n` and `m`, validation and sorting take
`O(n log n + m log m)` time. The within-sample prefix scans and merged cross
scan take `O(n + m)` time. Sorted copies use `O(n + m)` additional references.
The caps admit at most 2,048 observations per sample and 4,096 combined.

Every input value is signed-64, but one absolute difference can span the full
unsigned-64 range and accumulated sums can be much wider. Python integers and
`Fraction` keep those intermediate and normalized values exact. Duplicate
observations retain multiplicity, while sample order, a shared translation, a
shared reflection, and uniform replication do not change the normalized
distance.

This is the biased empirical V-statistic: each within sum covers ordered pairs,
includes zero diagonals, and uses the square of its sample size. It is not the
unbiased U-statistic with diagonal-adjusted denominators. The returned value is
the squared energy distance; the function deliberately does not take a square
root.

The result is descriptive evidence, not a hypothesis test. It does not provide
a p-value, threshold, confidence interval, direction, attribution, or causal
claim. Weights, floats, missing observations, multidimensional points,
streaming updates, and approximate large-sample computation are outside this
bounded contract.

## Related Snippets

<!-- catalog:related:start -->
- [Compute an Exact Normalized One-Dimensional Wasserstein Distance](compute-an-exact-normalized-one-dimensional-wasserstein-distance.md)
- [Compute an Exact CRPS for a Bounded Integer Ensemble Forecast](compute-an-exact-crps-for-a-bounded-integer-ensemble-forecast.md)
- [Compute an Exact Two-Sample Kolmogorov-Smirnov Distance and Witness for Bounded Integer Samples](compute-an-exact-two-sample-kolmogorov-smirnov-distance-and-witness-for-bounded-integer-samples.md)
<!-- catalog:related:end -->
