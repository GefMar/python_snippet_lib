---
title: "Bound the Exact Mean from Finite Integer Histogram Bins"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../observability-operations/locate-an-exact-quantile-rank-in-bounded-fixed-bucket-counts.md
  - ../observability-operations/merge-fixed-bound-histograms-at-their-shared-boundaries.md
---

# Bound the Exact Mean from Finite Integer Histogram Bins

## Idea and Problem

Find the smallest and largest possible mean of integer observations when only bounded bin counts are known.

Every observation contributes at least its bin's lower endpoint and at most
its upper endpoint. Multiply both endpoints by the bin count, sum them and
divide by the total count. Both bounds can occur: choose the lower endpoint
for every observation to attain the minimum, or choose the upper endpoints
to attain the maximum.

## When to Use

Use this when a data export retains frequencies for sorted, disjoint, finite
integer bins and the original measurements are unavailable. Supply each bin
as `(lower, upper, count)`, with both endpoints inclusive. Each observation
belongs to exactly one bin. A zero count is allowed, and no observations
produce `None` because the mean is undefined.

## Implementation

```python
from collections.abc import Iterable
from fractions import Fraction


def histogram_mean_bounds(
    bins: Iterable[tuple[int, int, int]],
) -> tuple[Fraction, Fraction] | None:
    """Return attainable (minimum, maximum) means for closed integer bins."""
    previous_upper: int | None = None
    minimum_sum = maximum_sum = total_count = 0
    for lower, upper, count in bins:
        if type(lower) is not int or type(upper) is not int or type(count) is not int:
            raise TypeError("bin endpoints and counts must be integers")
        if lower > upper or count < 0:
            raise ValueError("bins need ordered endpoints and non-negative counts")
        if previous_upper is not None and lower <= previous_upper:
            raise ValueError("closed bins must be ordered and disjoint")

        minimum_sum += lower * count
        maximum_sum += upper * count
        total_count += count
        previous_upper = upper

    if total_count == 0:
        return None
    return Fraction(minimum_sum, total_count), Fraction(maximum_sum, total_count)
```

## Example

```python
counts = ((-4, -2, 2), (0, 0, 3), (3, 4, 2))

assert (
    histogram_mean_bounds(iter(counts)),
    histogram_mean_bounds(((3, 3, 2), (5, 5, 1))),
    histogram_mean_bounds(((0, 2, 0),)),
) == ((Fraction(-2, 7), Fraction(4, 7)), (Fraction(11, 3), Fraction(11, 3)), None)
```

## Trade-offs and Limitations

The calculation visits each bin once and retains only running sums and the
previous upper endpoint. High counts do not expand into individual rows.
These endpoints give the tightest bounds from the bin counts alone. Extra
information about the observations can narrow the result.

Open-ended overflow or underflow bins need additional finite bounds before
using this recipe. Convert other boundary conventions to closed integer bins
first. An estimate such as the midpoint makes a separate modeling choice.

## Related Snippets

<!-- catalog:related:start -->
- [Locate an Exact Quantile Rank in Bounded Fixed-Bucket Counts](../observability-operations/locate-an-exact-quantile-rank-in-bounded-fixed-bucket-counts.md)
- [Merge Fixed-Bound Histograms at Their Shared Boundaries](../observability-operations/merge-fixed-bound-histograms-at-their-shared-boundaries.md)
<!-- catalog:related:end -->
