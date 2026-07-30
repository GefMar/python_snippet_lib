---
title: "Compute an Exact Normalized One-Dimensional Wasserstein Distance"
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
  - compute-an-exact-two-sample-kolmogorov-smirnov-distance-and-witness-for-bounded-integer-samples.md
  - select-the-lower-weighted-median-of-bounded-integer-observations.md
  - measure-drift-between-two-fixed-bin-count-distributions-with-psi.md
---

# Compute an Exact Normalized One-Dimensional Wasserstein Distance

## Idea and Problem

Measure the exact transport distance between two bounded weighted integer distributions even when their raw total weights differ.

For distributions on a line, the Wasserstein-1 distance is the integral of the
absolute gap between their cumulative distribution functions. A merged sweep
visits each support position once. Between consecutive positions, both CDFs
are constant, so their exact gap is multiplied by that integer interval width.

Each distribution is normalized by its own total weight. The running CDF gap
therefore has numerator
`first_seen * second_total - second_seen * first_total` over the common
denominator `first_total * second_total`.

## When to Use

Use this algorithm for small weighted histograms or empirical distributions
whose support is already strictly sorted and where reproducible exact
arithmetic is more useful than a floating-point approximation. It is useful
for reference tests, drift diagnostics, and comparing unequal-sized samples
after aggregating repeated integer observations into positive weights.

Use a statistics or optimal-transport library for multidimensional points,
continuous densities, inference, regularized transport, approximate solvers,
or large unsorted observations. Decide separately whether normalizing away a
difference in total mass is valid for the surrounding problem.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_COMBINED_SUPPORT_POINTS = 4_096


@dataclass(frozen=True, slots=True)
class WeightedSupportPoint:
    position: int
    weight: int


def _validate_weighted_support(
    name: str,
    support: object,
) -> tuple[tuple[WeightedSupportPoint, ...], int]:
    if type(support) is not tuple:
        raise TypeError(f"{name} must be an exact tuple")
    if not support:
        raise ValueError(f"{name} must not be empty")

    total_weight = 0
    previous_position: int | None = None
    for index, point in enumerate(support):
        if type(point) is not WeightedSupportPoint:
            raise TypeError(f"{name}[{index}] must be an exact WeightedSupportPoint")
        if type(point.position) is not int:
            raise TypeError(f"{name}[{index}].position must be an exact integer")
        if not _MIN_INT64 <= point.position <= _MAX_INT64:
            raise ValueError(f"{name}[{index}].position is outside the signed 64-bit range")
        if previous_position is not None and point.position <= previous_position:
            raise ValueError(f"{name} positions must be strictly increasing")
        previous_position = point.position

        if type(point.weight) is not int:
            raise TypeError(f"{name}[{index}].weight must be an exact integer")
        if not 1 <= point.weight <= _MAX_INT64:
            raise ValueError(f"{name}[{index}].weight is outside the positive 64-bit range")
        total_weight += point.weight
        if total_weight > _MAX_INT64:
            raise ValueError(f"{name} total weight exceeds the signed 64-bit range")

    return support, total_weight


def exact_normalized_wasserstein_distance(
    first: tuple[WeightedSupportPoint, ...],
    second: tuple[WeightedSupportPoint, ...],
) -> Fraction:
    """Return the exact normalized one-dimensional Wasserstein-1 distance."""
    first, first_total = _validate_weighted_support("first", first)
    second, second_total = _validate_weighted_support("second", second)
    if len(first) + len(second) > _MAX_COMBINED_SUPPORT_POINTS:
        raise ValueError("combined support size exceeds the supported limit")

    first_index = 0
    second_index = 0
    first_seen = 0
    second_seen = 0
    previous_position: int | None = None
    distance_numerator = 0

    while first_index < len(first) or second_index < len(second):
        if second_index == len(second) or (
            first_index < len(first) and first[first_index].position < second[second_index].position
        ):
            position = first[first_index].position
        elif first_index == len(first) or (
            second[second_index].position < first[first_index].position
        ):
            position = second[second_index].position
        else:
            position = first[first_index].position

        if previous_position is not None:
            cdf_gap_numerator = first_seen * second_total - second_seen * first_total
            distance_numerator += abs(cdf_gap_numerator) * (position - previous_position)

        if first_index < len(first) and first[first_index].position == position:
            first_seen += first[first_index].weight
            first_index += 1
        if second_index < len(second) and second[second_index].position == position:
            second_seen += second[second_index].weight
            second_index += 1
        previous_position = position

    if first_seen != first_total or second_seen != second_total:
        raise RuntimeError("support sweep did not consume both distributions")
    return Fraction(distance_numerator, first_total * second_total)


```

## Example

```python
first = (
    WeightedSupportPoint(0, 1),
    WeightedSupportPoint(10, 1),
)
second = (
    WeightedSupportPoint(2, 1),
    WeightedSupportPoint(8, 3),
)

distance = exact_normalized_wasserstein_distance(first, second)
swapped = exact_normalized_wasserstein_distance(second, first)
scaled = exact_normalized_wasserstein_distance(
    (
        WeightedSupportPoint(0, 7),
        WeightedSupportPoint(10, 7),
    ),
    second,
)
translated = exact_normalized_wasserstein_distance(
    tuple(WeightedSupportPoint(point.position - 20, point.weight) for point in first),
    tuple(WeightedSupportPoint(point.position - 20, point.weight) for point in second),
)

assert (distance, swapped, scaled, translated) == (
    Fraction(7, 2),
    Fraction(7, 2),
    Fraction(7, 2),
    Fraction(7, 2),
)
```

## Trade-offs and Limitations

For support sizes `n` and `m`, validation and the merged sweep use `O(n + m)`
time and `O(1)` auxiliary state beyond the inputs. Python integers and the
returned `Fraction` preserve exactness, but their arithmetic cost grows with
the magnitude of positions, weights, cumulative products, and interval sums.

Strict ordering and unique positions are part of the input contract. Aggregate
duplicate observations and sort them before calling; doing that here would
change both the memory bound and the taught linear-sweep precondition. Scaling
all weights in either distribution leaves the normalized result unchanged.

This is a descriptive normalized Wasserstein-1 distance. It does not return a
transport plan, p-value, confidence interval, threshold decision, signed
direction, or witness interval. Unlike the Kolmogorov-Smirnov distance, it
integrates a CDF gap across spatial distance instead of selecting its largest
height. Unlike fixed-bin PSI, it requires no bins and performs no logarithmic
floating-point calculation.

## Related Snippets

<!-- catalog:related:start -->
- [Compute an Exact Two-Sample Kolmogorov-Smirnov Distance and Witness for Bounded Integer Samples](compute-an-exact-two-sample-kolmogorov-smirnov-distance-and-witness-for-bounded-integer-samples.md)
- [Select the Lower Weighted Median of Bounded Integer Observations](select-the-lower-weighted-median-of-bounded-integer-observations.md)
- [Measure Drift Between Two Fixed-Bin Count Distributions with PSI](measure-drift-between-two-fixed-bin-count-distributions-with-psi.md)
<!-- catalog:related:end -->
