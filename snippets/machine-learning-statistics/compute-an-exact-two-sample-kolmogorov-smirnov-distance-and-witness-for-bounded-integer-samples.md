---
title: "Compute an Exact Two-Sample Kolmogorov-Smirnov Distance and Witness for Bounded Integer Samples"
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
  - compute-exact-squared-spearman-rank-correlation-with-direction.md
  - build-an-exact-kaplan-meier-curve-for-bounded-right-censored-data.md
---

# Compute an Exact Two-Sample Kolmogorov-Smirnov Distance and Witness for Bounded Integer Samples

## Idea and Problem

Measure the largest exact gap between two empirical cumulative distribution functions and identify its earliest observed location.

Sort both integer samples, sweep the pooled observed values, and consume every
copy of one value from both samples before comparing their right-continuous
empirical CDFs. With sample sizes `n` and `m`, the signed gap numerator is
`seen_first * m - seen_second * n` over the common denominator `n * m`.

Updating the result only for a strictly larger absolute numerator selects the
smallest observed value when several locations attain the same distance.

## When to Use

Use this statistic to compare two bounded empirical integer distributions when
exact arithmetic, duplicate handling, and a deterministic witness location are
useful for tests or diagnostics. The direction records which empirical CDF is
higher at that location.

Use a statistical library when a p-value, critical value, confidence band,
one-sided alternative, weighting, censoring, or a designed hypothesis test is
required. Ties and discrete data need an inference policy beyond calculating
the descriptive distance.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction
from typing import Literal, TypeAlias

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_KS_SAMPLE_SIZE = 100_000

KsDirection: TypeAlias = Literal["first-above", "second-above", "equal"]


@dataclass(frozen=True, slots=True)
class ExactTwoSampleKsResult:
    distance: Fraction
    at_value: int
    direction: KsDirection


def exact_two_sample_ks_distance(
    first: tuple[int, ...],
    second: tuple[int, ...],
) -> ExactTwoSampleKsResult:
    """Return exact two-sample empirical-CDF distance and earliest witness."""
    for sample_name, sample in (("first", first), ("second", second)):
        if type(sample) is not tuple:
            raise TypeError(f"{sample_name} must be an exact tuple")
        if not 1 <= len(sample) <= _MAX_KS_SAMPLE_SIZE:
            raise ValueError(f"{sample_name} size is outside the supported range")
        for index, value in enumerate(sample):
            if type(value) is not int:
                raise TypeError(f"{sample_name}[{index}] must be an exact integer")
            if not _MIN_INT64 <= value <= _MAX_INT64:
                raise ValueError(f"{sample_name}[{index}] is outside the signed 64-bit range")

    ordered_first = sorted(first)
    ordered_second = sorted(second)
    first_size = len(ordered_first)
    second_size = len(ordered_second)
    first_index = 0
    second_index = 0
    best_absolute_numerator = -1
    best_signed_numerator = 0
    best_value: int | None = None

    while first_index < first_size or second_index < second_size:
        if second_index == second_size or (
            first_index < first_size and ordered_first[first_index] < ordered_second[second_index]
        ):
            value = ordered_first[first_index]
        else:
            value = ordered_second[second_index]

        while first_index < first_size and ordered_first[first_index] == value:
            first_index += 1
        while second_index < second_size and ordered_second[second_index] == value:
            second_index += 1

        signed_numerator = first_index * second_size - second_index * first_size
        absolute_numerator = abs(signed_numerator)
        if absolute_numerator > best_absolute_numerator:
            best_absolute_numerator = absolute_numerator
            best_signed_numerator = signed_numerator
            best_value = value

    if best_value is None:
        raise AssertionError("non-empty samples must establish a witness")
    if best_signed_numerator > 0:
        direction: KsDirection = "first-above"
    elif best_signed_numerator < 0:
        direction = "second-above"
    else:
        direction = "equal"

    return ExactTwoSampleKsResult(
        distance=Fraction(
            best_absolute_numerator,
            first_size * second_size,
        ),
        at_value=best_value,
        direction=direction,
    )
```

## Example

```python
ordinary = exact_two_sample_ks_distance(
    (1, 2, 2, 3),
    (2, 3, 4),
)
swapped = exact_two_sample_ks_distance(
    (2, 3, 4),
    (1, 2, 2, 3),
)
identical = exact_two_sample_ks_distance(
    (1, 1, 2),
    (2, 1, 1),
)
earliest_tie = exact_two_sample_ks_distance(
    (0, 2),
    (1, 3),
)
signed_boundary = exact_two_sample_ks_distance(
    (_MIN_INT64,),
    (_MAX_INT64,),
)

assert ordinary == ExactTwoSampleKsResult(
    Fraction(5, 12),
    2,
    "first-above",
)
assert swapped == ExactTwoSampleKsResult(
    Fraction(5, 12),
    2,
    "second-above",
)
assert identical == ExactTwoSampleKsResult(Fraction(0), 1, "equal")
assert earliest_tie == ExactTwoSampleKsResult(
    Fraction(1, 2),
    0,
    "first-above",
)
assert signed_boundary == ExactTwoSampleKsResult(
    Fraction(1),
    _MIN_INT64,
    "first-above",
)
```

## Trade-offs and Limitations

Sorting samples of sizes `n` and `m` costs
`O(n log n + m log m)` time and `O(n + m)` copied references. The merged
sweep is linear. Numerators and the common denominator are exact Python
integers, and the returned `Fraction` is reduced.

Equal pooled values must be consumed as complete groups before evaluating the
right-continuous CDFs. Comparing inside a tie group would create
iteration-order-dependent gaps that do not occur at any valid CDF location.
Sample replication leaves the distance unchanged, although sorting still
processes every observation.

This is a descriptive two-sided distance, not an exact hypothesis test. It
returns no p-value, critical value, confidence band, significance decision, or
one-sided statistic. In particular, exact arithmetic does not make
continuous-null p-value formulas appropriate for discrete samples with ties.

## Related Snippets

<!-- catalog:related:start -->
- [Compute an Exact Paired Two-Sided Sign Test](compute-an-exact-paired-two-sided-sign-test.md)
- [Compute Exact Squared Spearman Rank Correlation with Direction](compute-exact-squared-spearman-rank-correlation-with-direction.md)
- [Build an Exact Kaplan-Meier Curve for Bounded Right-Censored Data](build-an-exact-kaplan-meier-curve-for-bounded-right-censored-data.md)
<!-- catalog:related:end -->
