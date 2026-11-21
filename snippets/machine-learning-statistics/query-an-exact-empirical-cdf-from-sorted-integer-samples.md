---
title: "Query an Exact Empirical CDF from Sorted Integer Samples"
snippet_type: recipe
use_cases:
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - fit-and-apply-fixed-quantile-clipping-bounds.md
  - ../algorithms-data-structures/find-the-matching-range-for-a-literal-prefix-in-sorted-strings.md
---

# Query an Exact Empirical CDF from Sorted Integer Samples

## Idea and Problem

Measure the fraction of observed integer samples at or below each requested threshold.

Sort the sample once. The insertion point after existing equal values counts all observations that belong at or below a threshold. Divide that count by the total as a fraction instead of losing precision to floats.

## When to Use

Use this to query a fixed sample at several thresholds, retaining the weight of repeated observations. This is the right-continuous empirical distribution function, not a density estimate.

## Implementation

```python
from bisect import bisect_right
from collections.abc import Iterable
from fractions import Fraction


def empirical_cdf(samples: Iterable[int], thresholds: Iterable[int]) -> tuple[Fraction, ...]:
    ordered = sorted(samples)
    if not ordered:
        raise ValueError("the sample must not be empty")

    count = len(ordered)
    return tuple(Fraction(bisect_right(ordered, threshold), count) for threshold in thresholds)
```

## Example

```python
sample = [9, -1, 2, 2, 6]
scores = empirical_cdf(iter(sample), (-4, -1, 2, 5, 9, 100))
assert (scores, sample, empirical_cdf(sample, ())) == (
    (Fraction(0), Fraction(1, 5), Fraction(3, 5), Fraction(3, 5), Fraction(1), Fraction(1)),
    [9, -1, 2, 2, 6],
    (),
)
```

## Trade-offs and Limitations

Sorting takes O(n log n) comparisons and stores the sample in memory. Each threshold uses a binary search; this function stores the returned fractions and sorts again on another call. It describes the observations, not uncertainty about the population that produced them.

## Related Snippets

<!-- catalog:related:start -->
- [Fit and Apply Fixed Quantile Clipping Bounds](fit-and-apply-fixed-quantile-clipping-bounds.md)
- [Find the Matching Range for a Literal Prefix in Sorted Strings](../algorithms-data-structures/find-the-matching-range-for-a-literal-prefix-in-sorted-strings.md)
<!-- catalog:related:end -->
