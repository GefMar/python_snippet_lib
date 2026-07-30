---
title: "Compute an Exact Two-Sided Mann-Whitney U Permutation Test for Bounded Integer Samples"
snippet_type: algorithm
use_cases:
  - data-transformation
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-an-exact-two-sample-permutation-test-for-a-mean-difference.md
  - compute-an-exact-two-sample-kolmogorov-smirnov-distance-and-witness-for-bounded-integer-samples.md
  - compute-an-exact-paired-two-sided-sign-test.md
---

# Compute an Exact Two-Sided Mann-Whitney U Permutation Test for Bounded Integer Samples

## Idea and Problem

Compute an exact pairwise-order statistic and a fully enumerated two-sided permutation probability for two small integer samples, including ties.

Each left-greater pair contributes two to doubled U, a tied pair contributes
one, and a left-smaller pair contributes zero. Keeping the statistic doubled
is equivalent to using midranks without introducing floats. The null
distribution reassigns labelled pooled positions to every left group of the
same size and counts statistics at least as far from the shared center as the
observed statistic.

## When to Use

Use this exact test for two small independent samples when group labels are
exchangeable under the null hypothesis and relative ordering is the intended
signal. It is useful when ties are common and an exhaustive conditional result
is preferable to an asymptotic normal approximation.

The precise two-sided definition is part of the contract; statistical
libraries can use different exact-tail or tie conventions. Use a reviewed
statistics package for larger samples, one-sided alternatives, asymptotic
approximations, confidence intervals, weighted observations, paired data, or
more than two groups.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction
from itertools import combinations
from math import comb

_MIN_INT64 = -(2**63)
_MAX_INT64 = 2**63 - 1
_MAX_SAMPLE_SIZE = 64
_MAX_TOTAL_SIZE = 64
_MAX_ALLOCATIONS = 50_000


@dataclass(frozen=True, slots=True)
class ExactMannWhitneyTest:
    left_u_twice: int
    right_u_twice: int
    left_u: Fraction
    right_u: Fraction
    qualifying_allocations: int
    total_allocations: int
    two_sided_p: Fraction


def _validated_sample(value: object, *, field: str) -> tuple[int, ...]:
    if type(value) is not tuple:
        raise TypeError(f"{field} must be an exact tuple")
    if not 1 <= len(value) <= _MAX_SAMPLE_SIZE:
        raise ValueError(f"{field} size is outside 1..64")
    for index, observation in enumerate(value):
        if type(observation) is not int:
            raise TypeError(f"{field}[{index}] must be an exact integer")
        if not _MIN_INT64 <= observation <= _MAX_INT64:
            raise ValueError(f"{field}[{index}] is outside the signed 64-bit range")
    return value


def _pairwise_u_twice(
    left: tuple[int, ...],
    right: tuple[int, ...],
) -> int:
    return sum(
        2 if left_value > right_value else left_value == right_value
        for left_value in left
        for right_value in right
    )


def exact_mann_whitney_test(
    left: tuple[int, ...],
    right: tuple[int, ...],
) -> ExactMannWhitneyTest:
    checked_left = _validated_sample(left, field="left")
    checked_right = _validated_sample(right, field="right")
    total_size = len(checked_left) + len(checked_right)
    if total_size > _MAX_TOTAL_SIZE:
        raise ValueError("combined sample size exceeds 64")

    allocation_count = comb(total_size, len(checked_left))
    if allocation_count > _MAX_ALLOCATIONS:
        raise ValueError("exact label-allocation count exceeds 50000")

    observed_u_twice = _pairwise_u_twice(checked_left, checked_right)
    pair_count = len(checked_left) * len(checked_right)
    observed_distance = abs(observed_u_twice - pair_count)
    pooled = (*checked_left, *checked_right)
    all_indexes = range(total_size)
    qualifying = 0

    for left_indexes_tuple in combinations(all_indexes, len(checked_left)):
        left_indexes = frozenset(left_indexes_tuple)
        candidate_left = tuple(
            pooled[index]
            for index in all_indexes
            if index in left_indexes
        )
        candidate_right = tuple(
            pooled[index]
            for index in all_indexes
            if index not in left_indexes
        )
        candidate_u_twice = _pairwise_u_twice(candidate_left, candidate_right)
        if abs(candidate_u_twice - pair_count) >= observed_distance:
            qualifying += 1

    right_u_twice = 2 * pair_count - observed_u_twice
    return ExactMannWhitneyTest(
        left_u_twice=observed_u_twice,
        right_u_twice=right_u_twice,
        left_u=Fraction(observed_u_twice, 2),
        right_u=Fraction(right_u_twice, 2),
        qualifying_allocations=qualifying,
        total_allocations=allocation_count,
        two_sided_p=Fraction(qualifying, allocation_count),
    )
```

## Example

```python
def rank_u_twice(
    left: tuple[int, ...],
    right: tuple[int, ...],
) -> int:
    pooled = sorted((*left, *right))
    doubled_midranks: dict[int, int] = {}
    start = 0
    while start < len(pooled):
        end = start + 1
        while end < len(pooled) and pooled[end] == pooled[start]:
            end += 1
        doubled_midranks[pooled[start]] = (start + 1) + end
        start = end
    doubled_rank_sum = sum(doubled_midranks[value] for value in left)
    return doubled_rank_sum - len(left) * (len(left) + 1)


def independent_test(
    left: tuple[int, ...],
    right: tuple[int, ...],
) -> tuple[int, int, Fraction]:
    pooled = (*left, *right)
    pair_count = len(left) * len(right)
    observed = rank_u_twice(left, right)
    distance = abs(observed - pair_count)
    qualifying = 0
    total = comb(len(pooled), len(left))
    for indexes in combinations(range(len(pooled)), len(left)):
        selected = frozenset(indexes)
        candidate_left = tuple(pooled[index] for index in indexes)
        candidate_right = tuple(
            pooled[index]
            for index in range(len(pooled))
            if index not in selected
        )
        if abs(rank_u_twice(candidate_left, candidate_right) - pair_count) >= distance:
            qualifying += 1
    return observed, 2 * pair_count - observed, Fraction(qualifying, total)


def check_tiny_samples() -> int:
    from itertools import product

    values = (-1, 0, 1)
    checked = 0
    for left_size in range(1, 4):
        for right_size in range(1, 4):
            for left in product(values, repeat=left_size):
                for right in product(values, repeat=right_size):
                    observed = exact_mann_whitney_test(left, right)
                    expected_left, expected_right, expected_p = independent_test(
                        left,
                        right,
                    )
                    assert (
                        observed.left_u_twice,
                        observed.right_u_twice,
                        observed.two_sided_p,
                    ) == (expected_left, expected_right, expected_p)
                    swapped = exact_mann_whitney_test(right, left)
                    assert swapped.left_u_twice == observed.right_u_twice
                    assert swapped.two_sided_p == observed.two_sided_p
                    checked += 1
    return checked


checked = check_tiny_samples()

all_ties = exact_mann_whitney_test((2, 2), (2, 2, 2))
boundary = exact_mann_whitney_test(tuple(range(9)), tuple(range(9, 18)))
try:
    exact_mann_whitney_test(tuple(range(9)), tuple(range(9, 19)))
except ValueError:
    allocation_cap_enforced = True
else:
    allocation_cap_enforced = False

assert (
    checked == 1_521
    and all_ties.left_u == Fraction(3, 1)
    and all_ties.two_sided_p == Fraction(1, 1)
    and boundary.total_allocations == 48_620
    and allocation_cap_enforced
)
```

## Trade-offs and Limitations

Computing one U statistic takes `O(LR)` time. Exact testing takes
`O(comb(L + R, L) * L * R)` time and `O(L + R)` transient memory; the explicit
allocation cap controls that combinatorial cost rather than relying only on a
sample-size limit. Pooled positions remain distinct even when values tie.

This is a conditional permutation test under label exchangeability. Its
inclusive absolute-distance tail is fully specified but is not claimed to
match every package's option named "exact" or "two-sided". The result contains
no effect-size interval, normal approximation, continuity adjustment, or
multiple-testing correction.

## Related Snippets

<!-- catalog:related:start -->
- [Compute an Exact Two-Sample Permutation Test for a Mean Difference](compute-an-exact-two-sample-permutation-test-for-a-mean-difference.md)
- [Compute an Exact Two-Sample Kolmogorov-Smirnov Distance and Witness for Bounded Integer Samples](compute-an-exact-two-sample-kolmogorov-smirnov-distance-and-witness-for-bounded-integer-samples.md)
- [Compute an Exact Paired Two-Sided Sign Test](compute-an-exact-paired-two-sided-sign-test.md)
<!-- catalog:related:end -->
