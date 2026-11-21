---
title: "Combine Equal-Distance Speeds with an Exact Harmonic Mean"
snippet_type: recipe
use_cases:
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../data-processing/integrate-regular-rate-samples-across-explicit-time-buckets.md
  - compute-an-exact-integer-median-and-unscaled-median-absolute-deviation.md
---

# Combine Equal-Distance Speeds with an Exact Harmonic Mean

## Idea and Problem

Combine different speeds over equal-distance segments by dividing total distance by total time.

A slower segment takes more time, so the arithmetic mean of its speed and a faster segment's speed overstates the overall speed. Use the harmonic mean, keeping each input quantity rational.

## When to Use

Use this when all segments have exactly the same positive length and their speeds are positive and expressed in the same units. Each rate describes the entire segment.

## Implementation

```python
from collections.abc import Iterable
from fractions import Fraction
from statistics import harmonic_mean


def equal_distance_speed(speeds: Iterable[Fraction]) -> Fraction:
    values = tuple(speeds)
    if not values or any(not isinstance(value, Fraction) or value <= 0 for value in values):
        raise ValueError("expected one or more positive rational speeds")

    return harmonic_mean(values)
```

## Example

```python
two_segments = equal_distance_speed([Fraction(30), Fraction(60)])
uneven_rates = equal_distance_speed([Fraction(1, 2), Fraction(3, 2)])
assert (two_segments, uneven_rates, equal_distance_speed([Fraction(7, 3)])) == (
    Fraction(40),
    Fraction(3, 4),
    Fraction(7, 3),
)
```

## Trade-offs and Limitations

Do not use this equal-weight formula for unequal distances or to combine samples taken over equal time intervals. A stop needs a time contribution instead of a positive moving-speed rate. Exact fractions can grow in size when denominators differ.

## Related Snippets

<!-- catalog:related:start -->
- [Integrate Regular Rate Samples Across Explicit Time Buckets](../data-processing/integrate-regular-rate-samples-across-explicit-time-buckets.md)
- [Compute an Exact Integer Median and Unscaled Median Absolute Deviation](compute-an-exact-integer-median-and-unscaled-median-absolute-deviation.md)
<!-- catalog:related:end -->
