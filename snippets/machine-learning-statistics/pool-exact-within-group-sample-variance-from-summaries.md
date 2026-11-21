---
title: "Pool Exact Within-Group Sample Variance from Summaries"
snippet_type: recipe
use_cases:
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - accumulate-and-merge-finite-mean-and-variance-statistics-under-a-count-limit.md
---

# Pool Exact Within-Group Sample Variance from Summaries

## Idea and Problem

Combine groups' sample variance summaries into one degrees-of-freedom-weighted within-group variance.

Convert each group variance back to its within-group sum of squared deviations using one fewer than its sample count. Divide the sum of those deviations by the sum of the degrees of freedom. Differences between group means are deliberately absent.

## When to Use

Use this for exact rational summaries where every group has at least two observations and each variance used the sample denominator. It reports within-group spread rather than the variance of all raw observations treated as one group.

## Implementation

```python
from collections.abc import Iterable
from fractions import Fraction


def pooled_within_variance(groups: Iterable[tuple[int, Fraction]]) -> Fraction:
    squares = Fraction(0)
    degrees = 0

    for count, group_variance in groups:
        if type(count) is not int or count < 2:
            raise ValueError("each group needs at least two observations")
        if not isinstance(group_variance, Fraction) or group_variance < 0:
            raise ValueError("sample variance must be a nonnegative Fraction")
        squares += (count - 1) * group_variance
        degrees += count - 1

    if not degrees:
        raise ValueError("at least one group is required")
    return squares / degrees
```

## Example

```python
from statistics import variance

groups = ((0, 2), (97, 99, 101))
summaries = ((len(group), Fraction(variance(group))) for group in groups)

assert (
    pooled_within_variance(summaries),
    pooled_within_variance([(2, Fraction(1, 3))]),
) == (Fraction(10, 3), Fraction(1, 3))
```

## Trade-offs and Limitations

Do not combine population variance and sample variance summaries with this formula. It does not reconstruct total scatter or show that a common population variance is plausible. Fraction numerators and denominators can grow for many incompatible denominators.

## Related Snippets

<!-- catalog:related:start -->
- [Accumulate and Merge Finite Mean and Variance Statistics Under a Count Limit](accumulate-and-merge-finite-mean-and-variance-statistics-under-a-count-limit.md)
<!-- catalog:related:end -->
