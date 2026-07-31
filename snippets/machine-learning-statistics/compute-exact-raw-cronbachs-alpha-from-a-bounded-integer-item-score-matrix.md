---
title: "Compute Exact Raw Cronbach's Alpha from a Bounded Integer Item-Score Matrix"
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
  - compute-exact-fleiss-kappa-from-a-bounded-rating-count-matrix.md
  - compute-exact-squared-pearson-correlation-with-direction.md
  - accumulate-and-merge-finite-mean-and-variance-statistics-under-a-count-limit.md
---

# Compute Exact Raw Cronbach's Alpha from a Bounded Integer Item-Score Matrix

## Idea and Problem

Summarize the internal consistency of complete integer item scores while retaining exact variance evidence behind raw Cronbach's alpha.

For each item, `n * sum(x²) - sum(x)²` is `n` times its centered sum of
squares. Applying the same identity to every subject's total score gives the
denominator used by raw alpha. The common scale cancels, so integer scatters
and one `Fraction` preserve the result without intermediate means or
floating-point rounding.

## When to Use

Use this calculation for a bounded rectangular matrix in which rows are
subjects, columns are equally oriented items, every cell is an integer score,
and raw rather than standardized alpha is the chosen descriptive summary. The
returned scatters make zero total-score variation and surprising negative
results inspectable instead of hiding them behind one float.

Reverse-score items and decide missing-data policy before calling this
function. Use a statistics package when item weights, standardized alpha,
ordinal models, missing observations, confidence intervals, or sampling
inference are required.

## Implementation

```python
from collections.abc import Callable
from dataclasses import dataclass
from fractions import Fraction

_MAX_SUBJECT_COUNT = 4_096
_MAX_ITEM_COUNT = 64
_MAX_CELL_COUNT = 262_144
_MIN_SCORE = -(2**31)
_MAX_SCORE = 2**31 - 1


@dataclass(frozen=True, slots=True)
class CronbachAlphaEvidence:
    subject_count: int
    item_count: int
    item_scatters: tuple[int, ...]
    total_score_scatter: int
    alpha: Fraction | None


def exact_raw_cronbach_alpha(
    matrix: tuple[tuple[int, ...], ...],
) -> CronbachAlphaEvidence:
    """Return exact raw-alpha evidence for a complete integer score matrix."""
    if type(matrix) is not tuple:
        raise TypeError("matrix must be an exact tuple")
    subject_count = len(matrix)
    if not 2 <= subject_count <= _MAX_SUBJECT_COUNT:
        raise ValueError("subject count is outside the supported range")

    first_row = matrix[0]
    if type(first_row) is not tuple:
        raise TypeError("every row must be an exact tuple")
    item_count = len(first_row)
    if not 2 <= item_count <= _MAX_ITEM_COUNT:
        raise ValueError("item count is outside the supported range")
    if subject_count * item_count > _MAX_CELL_COUNT:
        raise ValueError("matrix exceeds the cell limit")

    item_sums = [0] * item_count
    item_square_sums = [0] * item_count
    total_score_sum = 0
    total_score_square_sum = 0

    for row in matrix:
        if type(row) is not tuple:
            raise TypeError("every row must be an exact tuple")
        if len(row) != item_count:
            raise ValueError("matrix rows must have one common item count")
        row_total = 0
        for item_index, score in enumerate(row):
            if type(score) is not int:
                raise TypeError("scores must be exact integers")
            if not _MIN_SCORE <= score <= _MAX_SCORE:
                raise ValueError("score is outside the signed-32-bit range")
            item_sums[item_index] += score
            item_square_sums[item_index] += score * score
            row_total += score
        total_score_sum += row_total
        total_score_square_sum += row_total * row_total

    item_scatters = tuple(
        subject_count * square_sum - item_sum * item_sum
        for item_sum, square_sum in zip(item_sums, item_square_sums, strict=True)
    )
    total_score_scatter = subject_count * total_score_square_sum - total_score_sum * total_score_sum
    alpha = None
    if total_score_scatter != 0:
        alpha = Fraction(
            item_count * (total_score_scatter - sum(item_scatters)),
            (item_count - 1) * total_score_scatter,
        )

    return CronbachAlphaEvidence(
        subject_count=subject_count,
        item_count=item_count,
        item_scatters=item_scatters,
        total_score_scatter=total_score_scatter,
        alpha=alpha,
    )
```

## Example

```python

def _fraction_oracle(matrix: tuple[tuple[int, ...], ...]) -> Fraction | None:
    item_count = len(matrix[0])

    def centered_sum(values: tuple[int, ...]) -> Fraction:
        mean = Fraction(sum(values), len(values))
        return sum((Fraction(value) - mean) ** 2 for value in values)

    item_variation = sum(
        centered_sum(tuple(row[item_index] for row in matrix)) for item_index in range(item_count)
    )
    total_variation = centered_sum(tuple(sum(row) for row in matrix))
    if total_variation == 0:
        return None
    return Fraction(item_count, item_count - 1) * (1 - item_variation / total_variation)


def _raises(
    error_type: type[BaseException],
    function: Callable[..., object],
    *args: object,
) -> bool:
    try:
        function(*args)
    except error_type:
        return True
    return False


def _permutations(values: range) -> tuple[tuple[int, ...], ...]:
    from itertools import permutations

    return tuple(permutations(values))


perfect = ((0, 0), (1, 1), (2, 2))
assert exact_raw_cronbach_alpha(perfect) == CronbachAlphaEvidence(
    subject_count=3,
    item_count=2,
    item_scatters=(6, 6),
    total_score_scatter=24,
    alpha=Fraction(1),
)
assert exact_raw_cronbach_alpha(((0, 2), (1, 0), (2, 1))).alpha == -2
assert exact_raw_cronbach_alpha(((0, 1), (1, 0))).alpha is None

checked = 0
for subject_count in range(2, 5):
    for item_count in range(2, 4):
        cell_count = subject_count * item_count
        for encoded_matrix in range(1 << cell_count):
            cells = tuple((encoded_matrix >> index) & 1 for index in range(cell_count))
            matrix = tuple(
                tuple(cells[row * item_count : (row + 1) * item_count])
                for row in range(subject_count)
            )
            assert exact_raw_cronbach_alpha(matrix).alpha == _fraction_oracle(matrix)
            checked += 1

baseline = ((3, -1, 4), (0, 2, 5), (7, 1, -2), (4, 4, 4))
expected = exact_raw_cronbach_alpha(baseline).alpha
for row_order in _permutations(range(len(baseline))):
    permuted_rows = tuple(baseline[index] for index in row_order)
    assert exact_raw_cronbach_alpha(permuted_rows).alpha == expected
for item_order in _permutations(range(len(baseline[0]))):
    permuted_items = tuple(tuple(row[index] for index in item_order) for row in baseline)
    assert exact_raw_cronbach_alpha(permuted_items).alpha == expected

extremes = ((-(2**31), 2**31 - 1), (2**31 - 1, -(2**31)))
assert exact_raw_cronbach_alpha(extremes).alpha is None
assert _raises(TypeError, exact_raw_cronbach_alpha, [[0, 0], [1, 1]])
assert _raises(ValueError, exact_raw_cronbach_alpha, ((0, 0),))
assert _raises(ValueError, exact_raw_cronbach_alpha, ((0,), (1,)))
assert _raises(ValueError, exact_raw_cronbach_alpha, ((0, 1), (1,)))
assert _raises(TypeError, exact_raw_cronbach_alpha, ((False, 0), (1, 1)))
assert _raises(ValueError, exact_raw_cronbach_alpha, ((-(2**31) - 1, 0), (0, 0)))
assert checked == 5_008
```

## Trade-offs and Limitations

For `N` subjects and `K` items, validation and aggregation take `O(N * K)`
time and retain `O(K)` auxiliary integer totals. Exact integer and `Fraction`
arithmetic avoid numeric drift, although their cost still depends on bounded
intermediate bit lengths. At most 262,144 signed-32-bit cells are admitted.

This is raw alpha for a complete equally weighted matrix, not standardized
alpha. A constant item is permitted, while zero variation in subject total
scores makes the coefficient undefined and returns `None`. Negative alpha is
retained because clamping would hide evidence of opposing item movement.

Alpha is descriptive. A high value does not prove reliability,
unidimensionality, validity, or item quality, and this function supplies no
reverse coding, missing-data policy, uncertainty estimate, hypothesis test, or
interpretation threshold.

## Related Snippets

<!-- catalog:related:start -->
- [Compute Exact Fleiss' Kappa from a Bounded Rating-Count Matrix](compute-exact-fleiss-kappa-from-a-bounded-rating-count-matrix.md)
- [Compute Exact Squared Pearson Correlation with Direction](compute-exact-squared-pearson-correlation-with-direction.md)
- [Accumulate and Merge Finite Mean and Variance Statistics Under a Count Limit](accumulate-and-merge-finite-mean-and-variance-statistics-under-a-count-limit.md)
<!-- catalog:related:end -->
