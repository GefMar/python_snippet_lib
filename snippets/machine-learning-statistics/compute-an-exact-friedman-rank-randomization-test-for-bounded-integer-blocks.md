---
title: "Compute an Exact Friedman Rank-Randomization Test for Bounded Integer Blocks"
snippet_type: algorithm
use_cases:
  - data-transformation
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-an-exact-two-sided-wilcoxon-signed-rank-randomization-test-for-bounded-integer-pairs.md
  - compute-an-exact-two-sided-mann-whitney-u-permutation-test-for-bounded-integer-samples.md
  - compute-an-exact-two-sample-permutation-test-for-a-mean-difference.md
---

# Compute an Exact Friedman Rank-Randomization Test for Bounded Integer Blocks

## Idea and Problem

Compare three to five treatments observed in the same small set of complete blocks by exhaustively permuting treatment labels within every block.

Values are ranked independently inside each block. Doubled midranks represent
ties without floats, and the squared dispersion of treatment rank sums provides
an integer statistic. The inclusive upper tail counts every labelled
permutation, including distinct label assignments that look identical because
some observed values tie.

## When to Use

Use this exact Friedman randomization test for a small complete blocked design
when observations are at least ordinal, blocks are independent, and treatment
labels are exchangeable within each block under the null. Exhaustive counting
is attractive when ties make an asymptotic approximation questionable and the
declared permutation cap is affordable.

Use a reviewed statistics package for larger designs, missing observations,
weights, covariates, post-hoc comparisons, effect estimates, or multiplicity
adjustment. This function reports one explicitly defined conditional exact
upper tail; it does not promise equivalence to every package option named
`exact` or `friedman`.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction
from itertools import permutations, product
from math import factorial

_MIN_INT64 = -(2**63)
_MAX_INT64 = 2**63 - 1
_MAX_BLOCKS = 64
_MAX_CELLS = 512
_MAX_ASSIGNMENTS = 65_536


@dataclass(frozen=True, slots=True)
class ExactFriedmanTest:
    doubled_rank_sums: tuple[int, ...]
    rank_dispersion: int
    qualifying_assignments: int
    total_assignments: int
    upper_tail_p: Fraction


def _doubled_midranks(row: tuple[int, ...]) -> tuple[int, ...]:
    order = sorted(range(len(row)), key=lambda index: (row[index], index))
    ranks = [0] * len(row)
    start = 0
    while start < len(order):
        end = start + 1
        while end < len(order) and row[order[end]] == row[order[start]]:
            end += 1
        doubled_midrank = (start + 1) + end
        for position in range(start, end):
            ranks[order[position]] = doubled_midrank
        start = end
    return tuple(ranks)


def exact_friedman_test(
    blocks: tuple[tuple[int, ...], ...],
) -> ExactFriedmanTest:
    if type(blocks) is not tuple:
        raise TypeError("blocks must be an exact tuple")
    if not 2 <= len(blocks) <= _MAX_BLOCKS:
        raise ValueError("block count is outside 2..64")
    if type(blocks[0]) is not tuple:
        raise TypeError("blocks[0] must be an exact tuple")
    treatment_count = len(blocks[0])
    if not 3 <= treatment_count <= 5:
        raise ValueError("treatment count is outside 3..5")
    if len(blocks) * treatment_count > _MAX_CELLS:
        raise ValueError("matrix contains more than 512 cells")

    checked_blocks: list[tuple[int, ...]] = []
    for block_index, block in enumerate(blocks):
        if type(block) is not tuple:
            raise TypeError(f"blocks[{block_index}] must be an exact tuple")
        if len(block) != treatment_count:
            raise ValueError("blocks must form a complete rectangular matrix")
        for treatment_index, value in enumerate(block):
            if type(value) is not int:
                raise TypeError(
                    f"blocks[{block_index}][{treatment_index}] must be an exact integer"
                )
            if not _MIN_INT64 <= value <= _MAX_INT64:
                raise ValueError(
                    f"blocks[{block_index}][{treatment_index}] is outside signed int64"
                )
        checked_blocks.append(block)

    assignments_per_block = factorial(treatment_count)
    total_assignments = 1
    for _ in checked_blocks:
        total_assignments *= assignments_per_block
        if total_assignments > _MAX_ASSIGNMENTS:
            raise ValueError("labelled permutation count exceeds 65536")

    block_ranks = tuple(_doubled_midranks(block) for block in checked_blocks)
    doubled_rank_sums = tuple(
        sum(ranks[treatment] for ranks in block_ranks)
        for treatment in range(treatment_count)
    )
    center = len(checked_blocks) * (treatment_count + 1)
    observed_dispersion = sum(
        (rank_sum - center) ** 2 for rank_sum in doubled_rank_sums
    )

    labelled_permutations = tuple(permutations(range(treatment_count)))
    qualifying = 0
    for assignments in product(
        labelled_permutations,
        repeat=len(checked_blocks),
    ):
        candidate_sums = [0] * treatment_count
        for ranks, assignment in zip(block_ranks, assignments, strict=True):
            for treatment, source in enumerate(assignment):
                candidate_sums[treatment] += ranks[source]
        candidate_dispersion = sum(
            (rank_sum - center) ** 2 for rank_sum in candidate_sums
        )
        if candidate_dispersion >= observed_dispersion:
            qualifying += 1

    return ExactFriedmanTest(
        doubled_rank_sums,
        observed_dispersion,
        qualifying,
        total_assignments,
        Fraction(qualifying, total_assignments),
    )
```

## Example

```python
def independent_dispersion(blocks: tuple[tuple[int, ...], ...]) -> int:
    treatment_count = len(blocks[0])
    rank_sums = [0] * treatment_count
    for block in blocks:
        ordered = sorted(block)
        for treatment, value in enumerate(block):
            occupied_ranks = tuple(
                position + 1
                for position, ordered_value in enumerate(ordered)
                if ordered_value == value
            )
            rank_sums[treatment] += (
                2 * sum(occupied_ranks) // len(occupied_ranks)
            )
    center = len(blocks) * (treatment_count + 1)
    return sum((rank_sum - center) ** 2 for rank_sum in rank_sums)


def independent_result(
    blocks: tuple[tuple[int, ...], ...],
) -> tuple[int, int, Fraction]:
    treatment_count = len(blocks[0])
    all_permutations = tuple(permutations(range(treatment_count)))
    observed = independent_dispersion(blocks)
    qualifying = 0
    total = 0
    for assignments in product(all_permutations, repeat=len(blocks)):
        permuted = tuple(
            tuple(block[source] for source in assignment)
            for block, assignment in zip(blocks, assignments, strict=True)
        )
        qualifying += independent_dispersion(permuted) >= observed
        total += 1
    return observed, total, Fraction(qualifying, total)


blocks = ((1, 1, 4), (2, 5, 3), (2, 2, 7))
observed = exact_friedman_test(blocks)
expected_dispersion, expected_total, expected_p = independent_result(blocks)

column_permutation = (2, 0, 1)
permuted_columns = exact_friedman_test(
    tuple(
        tuple(block[index] for index in column_permutation)
        for block in blocks
    )
)
permuted_blocks = exact_friedman_test(tuple(reversed(blocks)))
transformed = exact_friedman_test(
    tuple(tuple(value * 3 + 1 for value in block) for block in blocks)
)
all_flat = exact_friedman_test(((7, 7, 7), (9, 9, 9)))
boundary = exact_friedman_test(
    tuple((index, index + 1, index + 2) for index in range(6))
)
four_by_three = exact_friedman_test(
    ((1, 2, 3, 4), (2, 1, 4, 3), (1, 3, 2, 4))
)
try:
    exact_friedman_test(
        tuple((index, index + 1, index + 2) for index in range(7))
    )
except ValueError:
    cap_enforced = True
else:
    cap_enforced = False

assert (
    observed.rank_dispersion == expected_dispersion
    and observed.total_assignments == expected_total == 216
    and observed.upper_tail_p == expected_p
    and sorted(permuted_columns.doubled_rank_sums)
    == sorted(observed.doubled_rank_sums)
    and permuted_columns.rank_dispersion == observed.rank_dispersion
    and permuted_columns.upper_tail_p == observed.upper_tail_p
    and permuted_blocks.upper_tail_p == observed.upper_tail_p
    and transformed == observed
    and all_flat.rank_dispersion == 0
    and all_flat.upper_tail_p == Fraction(1, 1)
    and _doubled_midranks((1, 1, 2)) == (3, 3, 6)
    and boundary.total_assignments == 46_656
    and four_by_three.total_assignments == 13_824
    and cap_enforced
)
```

## Trade-offs and Limitations

Ranking takes `O(B * K log K)` time for `B` blocks and `K` treatments.
Enumeration takes `O(B * K * (K!)**B)` integer work and `O(B * K)` auxiliary
state plus `O(K * K!)` space for the materialized within-block permutation pool;
the explicit assignment cap and `K <= 5` bound both. Tied observations still
generate all labelled assignments, so visually identical permutations retain
their correct multiplicity.

The integer dispersion is proportional to the usual raw Friedman statistic:
`Q = 3 * dispersion / (B * K * (K + 1))`. A tie correction is constant across
the admitted conditional permutations and cannot change their upper-tail
ordering, so it is intentionally omitted. The result contains no alpha decision,
effect size, confidence interval, or treatment-by-treatment follow-up.

## Related Snippets

<!-- catalog:related:start -->
- [Compute an Exact Two-Sided Wilcoxon Signed-Rank Randomization Test for Bounded Integer Pairs](compute-an-exact-two-sided-wilcoxon-signed-rank-randomization-test-for-bounded-integer-pairs.md)
- [Compute an Exact Two-Sided Mann-Whitney U Permutation Test for Bounded Integer Samples](compute-an-exact-two-sided-mann-whitney-u-permutation-test-for-bounded-integer-samples.md)
- [Compute an Exact Two-Sample Permutation Test for a Mean Difference](compute-an-exact-two-sample-permutation-test-for-a-mean-difference.md)
<!-- catalog:related:end -->
