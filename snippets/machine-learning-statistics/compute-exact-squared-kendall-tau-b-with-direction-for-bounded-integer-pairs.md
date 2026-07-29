---
title: "Compute Exact Squared Kendall Tau-b with Direction for Bounded Integer Pairs"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-exact-squared-spearman-rank-correlation-with-direction.md
  - compute-exact-squared-pearson-correlation-with-direction.md
  - ../algorithms-data-structures/count-strict-inversions-in-a-bounded-integer-sequence.md
---

# Compute Exact Squared Kendall Tau-b with Direction for Bounded Integer Pairs

## Idea and Problem

Measure ordinal association from pair concordance while accounting separately for ties on either coordinate.

For every pair of observations, Kendall's tau-b distinguishes concordance,
discordance, a tie only on the first coordinate, a tie only on the second, and
a joint tie. The signed numerator is the number of concordant pairs minus the
number of discordant pairs.

The coefficient can be irrational because its denominator contains a square
root. Returning its exact square together with the numerator's direction keeps
all arithmetic rational without losing the sign.

## When to Use

Use this statistic when two bounded ordinal measurements may contain ties and
their monotone association matters more than a linear relationship. The
returned pair counts are also useful for auditing how ties influence the
coefficient.

Use a statistical library when a p-value, confidence interval, missing-value
policy, weights, sampling design, or asymptotic inference is required. A high
association does not establish causality or agreement in scale.

## Implementation

```python
from collections import Counter
from dataclasses import dataclass
from fractions import Fraction
from typing import Literal

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_OBSERVATION_COUNT = 100_000

type TauDirection = Literal[
    "positive",
    "negative",
    "zero",
    "undefined",
]


@dataclass(frozen=True, slots=True)
class ExactKendallTauBResult:
    concordant_pairs: int
    discordant_pairs: int
    first_only_ties: int
    second_only_ties: int
    joint_ties: int
    squared_tau_b: Fraction | None
    direction: TauDirection


def _tied_pair_count(values: tuple[object, ...] | list[object]) -> int:
    return sum(count * (count - 1) // 2 for count in Counter(values).values())


def exact_squared_kendall_tau_b(
    first: tuple[int, ...],
    second: tuple[int, ...],
) -> ExactKendallTauBResult:
    """Return exact tau-b pair counts, squared magnitude, and direction."""
    for name, values in (("first", first), ("second", second)):
        if type(values) is not tuple:
            raise TypeError(f"{name} must be an exact tuple")
        if not 2 <= len(values) <= _MAX_OBSERVATION_COUNT:
            raise ValueError(f"{name} size is outside the supported range")
        for index, value in enumerate(values):
            if type(value) is not int:
                raise TypeError(f"{name}[{index}] must be an exact integer")
            if not _MIN_INT64 <= value <= _MAX_INT64:
                raise ValueError(f"{name}[{index}] is outside the signed 64-bit range")

    if len(first) != len(second):
        raise ValueError("first and second must have equal lengths")

    paired = sorted(zip(first, second, strict=True))
    first_ties = _tied_pair_count(first)
    second_ties = _tied_pair_count(second)
    joint_ties = _tied_pair_count(paired)

    ordered_second = sorted(set(second))
    second_rank = {value: index + 1 for index, value in enumerate(ordered_second)}
    fenwick = [0] * (len(ordered_second) + 1)

    def prefix_count(index: int) -> int:
        count = 0
        while index > 0:
            count += fenwick[index]
            index -= index & -index
        return count

    def add(index: int) -> None:
        while index < len(fenwick):
            fenwick[index] += 1
            index += index & -index

    discordant_pairs = 0
    prior_count = 0
    group_start = 0
    while group_start < len(paired):
        group_end = group_start + 1
        while group_end < len(paired) and paired[group_end][0] == paired[group_start][0]:
            group_end += 1

        for _, second_value in paired[group_start:group_end]:
            rank = second_rank[second_value]
            discordant_pairs += prior_count - prefix_count(rank)
        for _, second_value in paired[group_start:group_end]:
            add(second_rank[second_value])

        prior_count += group_end - group_start
        group_start = group_end

    total_pairs = len(first) * (len(first) - 1) // 2
    comparable_pairs = total_pairs - first_ties - second_ties + joint_ties
    concordant_pairs = comparable_pairs - discordant_pairs
    first_only_ties = first_ties - joint_ties
    second_only_ties = second_ties - joint_ties

    signed_numerator = concordant_pairs - discordant_pairs
    first_factor = comparable_pairs + first_only_ties
    second_factor = comparable_pairs + second_only_ties
    if first_factor == 0 or second_factor == 0:
        squared_tau_b = None
        direction: TauDirection = "undefined"
    else:
        squared_tau_b = Fraction(
            signed_numerator * signed_numerator,
            first_factor * second_factor,
        )
        if signed_numerator > 0:
            direction = "positive"
        elif signed_numerator < 0:
            direction = "negative"
        else:
            direction = "zero"

    return ExactKendallTauBResult(
        concordant_pairs=concordant_pairs,
        discordant_pairs=discordant_pairs,
        first_only_ties=first_only_ties,
        second_only_ties=second_only_ties,
        joint_ties=joint_ties,
        squared_tau_b=squared_tau_b,
        direction=direction,
    )
```

## Example

```python
one_inversion = exact_squared_kendall_tau_b(
    (1, 2, 3, 4),
    (1, 3, 2, 4),
)
with_ties = exact_squared_kendall_tau_b(
    (1, 1, 2, 2),
    (1, 2, 2, 2),
)
undefined = exact_squared_kendall_tau_b(
    (5, 5, 5),
    (1, 2, 3),
)

assert one_inversion == ExactKendallTauBResult(
    concordant_pairs=5,
    discordant_pairs=1,
    first_only_ties=0,
    second_only_ties=0,
    joint_ties=0,
    squared_tau_b=Fraction(4, 9),
    direction="positive",
)
assert with_ties == ExactKendallTauBResult(
    concordant_pairs=2,
    discordant_pairs=0,
    first_only_ties=1,
    second_only_ties=2,
    joint_ties=1,
    squared_tau_b=Fraction(1, 3),
    direction="positive",
)
assert undefined.squared_tau_b is None
assert undefined.direction == "undefined"
```

## Trade-offs and Limitations

Sorting the paired observations and processing Fenwick-tree operations takes
`O(n log n)` time and `O(n)` auxiliary state. Equal first-coordinate groups
are queried before any member is inserted, so their internal order cannot
create false inversions.

The five counts partition all `n * (n - 1) // 2` observation pairs. Joint ties
are excluded from both denominator factors; first-only and second-only ties
enter their respective factors. If either coordinate has no comparable
variation, the coefficient is undefined even though the pair counts remain
available.

Squaring removes the irrational square root but also removes the sign, which
is why direction is returned separately. The function provides no signed
floating approximation, p-value, confidence interval, weights, missing-value
handling, or protection against biased sampling.

## Related Snippets

<!-- catalog:related:start -->
- [Compute Exact Squared Spearman Rank Correlation with Direction](compute-exact-squared-spearman-rank-correlation-with-direction.md)
- [Compute Exact Squared Pearson Correlation with Direction](compute-exact-squared-pearson-correlation-with-direction.md)
- [Count Strict Inversions in a Bounded Integer Sequence](../algorithms-data-structures/count-strict-inversions-in-a-bounded-integer-sequence.md)
<!-- catalog:related:end -->
