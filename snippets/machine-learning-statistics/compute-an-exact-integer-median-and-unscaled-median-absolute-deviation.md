---
title: "Compute an Exact Integer Median and Unscaled Median Absolute Deviation"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-exact-full-window-trailing-medians-for-bounded-integers.md
  - calculate-a-symmetrically-trimmed-mean.md
  - flag-groupwise-numeric-outliers-with-iqr-fences.md
---

# Compute an Exact Integer Median and Unscaled Median Absolute Deviation

## Idea and Problem

Describe the center and raw median absolute deviation of one bounded integer sample without converting either statistic to binary floating point.

Sort exact `Fraction` representations to find the sample median, then take the
median of every absolute distance from that center. Keeping the even-sample
midpoints as fractions preserves the mathematical result instead of rounding
it to an integer or float.

## When to Use

Use these statistics for a small in-memory integer sample when a robust center
and an unscaled measure of typical absolute distance are useful. The median
absolute deviation, or MAD, is less affected by isolated extremes than a
mean-based spread, while exact fractions make later conversion an explicit
caller decision.

Choose a domain-specific statistical implementation when observations have
weights, missing values, windows, or an established scaling and threshold
policy. A raw MAD is a descriptive statistic; it is not by itself an outlier
classification rule or an estimate of standard deviation.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_INTEGER_SAMPLE_COUNT = 10_000


@dataclass(frozen=True, slots=True)
class ExactMedianMad:
    median: Fraction
    raw_mad: Fraction


def _median_of_sorted_fractions(values: list[Fraction]) -> Fraction:
    midpoint = len(values) // 2
    if len(values) % 2:
        return values[midpoint]
    return (values[midpoint - 1] + values[midpoint]) / 2


def exact_integer_median_mad(values: tuple[int, ...]) -> ExactMedianMad:
    """Return the exact median and unscaled median absolute deviation."""
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    if not 1 <= len(values) <= _MAX_INTEGER_SAMPLE_COUNT:
        raise ValueError("value count is outside the supported range")

    for index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"values[{index}] must be an exact integer")
        if not _MIN_INT64 <= value <= _MAX_INT64:
            raise ValueError(f"values[{index}] is outside the signed 64-bit range")

    ordered = sorted(Fraction(value) for value in values)
    center = _median_of_sorted_fractions(ordered)
    absolute_deviations = sorted(abs(value - center) for value in ordered)
    return ExactMedianMad(
        median=center,
        raw_mad=_median_of_sorted_fractions(absolute_deviations),
    )
```

## Example

```python
fractional = exact_integer_median_mad((0, 0, 1, 3))
zero_without_equality = exact_integer_median_mad((1, 1, 1, 10, 20))
extremes = exact_integer_median_mad((_MIN_INT64, _MAX_INT64))

try:
    exact_integer_median_mad((1, True, 3))
except TypeError:
    boolean_rejected = True
else:
    boolean_rejected = False

assert (fractional, zero_without_equality, extremes, boolean_rejected) == (
    ExactMedianMad(median=Fraction(1, 2), raw_mad=Fraction(1, 2)),
    ExactMedianMad(median=Fraction(1), raw_mad=Fraction(0)),
    ExactMedianMad(
        median=Fraction(-1, 2),
        raw_mad=Fraction((1 << 64) - 1, 2),
    ),
    True,
)
```

## Trade-offs and Limitations

Two sorts take `O(n log n)` time, and the ordered values plus deviations use
`O(n)` `Fraction` objects. Full validation happens before either sort. Derived
subtraction and midpoint arithmetic remain exact even when a deviation or
intermediate numerator exceeds the signed 64-bit input range.

The reported MAD is raw and unscaled: no normal-consistency factor is applied.
A zero raw MAD means that at least half the absolute deviations concentrate at
zero under the median rule; it does not prove that every observation is equal.
The function defines no modified z-score, outlier threshold, float conversion,
serialization, missing-value, weighting, rolling-window, streaming, or mutable
update policy.

## Related Snippets

<!-- catalog:related:start -->
- [Compute Exact Full-Window Trailing Medians for Bounded Integers](compute-exact-full-window-trailing-medians-for-bounded-integers.md)
- [Calculate a Symmetrically Trimmed Mean](calculate-a-symmetrically-trimmed-mean.md)
- [Flag Groupwise Numeric Outliers with IQR Fences](flag-groupwise-numeric-outliers-with-iqr-fences.md)
<!-- catalog:related:end -->
