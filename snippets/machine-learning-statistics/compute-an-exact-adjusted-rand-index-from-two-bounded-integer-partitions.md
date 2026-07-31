---
title: "Compute an Exact Adjusted Rand Index from Two Bounded Integer Partitions"
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
  - compute-exact-unweighted-cohens-kappa-from-a-confusion-matrix.md
  - compute-mutual-information-from-a-bounded-integer-contingency-table.md
  - compute-exact-pearson-chi-square-and-cramers-v-squared-from-a-bounded-contingency-table.md
---

# Compute an Exact Adjusted Rand Index from Two Bounded Integer Partitions

## Idea and Problem

Compare two partitions of the same items through pair co-membership while correcting the observed agreement for chance.

For each partition, count how many unordered item pairs share a label. A
contingency table between the two labelings also reveals how many pairs share a
label in both. The adjusted Rand index compares that joint count with its
chance expectation while treating label values only as opaque cluster names.

All combinatorial counts are integers, so the expectation, adjustment and
final index can remain exact `Fraction` values. Renaming clusters or applying
the same item permutation to both inputs cannot change the result.

## When to Use

Use this calculation to compare two hard clusterings, segmentations, grouping
rules, or reference partitions of the same bounded item set. It is especially
useful in reference tests where floating-point rounding or arbitrary numeric
cluster identifiers should not affect the evidence.

Use a specialized evaluation library for weighted items, fuzzy or overlapping
membership, sampling uncertainty, confidence intervals, or large sparse
contingency structures. Use Cohen's kappa when the numeric labels identify the
same named rating categories rather than interchangeable cluster names.

## Implementation

```python
from collections import Counter
from dataclasses import dataclass
from fractions import Fraction

_MIN_SIGNED_64 = -(1 << 63)
_MAX_SIGNED_64 = (1 << 63) - 1
_MAX_PARTITION_SIZE = 4_096


@dataclass(frozen=True, slots=True)
class AdjustedRandEvidence:
    sample_size: int
    first_cluster_count: int
    second_cluster_count: int
    total_pair_count: int
    same_in_first: int
    same_in_second: int
    same_in_both: int
    expected_same_in_both: Fraction
    maximum_same_in_both: Fraction
    adjusted_rand_index: Fraction


def _validated_partition(labels: object, *, field: str) -> tuple[int, ...]:
    if type(labels) is not tuple:
        raise TypeError(f"{field} must be an exact tuple")
    if not 1 <= len(labels) <= _MAX_PARTITION_SIZE:
        raise ValueError(f"{field} length is outside the supported range")

    for index, label in enumerate(labels):
        if type(label) is not int:
            raise TypeError(f"{field}[{index}] must be an exact integer")
        if not _MIN_SIGNED_64 <= label <= _MAX_SIGNED_64:
            raise ValueError(f"{field}[{index}] is outside the signed 64-bit range")
    return labels


def _pair_count(item_count: int) -> int:
    return item_count * (item_count - 1) // 2


def exact_adjusted_rand_index(
    first_labels: tuple[int, ...],
    second_labels: tuple[int, ...],
) -> AdjustedRandEvidence:
    """Return exact pair-count evidence and the adjusted Rand index."""
    first = _validated_partition(first_labels, field="first_labels")
    second = _validated_partition(second_labels, field="second_labels")
    if len(first) != len(second):
        raise ValueError("partitions must contain the same number of items")

    first_counts = Counter(first)
    second_counts = Counter(second)
    joint_counts = Counter(zip(first, second, strict=True))

    sample_size = len(first)
    total_pair_count = _pair_count(sample_size)
    same_in_first = sum(_pair_count(count) for count in first_counts.values())
    same_in_second = sum(_pair_count(count) for count in second_counts.values())
    same_in_both = sum(_pair_count(count) for count in joint_counts.values())

    if total_pair_count == 0:
        expected = Fraction()
        maximum = Fraction()
        adjusted = Fraction(1)
    else:
        expected = Fraction(same_in_first * same_in_second, total_pair_count)
        maximum = Fraction(same_in_first + same_in_second, 2)
        denominator = maximum - expected
        adjusted = (
            Fraction(1) if denominator == 0 else (Fraction(same_in_both) - expected) / denominator
        )

    return AdjustedRandEvidence(
        sample_size=sample_size,
        first_cluster_count=len(first_counts),
        second_cluster_count=len(second_counts),
        total_pair_count=total_pair_count,
        same_in_first=same_in_first,
        same_in_second=same_in_second,
        same_in_both=same_in_both,
        expected_same_in_both=expected,
        maximum_same_in_both=maximum,
        adjusted_rand_index=adjusted,
    )
```

## Example

```python
def direct_pair_oracle(
    first: tuple[int, ...],
    second: tuple[int, ...],
) -> tuple[int, int, int, Fraction]:
    same_in_first = 0
    same_in_second = 0
    same_in_both = 0
    for left in range(len(first)):
        for right in range(left + 1, len(first)):
            first_equal = first[left] == first[right]
            second_equal = second[left] == second[right]
            same_in_first += first_equal
            same_in_second += second_equal
            same_in_both += first_equal and second_equal

    total_pairs = _pair_count(len(first))
    if total_pairs == 0:
        adjusted = Fraction(1)
    else:
        expected = Fraction(same_in_first * same_in_second, total_pairs)
        maximum = Fraction(same_in_first + same_in_second, 2)
        adjusted = (
            Fraction(1)
            if maximum == expected
            else (Fraction(same_in_both) - expected) / (maximum - expected)
        )
    return same_in_first, same_in_second, same_in_both, adjusted


def verify_small_labelings() -> int:
    from itertools import product

    checked = 0
    for size in range(1, 5):
        labelings = tuple(product(range(3), repeat=size))
        for first in labelings:
            for second in labelings:
                evidence = exact_adjusted_rand_index(first, second)
                oracle = direct_pair_oracle(first, second)
                assert (
                    evidence.same_in_first,
                    evidence.same_in_second,
                    evidence.same_in_both,
                    evidence.adjusted_rand_index,
                ) == oracle
                checked += 1
    return checked


identical = exact_adjusted_rand_index((10, 10, 20, 20), (7, 7, 9, 9))
crossed = exact_adjusted_rand_index((0, 0, 1, 1), (0, 1, 0, 1))
singletons = exact_adjusted_rand_index((1, 2, 3), (8, 7, 6))
one_cluster = exact_adjusted_rand_index((1, 1, 1), (9, 9, 9))
permuted = exact_adjusted_rand_index((20, 10, 20, 10), (9, 7, 9, 7))


def rejected(first: object, second: object) -> bool:
    try:
        exact_adjusted_rand_index(first, second)
    except (TypeError, ValueError):
        return True
    return False


invalid_calls = (
    ((), ()),
    ((0, 1), (0,)),
    ([0, 1], (0, 1)),
    ((True, 1), (0, 1)),
    (((1 << 63), 0), (0, 1)),
)

assert (
    identical.adjusted_rand_index,
    crossed.adjusted_rand_index,
    singletons.adjusted_rand_index,
    one_cluster.adjusted_rand_index,
    permuted.adjusted_rand_index,
    verify_small_labelings(),
    sum(rejected(first, second) for first, second in invalid_calls),
) == (Fraction(1), Fraction(-1, 2), Fraction(1), Fraction(1), Fraction(1), 7_380, 5)
```

## Trade-offs and Limitations

For `n` items, the counters take expected `O(n)` time and store at most `O(n)`
distinct marginal and joint keys. Python integer arithmetic keeps pair counts
and rational intermediates exact. The explicit 4,096-item cap bounds memory
while remaining large enough for reference comparisons.

The result depends only on pair co-membership. Cluster identifiers can be
renamed freely, and the same item permutation can be applied to both inputs.
When both partitions are all singletons or both place every item in one
cluster, the chance-adjustment denominator is zero; the documented convention
returns one for these structurally identical trivial partitions. A one-item
comparison follows the same convention.

This is a hard-partition agreement measure, not classification accuracy. It
does not align semantic class names, accept fuzzy or overlapping membership,
weight items, handle missing labels, calculate uncertainty, produce a p-value,
or establish that either partition is correct.

## Related Snippets

<!-- catalog:related:start -->
- [Compute Exact Unweighted Cohen's Kappa from a Confusion Matrix](compute-exact-unweighted-cohens-kappa-from-a-confusion-matrix.md)
- [Compute Mutual Information from a Bounded Integer Contingency Table](compute-mutual-information-from-a-bounded-integer-contingency-table.md)
- [Compute Exact Pearson Chi-Square and Cramer's V Squared from a Bounded Contingency Table](compute-exact-pearson-chi-square-and-cramers-v-squared-from-a-bounded-contingency-table.md)
<!-- catalog:related:end -->
