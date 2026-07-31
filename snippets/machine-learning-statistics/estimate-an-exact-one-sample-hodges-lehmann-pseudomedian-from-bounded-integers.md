---
title: "Estimate an Exact One-Sample Hodges-Lehmann Pseudomedian from Bounded Integers"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - compute-an-exact-integer-median-and-unscaled-median-absolute-deviation.md
  - estimate-an-exact-theil-sen-slope-from-bounded-integer-points.md
  - compute-an-exact-two-sided-wilcoxon-signed-rank-randomization-test-for-bounded-integer-pairs.md
---

# Estimate an Exact One-Sample Hodges-Lehmann Pseudomedian from Bounded Integers

## Idea and Problem

Estimate one distribution's location from the median of every self-inclusive pairwise average while preserving half- and quarter-integer results exactly.

For observations `x[0]` through `x[n - 1]`, the one-sample Hodges-Lehmann
pseudomedian uses the `n * (n + 1) / 2` Walsh averages
`(x[i] + x[j]) / 2` for `i <= j`. Sorting the integer Walsh sums is sufficient:
division by two preserves their order. If the number of averages is even, the
two central averages are averaged once more.

The result retains both central Walsh averages as evidence, so callers can see
whether the final estimate came from one central value or the midpoint of two.

## When to Use

Use this estimator for a bounded integer sample when a robust location summary
should reflect pairwise averages rather than only the central observation. It
is useful for small exploratory datasets or for checking another
implementation with exact arithmetic.

Use an ordinary median when its simpler interpretation is the real
requirement. Use a dedicated inferential procedure when confidence intervals,
a null distribution, a p-value, paired observations, or a two-sample location
shift is required.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction
from itertools import product

_MAX_HODGES_LEHMANN_ITEMS = 512
_MIN_SIGNED_32 = -(1 << 31)
_MAX_SIGNED_32 = (1 << 31) - 1


@dataclass(frozen=True, slots=True)
class HodgesLehmannEstimate:
    lower_walsh_average: Fraction
    upper_walsh_average: Fraction
    pseudomedian: Fraction
    walsh_count: int


def estimate_hodges_lehmann_pseudomedian(
    sample: tuple[int, ...],
) -> HodgesLehmannEstimate:
    """Return the exact one-sample pseudomedian and its central evidence."""
    if type(sample) is not tuple:
        raise TypeError("sample must be an exact tuple")
    if not 1 <= len(sample) <= _MAX_HODGES_LEHMANN_ITEMS:
        raise ValueError("sample size is outside 1..512")
    for index, value in enumerate(sample):
        if type(value) is not int:
            raise TypeError(f"sample[{index}] must be an exact integer")
        if not _MIN_SIGNED_32 <= value <= _MAX_SIGNED_32:
            raise ValueError(f"sample[{index}] is outside signed 32-bit range")

    walsh_sums = sorted(
        sample[left] + sample[right]
        for left in range(len(sample))
        for right in range(left, len(sample))
    )
    count = len(walsh_sums)
    lower_sum = walsh_sums[(count - 1) // 2]
    upper_sum = walsh_sums[count // 2]
    lower_average = Fraction(lower_sum, 2)
    upper_average = Fraction(upper_sum, 2)
    return HodgesLehmannEstimate(
        lower_average,
        upper_average,
        (lower_average + upper_average) / 2,
        count,
    )
```

## Example

```python
def direct_walsh_oracle(sample: tuple[int, ...]) -> HodgesLehmannEstimate:
    averages = sorted(
        Fraction(sample[left] + sample[right], 2)
        for left in range(len(sample))
        for right in range(left, len(sample))
    )
    count = len(averages)
    lower = averages[(count - 1) // 2]
    upper = averages[count // 2]
    return HodgesLehmannEstimate(lower, upper, (lower + upper) / 2, count)


example = estimate_hodges_lehmann_pseudomedian((1, 2, 10))
assert example == HodgesLehmannEstimate(
    lower_walsh_average=Fraction(2),
    upper_walsh_average=Fraction(11, 2),
    pseudomedian=Fraction(15, 4),
    walsh_count=6,
)

checked = 0
for length in range(1, 6):
    for sample in product(range(-2, 3), repeat=length):
        result = estimate_hodges_lehmann_pseudomedian(sample)
        assert result == direct_walsh_oracle(sample)
        assert result == estimate_hodges_lehmann_pseudomedian(tuple(reversed(sample)))

        shifted = estimate_hodges_lehmann_pseudomedian(
            tuple(value + 7 for value in sample)
        )
        assert shifted.pseudomedian == result.pseudomedian + 7

        reflected = estimate_hodges_lehmann_pseudomedian(
            tuple(-value for value in sample)
        )
        assert reflected.pseudomedian == -result.pseudomedian
        checked += 1

assert checked == 3_905 and estimate_hodges_lehmann_pseudomedian((4,)).pseudomedian == 4
```

## Trade-offs and Limitations

For `N` observations, this direct implementation materializes
`N * (N + 1) / 2` integer sums. It takes `O(N² log N)` time and `O(N²)` memory;
the explicit sample cap prevents the quadratic evidence set from growing
silently. Selection algorithms can avoid fully sorting the sums, but are much
more involved.

Exact `Fraction` output avoids binary floating-point rounding, but does not
make the estimator an inference result. The pseudomedian need not equal the
sample median or a population median, and no confidence level, p-value,
distributional assumption, missing-value policy, weighting, streaming update,
or two-sample interpretation is provided.

## Related Snippets

<!-- catalog:related:start -->
- [Compute an Exact Integer Median and Unscaled Median Absolute Deviation](compute-an-exact-integer-median-and-unscaled-median-absolute-deviation.md)
- [Estimate an Exact Theil-Sen Slope from Bounded Integer Points](estimate-an-exact-theil-sen-slope-from-bounded-integer-points.md)
- [Compute an Exact Two-Sided Wilcoxon Signed-Rank Randomization Test for Bounded Integer Pairs](compute-an-exact-two-sided-wilcoxon-signed-rank-randomization-test-for-bounded-integer-pairs.md)
<!-- catalog:related:end -->
