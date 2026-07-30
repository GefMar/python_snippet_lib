---
title: "Compute an Exact Two-Sided Wilcoxon Signed-Rank Randomization Test for Bounded Integer Pairs"
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
  - compute-an-exact-two-sided-mann-whitney-u-permutation-test-for-bounded-integer-samples.md
  - compute-an-exact-two-sample-permutation-test-for-a-mean-difference.md
---

# Compute an Exact Two-Sided Wilcoxon Signed-Rank Randomization Test for Bounded Integer Pairs

## Idea and Problem

Test paired integer changes with exact magnitude ranks and one fully specified two-sided sign-randomization probability.

Each difference is `after - before`. Zero differences are discarded, tied
absolute differences receive exact midranks, and doubled ranks avoid floats.
The null distribution keeps those magnitudes fixed while enumerating every
labelled reassignment of positive and negative signs.

## When to Use

Use this test for a small paired sample when both the direction and ordinal
magnitude of each nonzero change are meaningful and the difference
distribution can reasonably be treated as symmetric about zero under the null.
It is useful when ties are present and an exhaustive conditional calculation
is preferable to a normal approximation.

Use the paired sign test when only direction is trustworthy. Use a reviewed
statistics package for larger samples, Pratt or split-zero rules, one-sided
alternatives, effect intervals, clustered pairs, or multiplicity correction.
The exact two-sided definition below is part of this function's contract and
is not a promise to reproduce every package option named `exact`.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction

_MIN_INT64 = -(2**63)
_MAX_INT64 = 2**63 - 1
_MAX_PAIRS = 64
_MAX_SIGN_ASSIGNMENTS = 65_536


@dataclass(frozen=True, slots=True)
class ExactSignedRankTest:
    positive_rank_twice: int
    negative_rank_twice: int
    positive_rank: Fraction
    negative_rank: Fraction
    zero_pairs: int
    effective_pairs: int
    qualifying_assignments: int
    total_assignments: int
    two_sided_p: Fraction


def _validated_sample(value: object, *, field: str) -> tuple[int, ...]:
    if type(value) is not tuple:
        raise TypeError(f"{field} must be an exact tuple")
    if not 1 <= len(value) <= _MAX_PAIRS:
        raise ValueError(f"{field} size is outside 1..64")
    for index, observation in enumerate(value):
        if type(observation) is not int:
            raise TypeError(f"{field}[{index}] must be an exact integer")
        if not _MIN_INT64 <= observation <= _MAX_INT64:
            raise ValueError(f"{field}[{index}] is outside the signed 64-bit range")
    return value


def _doubled_midranks(absolute_differences: tuple[int, ...]) -> tuple[int, ...]:
    order = sorted(
        range(len(absolute_differences)),
        key=lambda index: (absolute_differences[index], index),
    )
    ranks_twice = [0] * len(order)
    start = 0
    while start < len(order):
        end = start + 1
        while (
            end < len(order)
            and absolute_differences[order[end]]
            == absolute_differences[order[start]]
        ):
            end += 1
        midrank_twice = (start + 1) + end
        for ordered_index in range(start, end):
            ranks_twice[order[ordered_index]] = midrank_twice
        start = end
    return tuple(ranks_twice)


def exact_signed_rank_test(
    before: tuple[int, ...],
    after: tuple[int, ...],
) -> ExactSignedRankTest:
    checked_before = _validated_sample(before, field="before")
    checked_after = _validated_sample(after, field="after")
    if len(checked_before) != len(checked_after):
        raise ValueError("before and after must have equal lengths")

    differences = tuple(
        after_value - before_value
        for before_value, after_value in zip(
            checked_before,
            checked_after,
            strict=True,
        )
        if after_value != before_value
    )
    effective_pairs = len(differences)
    total_assignments = 1 << effective_pairs
    if total_assignments > _MAX_SIGN_ASSIGNMENTS:
        raise ValueError("exact sign-assignment count exceeds 65536")

    ranks_twice = _doubled_midranks(tuple(abs(value) for value in differences))
    positive_rank_twice = sum(
        rank
        for difference, rank in zip(differences, ranks_twice, strict=True)
        if difference > 0
    )
    total_rank_twice = sum(ranks_twice)
    negative_rank_twice = total_rank_twice - positive_rank_twice
    observed_distance = abs(2 * positive_rank_twice - total_rank_twice)

    qualifying = 0
    for sign_mask in range(total_assignments):
        candidate_positive = sum(
            rank
            for index, rank in enumerate(ranks_twice)
            if sign_mask & (1 << index)
        )
        if abs(2 * candidate_positive - total_rank_twice) >= observed_distance:
            qualifying += 1

    return ExactSignedRankTest(
        positive_rank_twice=positive_rank_twice,
        negative_rank_twice=negative_rank_twice,
        positive_rank=Fraction(positive_rank_twice, 2),
        negative_rank=Fraction(negative_rank_twice, 2),
        zero_pairs=len(checked_before) - effective_pairs,
        effective_pairs=effective_pairs,
        qualifying_assignments=qualifying,
        total_assignments=total_assignments,
        two_sided_p=Fraction(qualifying, total_assignments),
    )
```

## Example

```python
def independent_result(differences: tuple[int, ...]) -> tuple[int, int, Fraction]:
    from itertools import product

    nonzero = tuple(value for value in differences if value)
    ordered = sorted(abs(value) for value in nonzero)
    ranks_twice = []
    for difference in nonzero:
        positions = tuple(
            index + 1
            for index, value in enumerate(ordered)
            if value == abs(difference)
        )
        ranks_twice.append(2 * sum(positions) // len(positions))

    positive = sum(
        rank
        for difference, rank in zip(nonzero, ranks_twice, strict=True)
        if difference > 0
    )
    total_rank = sum(ranks_twice)
    distance = abs(2 * positive - total_rank)
    qualifying = sum(
        abs(
            2
            * sum(
                rank
                for sign, rank in zip(signs, ranks_twice, strict=True)
                if sign > 0
            )
            - total_rank
        )
        >= distance
        for signs in product((-1, 1), repeat=len(nonzero))
    )
    assignments = 1 << len(nonzero)
    return positive, total_rank - positive, Fraction(qualifying, assignments)


def check_tiny_pairs() -> int:
    from itertools import product

    checked = 0
    for pair_count in range(1, 6):
        before = (0,) * pair_count
        for differences in product(range(-2, 3), repeat=pair_count):
            observed = exact_signed_rank_test(before, differences)
            expected_positive, expected_negative, expected_p = independent_result(
                differences
            )
            assert (
                observed.positive_rank_twice,
                observed.negative_rank_twice,
                observed.two_sided_p,
            ) == (expected_positive, expected_negative, expected_p)
            checked += 1
    return checked


checked = check_tiny_pairs()
sample = exact_signed_rank_test((0, 0, 0, 0), (1, 2, 3, 0))
swapped = exact_signed_rank_test((1, 2, 3, 0), (0, 0, 0, 0))
all_zero = exact_signed_rank_test((4, 5), (4, 5))
boundary = exact_signed_rank_test((0,) * 16, tuple(range(1, 17)))
try:
    exact_signed_rank_test((0,) * 17, tuple(range(1, 18)))
except ValueError:
    allocation_cap_enforced = True
else:
    allocation_cap_enforced = False

assert (
    checked == 3_905
    and sample.positive_rank_twice == 12
    and sample.negative_rank_twice == 0
    and sample.two_sided_p == Fraction(1, 4)
    and swapped.positive_rank_twice == sample.negative_rank_twice
    and swapped.two_sided_p == sample.two_sided_p
    and all_zero.two_sided_p == Fraction(1, 1)
    and boundary.total_assignments == 65_536
    and allocation_cap_enforced
)
```

## Trade-offs and Limitations

Validation and ranking take `O(P + N log N)` time for `P` pairs and `N`
nonzero differences. Exhaustive testing takes `O(N * 2**N)` integer work and
`O(N)` memory; the explicit assignment cap bounds that cost. Labelled pairs
remain distinct even when their absolute differences tie.

Discarding zeros is one declared convention, not a universally correct choice.
Midranks preserve ties exactly, but the sign-randomization model still assumes
independent paired differences whose null distribution is symmetric. The
inclusive absolute-distance tail can be conservative and contains no effect
estimate, interval, alpha decision, or multiple-testing adjustment.

## Related Snippets

<!-- catalog:related:start -->
- [Compute an Exact Paired Two-Sided Sign Test](compute-an-exact-paired-two-sided-sign-test.md)
- [Compute an Exact Two-Sided Mann-Whitney U Permutation Test for Bounded Integer Samples](compute-an-exact-two-sided-mann-whitney-u-permutation-test-for-bounded-integer-samples.md)
- [Compute an Exact Two-Sample Permutation Test for a Mean Difference](compute-an-exact-two-sample-permutation-test-for-a-mean-difference.md)
<!-- catalog:related:end -->
