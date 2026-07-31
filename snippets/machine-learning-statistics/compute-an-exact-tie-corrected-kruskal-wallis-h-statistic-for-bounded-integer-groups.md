---
title: "Compute an Exact Tie-Corrected Kruskal-Wallis H Statistic for Bounded Integer Groups"
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
  - compute-an-exact-two-sided-mann-whitney-u-permutation-test-for-bounded-integer-samples.md
  - compute-an-exact-friedman-rank-randomization-test-for-bounded-integer-blocks.md
  - compute-exact-squared-spearman-rank-correlation-with-direction.md
---

# Compute an Exact Tie-Corrected Kruskal-Wallis H Statistic for Bounded Integer Groups

## Idea and Problem

Summarize how strongly several independent integer groups differ in pooled rank location while retaining exact tie-corrected arithmetic.

Kruskal-Wallis replaces observations with global midranks, totals those ranks
inside each group, and compares the totals with what equal rank location would
produce. Doubled midranks keep rank sums integral even when a tie block has a
half-integer midpoint. `Fraction` then preserves the raw H statistic, the tie
factor, and their quotient without rounding.

The statistic becomes undefined after tie correction when every pooled value
is equal. Returning `corrected_h=None` distinguishes that case from evidence
whose corrected statistic is numerically zero.

## When to Use

Use this calculation when three or more small independent groups contain
ordinally comparable integer observations and exact reproducible rank evidence
is useful in tests, reports, or a separately governed analysis. The returned
doubled rank sums expose enough intermediate state to audit the calculation.

This page computes a statistic only. Choose and validate an appropriate null
distribution, p-value method, multiple-testing policy, and effect-size measure
outside this helper. Use Friedman-style methods for blocked or repeated
measurements instead of treating dependent groups as independent.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction
from random import Random

_MIN_KRUSKAL_GROUPS = 3
_MAX_KRUSKAL_GROUPS = 16
_MAX_KRUSKAL_GROUP_SIZE = 64
_MAX_KRUSKAL_OBSERVATIONS = 256
_MIN_SIGNED_64 = -(2**63)
_MAX_SIGNED_64 = 2**63 - 1


@dataclass(frozen=True, slots=True)
class KruskalWallisEvidence:
    group_sizes: tuple[int, ...]
    doubled_rank_sums: tuple[int, ...]
    raw_h: Fraction
    tie_correction: Fraction
    corrected_h: Fraction | None


def exact_kruskal_wallis_h(
    groups: tuple[tuple[int, ...], ...],
) -> KruskalWallisEvidence:
    """Return exact pooled-rank H evidence without performing inference."""
    if type(groups) is not tuple:
        raise TypeError("groups must be an exact tuple")
    if not _MIN_KRUSKAL_GROUPS <= len(groups) <= _MAX_KRUSKAL_GROUPS:
        raise ValueError("group count is outside 3..16")

    pooled: list[tuple[int, int]] = []
    group_sizes: list[int] = []
    for group_index, group in enumerate(groups):
        if type(group) is not tuple:
            raise TypeError(f"groups[{group_index}] must be an exact tuple")
        if not 1 <= len(group) <= _MAX_KRUSKAL_GROUP_SIZE:
            raise ValueError("each group must contain 1..64 observations")
        group_sizes.append(len(group))
        for value in group:
            if type(value) is not int:
                raise TypeError("observations must be exact integers")
            if not _MIN_SIGNED_64 <= value <= _MAX_SIGNED_64:
                raise ValueError("observation is outside the signed 64-bit range")
            pooled.append((value, group_index))
    if len(pooled) > _MAX_KRUSKAL_OBSERVATIONS:
        raise ValueError("pooled observation count exceeds 256")

    pooled.sort(key=lambda item: item[0])
    doubled_rank_sums = [0] * len(groups)
    tie_sum = 0
    block_start = 0
    while block_start < len(pooled):
        block_end = block_start + 1
        while block_end < len(pooled) and pooled[block_end][0] == pooled[block_start][0]:
            block_end += 1
        doubled_midrank = block_start + block_end + 1
        for _, group_index in pooled[block_start:block_end]:
            doubled_rank_sums[group_index] += doubled_midrank
        tie_size = block_end - block_start
        tie_sum += tie_size**3 - tie_size
        block_start = block_end

    observation_count = len(pooled)
    weighted_rank_sum = sum(
        Fraction(rank_sum**2, group_size)
        for rank_sum, group_size in zip(doubled_rank_sums, group_sizes, strict=True)
    )
    raw_h = Fraction(3, observation_count * (observation_count + 1)) * weighted_rank_sum - 3 * (
        observation_count + 1
    )
    tie_denominator = observation_count**3 - observation_count
    tie_correction = Fraction(tie_denominator - tie_sum, tie_denominator)
    corrected_h = None if tie_correction == 0 else raw_h / tie_correction
    return KruskalWallisEvidence(
        group_sizes=tuple(group_sizes),
        doubled_rank_sums=tuple(doubled_rank_sums),
        raw_h=raw_h,
        tie_correction=tie_correction,
        corrected_h=corrected_h,
    )
```

## Example

```python
def count_based_oracle(
    groups: tuple[tuple[int, ...], ...],
) -> KruskalWallisEvidence:
    values = tuple(value for group in groups for value in group)
    doubled_rank_sums = tuple(
        sum(
            2 * sum(candidate < value for candidate in values)
            + sum(candidate == value for candidate in values)
            + 1
            for value in group
        )
        for group in groups
    )
    total = len(values)
    raw_h = Fraction(3, total * (total + 1)) * sum(
        (
            Fraction(rank_sum**2, len(group))
            for rank_sum, group in zip(doubled_rank_sums, groups, strict=True)
        ),
        Fraction(),
    ) - 3 * (total + 1)
    tie_sum = sum(count**3 - count for value in set(values) if (count := values.count(value)))
    denominator = total**3 - total
    correction = Fraction(denominator - tie_sum, denominator)
    return KruskalWallisEvidence(
        group_sizes=tuple(map(len, groups)),
        doubled_rank_sums=doubled_rank_sums,
        raw_h=raw_h,
        tie_correction=correction,
        corrected_h=None if correction == 0 else raw_h / correction,
    )


separated = ((1, 2), (3, 4), (5, 6))
assert exact_kruskal_wallis_h(separated) == count_based_oracle(separated)
assert exact_kruskal_wallis_h(separated).corrected_h == Fraction(32, 7)

all_tied = exact_kruskal_wallis_h(((4,), (4, 4), (4,)))
assert all_tied.tie_correction == 0
assert all_tied.corrected_h is None

rng = Random(0x6B_57)
checked = 0
for _ in range(5_000):
    groups = tuple(
        tuple(rng.randrange(-4, 5) for _ in range(rng.randrange(1, 7)))
        for _ in range(rng.randrange(3, 7))
    )
    assert exact_kruskal_wallis_h(groups) == count_based_oracle(groups)
    checked += 1

assert checked == 5_000
```

## Trade-offs and Limitations

Sorting `N` pooled observations takes `O(N log N)` time; tie scanning and rank
aggregation are linear, with `O(N + K)` state for `K` groups. Exact fractions
make boundary cases reviewable but can cost more than floating-point arithmetic
for repeated calculations.

The result is not a p-value, an exact test, or a significance decision. It
does not choose between asymptotic and permutation null distributions, correct
for multiple comparisons, quantify effect size, or justify the independence
and comparability assumptions. `corrected_h=None` means the tie factor is zero,
not that the groups have been shown equivalent.

## Related Snippets

<!-- catalog:related:start -->
- [Compute an Exact Two-Sided Mann-Whitney U Permutation Test for Bounded Integer Samples](compute-an-exact-two-sided-mann-whitney-u-permutation-test-for-bounded-integer-samples.md)
- [Compute an Exact Friedman Rank-Randomization Test for Bounded Integer Blocks](compute-an-exact-friedman-rank-randomization-test-for-bounded-integer-blocks.md)
- [Compute Exact Squared Spearman Rank Correlation with Direction](compute-exact-squared-spearman-rank-correlation-with-direction.md)
<!-- catalog:related:end -->
