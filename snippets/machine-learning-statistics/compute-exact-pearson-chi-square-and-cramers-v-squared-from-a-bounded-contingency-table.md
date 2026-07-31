---
title: "Compute Exact Pearson Chi-Square and Cramer's V Squared from a Bounded Contingency Table"
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
  - compute-mutual-information-from-a-bounded-integer-contingency-table.md
  - compute-fishers-exact-test-with-a-probability-ordered-two-sided-p-value.md
  - build-an-exact-bounded-multiclass-confusion-matrix.md
---

# Compute Exact Pearson Chi-Square and Cramer's V Squared from a Bounded Contingency Table

## Idea and Problem

Measure categorical association from bounded integer counts while retaining exact rational evidence.

Pearson's chi-square compares each observed cell with the count expected from
its row and column marginals. Rearranging the usual term avoids constructing a
rounded expected count:

`(observed * total - row_total * column_total) ** 2 /
(total * row_total * column_total)`.

Cramer's V squared divides the summed statistic by the sample size and the
smaller active dimension minus one. Returning V squared avoids an irrational
square root and preserves the complete result as a `Fraction`.

## When to Use

Use this algorithm for a small complete contingency table when exact arithmetic
is useful for reference tests, reproducible descriptive evidence, or comparing
the strength of unsigned association across tables with the same category
policy. Zero-marginal padding can remain in the table without changing the
active calculation.

Use a statistics package when the work needs a p-value, a sampling model,
continuity or bias correction, sparse-table guidance, confidence intervals,
weights, or a significance decision. Exact rational arithmetic here does not
turn Pearson's asymptotic inferential procedure into an exact test.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction

_MAX_ROWS = 64
_MAX_COLUMNS = 64
_MAX_SIGNED_64 = (1 << 63) - 1


@dataclass(frozen=True, slots=True)
class PearsonAssociationEvidence:
    row_totals: tuple[int, ...]
    column_totals: tuple[int, ...]
    sample_size: int
    active_row_count: int
    active_column_count: int
    degrees_of_freedom: int
    chi_square: Fraction
    cramers_v_squared: Fraction


def exact_pearson_association(
    table: tuple[tuple[int, ...], ...],
) -> PearsonAssociationEvidence:
    """Return exact Pearson chi-square and uncorrected Cramer's V squared."""
    if type(table) is not tuple:
        raise TypeError("table must be an exact tuple")
    if not 1 <= len(table) <= _MAX_ROWS:
        raise ValueError("row count is outside the supported range")
    if type(table[0]) is not tuple:
        raise TypeError("every row must be an exact tuple")

    column_count = len(table[0])
    if not 1 <= column_count <= _MAX_COLUMNS:
        raise ValueError("column count is outside the supported range")

    row_totals: list[int] = []
    column_totals = [0] * column_count
    sample_size = 0
    for row_index, row in enumerate(table):
        if type(row) is not tuple:
            raise TypeError(f"table[{row_index}] must be an exact tuple")
        if len(row) != column_count:
            raise ValueError("table must be rectangular")

        row_total = 0
        for column_index, count in enumerate(row):
            if type(count) is not int:
                raise TypeError(f"table[{row_index}][{column_index}] must be an exact integer")
            if not 0 <= count <= _MAX_SIGNED_64:
                raise ValueError(
                    f"table[{row_index}][{column_index}] is outside the supported range"
                )
            row_total += count
            column_totals[column_index] += count
            sample_size += count
            if sample_size > _MAX_SIGNED_64:
                raise ValueError("grand total exceeds the signed 64-bit range")
        row_totals.append(row_total)

    if sample_size == 0:
        raise ValueError("grand total must be positive")
    active_rows = tuple(index for index, total in enumerate(row_totals) if total > 0)
    active_columns = tuple(index for index, total in enumerate(column_totals) if total > 0)
    if len(active_rows) < 2 or len(active_columns) < 2:
        raise ValueError("at least two positive row and column marginals are required")

    chi_square = Fraction()
    for row_index in active_rows:
        row_total = row_totals[row_index]
        for column_index in active_columns:
            column_total = column_totals[column_index]
            difference = table[row_index][column_index] * sample_size - row_total * column_total
            chi_square += Fraction(
                difference * difference,
                sample_size * row_total * column_total,
            )

    degrees_of_freedom = (len(active_rows) - 1) * (len(active_columns) - 1)
    cramers_v_squared = chi_square / (
        sample_size * min(len(active_rows) - 1, len(active_columns) - 1)
    )
    return PearsonAssociationEvidence(
        row_totals=tuple(row_totals),
        column_totals=tuple(column_totals),
        sample_size=sample_size,
        active_row_count=len(active_rows),
        active_column_count=len(active_columns),
        degrees_of_freedom=degrees_of_freedom,
        chi_square=chi_square,
        cramers_v_squared=cramers_v_squared,
    )
```

## Example

```python
def direct_chi_square_oracle(table: tuple[tuple[int, ...], ...]) -> Fraction:
    row_totals = tuple(sum(row) for row in table)
    column_totals = tuple(sum(column) for column in zip(*table, strict=True))
    total = sum(row_totals)
    statistic = Fraction()
    for row_index, row_total in enumerate(row_totals):
        if row_total == 0:
            continue
        for column_index, column_total in enumerate(column_totals):
            if column_total == 0:
                continue
            expected = Fraction(row_total * column_total, total)
            difference = Fraction(table[row_index][column_index]) - expected
            statistic += difference * difference / expected
    return statistic


perfect = exact_pearson_association(((10, 0), (0, 10)))
independent = exact_pearson_association(((1, 1), (2, 2)))
padded = exact_pearson_association(
    (
        (10, 0, 0),
        (0, 10, 0),
        (0, 0, 0),
    )
)
permuted = exact_pearson_association(((10, 0), (0, 10))[::-1])

assert perfect == PearsonAssociationEvidence(
    row_totals=(10, 10),
    column_totals=(10, 10),
    sample_size=20,
    active_row_count=2,
    active_column_count=2,
    degrees_of_freedom=1,
    chi_square=Fraction(20),
    cramers_v_squared=Fraction(1),
)
assert independent.chi_square == Fraction(0)
assert independent.cramers_v_squared == Fraction(0)
assert padded.chi_square == perfect.chi_square
assert padded.cramers_v_squared == perfect.cramers_v_squared
assert padded.active_row_count == padded.active_column_count == 2
assert permuted.chi_square == perfect.chi_square
assert perfect.chi_square == direct_chi_square_oracle(((10, 0), (0, 10)))


def rejected(candidate: object) -> bool:
    try:
        exact_pearson_association(candidate)
    except (TypeError, ValueError):
        return True
    return False


assert rejected(((0, 0), (0, 0)))
assert rejected(((1, 2), (3,)))
assert rejected(((1, 2),))
assert rejected(((True, 0), (0, 1)))
assert rejected(((_MAX_SIGNED_64, 0), (0, 1)))
```

## Trade-offs and Limitations

For `r` rows and `c` columns, validation, marginal accumulation, and statistic
calculation use `O(r*c)` exact-arithmetic operations and `O(r+c)` auxiliary
memory. Python integers safely hold intermediate products wider than 64 bits,
while the admitted grand total is capped at the positive signed-64 range.

Rows and columns with zero marginal totals are retained in the evidence but
excluded from the active dimensions. A zero cell inside positive marginals is
not structural absence and still contributes through its non-zero expected
count. At least two active rows and columns keep Cramer's V squared defined.

The reported V squared is the ordinary uncorrected empirical association
measure. It is unsigned and invariant to row or column permutation. The result
does not provide effect direction, a p-value, a threshold, Yates correction,
bias correction, structural-zero modeling, sparse-table advice, weights,
floating observations, uncertainty, or evidence of causality.

## Related Snippets

<!-- catalog:related:start -->
- [Compute Mutual Information from a Bounded Integer Contingency Table](compute-mutual-information-from-a-bounded-integer-contingency-table.md)
- [Compute Fisher's Exact Test with a Probability-Ordered Two-Sided P-Value](compute-fishers-exact-test-with-a-probability-ordered-two-sided-p-value.md)
- [Build an Exact Bounded Multiclass Confusion Matrix](build-an-exact-bounded-multiclass-confusion-matrix.md)
<!-- catalog:related:end -->
