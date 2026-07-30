---
title: "Compute Mutual Information from a Bounded Integer Contingency Table"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - build-an-exact-bounded-multiclass-confusion-matrix.md
  - compute-exact-unweighted-cohens-kappa-from-a-confusion-matrix.md
  - measure-drift-between-two-fixed-bin-count-distributions-with-psi.md
---

# Compute Mutual Information from a Bounded Integer Contingency Table

## Idea and Problem

Measure categorical dependence from exact joint counts while retaining the evidence behind the rounded result.

Mutual information compares each observed joint frequency with the frequency
expected from its row and column marginals. Mathematically, the exact value is
zero exactly when the empirical table factors into those marginals; larger
values indicate more information about one category is carried by the other.

The counts and marginals remain exact integers. Logarithms generally do not
have exact finite representations, so every per-cell contribution and the
reported result are explicitly binary64 values measured in bits.

## When to Use

Use this calculation when two already-binned categorical variables are
represented by one complete contingency table and an unsigned, label-order
independent dependence summary is useful. Rectangular tables are supported, so
the variables do not need to share categories or represent agreement.

Use Cohen's kappa when matching category labels and chance-corrected agreement
are the question. Use correlation for meaningful numeric ordering, and use a
statistics library when bias correction, smoothing, normalization,
significance testing, sample weights, uncertainty, or sparse high-cardinality
tables are required.

## Implementation

```python
from dataclasses import dataclass
from math import fsum, isfinite, log, log1p, ulp

_MAX_ROWS = 256
_MAX_COLUMNS = 256
_MAX_CELLS = 65_536
_MAX_SIGNED_64 = (1 << 63) - 1
_LOG_2 = log(2.0)


@dataclass(frozen=True, slots=True)
class MutualInformationTerm:
    row_index: int
    column_index: int
    count: int
    row_total: int
    column_total: int
    contribution_bits: float


@dataclass(frozen=True, slots=True)
class MutualInformationEvidence:
    row_totals: tuple[int, ...]
    column_totals: tuple[int, ...]
    total: int
    terms: tuple[MutualInformationTerm, ...]
    mutual_information_bits: float


def mutual_information_from_counts(
    table: tuple[tuple[int, ...], ...],
) -> MutualInformationEvidence:
    """Return exact marginals and rounded mutual-information evidence."""
    if type(table) is not tuple:
        raise TypeError("table must be an exact tuple")
    if not 1 <= len(table) <= _MAX_ROWS:
        raise ValueError("row count is outside the supported range")
    if type(table[0]) is not tuple:
        raise TypeError("every row must be an exact tuple")

    column_count = len(table[0])
    if not 1 <= column_count <= _MAX_COLUMNS:
        raise ValueError("column count is outside the supported range")
    if len(table) * column_count > _MAX_CELLS:
        raise ValueError("cell count exceeds the supported limit")

    row_totals: list[int] = []
    column_totals = [0] * column_count
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
        row_totals.append(row_total)

    total = sum(row_totals)
    if not 1 <= total <= _MAX_SIGNED_64:
        raise ValueError("aggregate count is outside the supported range")

    terms: list[MutualInformationTerm] = []
    contributions: list[float] = []
    for row_index, row in enumerate(table):
        row_total = row_totals[row_index]
        for column_index, count in enumerate(row):
            if count == 0:
                continue
            column_total = column_totals[column_index]
            observed_product = count * total
            marginal_product = row_total * column_total
            if observed_product == marginal_product:
                contribution_bits = 0.0
            else:
                difference = observed_product - marginal_product
                if 2 * abs(difference) <= marginal_product:
                    pointwise_bits = log1p(difference / marginal_product) / _LOG_2
                else:
                    pointwise_bits = log(observed_product / marginal_product) / _LOG_2
                contribution_bits = (count / total) * pointwise_bits
            if not isfinite(contribution_bits):
                raise ArithmeticError("a contribution is not finite")
            terms.append(
                MutualInformationTerm(
                    row_index=row_index,
                    column_index=column_index,
                    count=count,
                    row_total=row_total,
                    column_total=column_total,
                    contribution_bits=contribution_bits,
                )
            )
            contributions.append(contribution_bits)

    mutual_information_bits = fsum(contributions)
    rounding_scale = fsum(abs(value) for value in contributions)
    negative_tolerance = 64 * ulp(rounding_scale)
    if mutual_information_bits < -negative_tolerance:
        raise ArithmeticError("rounded contributions violate non-negativity")
    mutual_information_bits = max(0.0, mutual_information_bits)

    return MutualInformationEvidence(
        row_totals=tuple(row_totals),
        column_totals=tuple(column_totals),
        total=total,
        terms=tuple(terms),
        mutual_information_bits=mutual_information_bits,
    )
```

## Example

```python


perfect = mutual_information_from_counts(((2, 0), (0, 2)))
independent = mutual_information_from_counts(((1, 1), (1, 1)))
one_row = mutual_information_from_counts(((2, 3),))
large = 4_000_000_000_000_000_000
skewed = mutual_information_from_counts(((1, large), (large, 0)))

assert perfect.row_totals == (2, 2)
assert perfect.column_totals == (2, 2)
assert perfect.total == 4
assert perfect.mutual_information_bits == 1.0
assert tuple(term.contribution_bits for term in perfect.terms) == (0.5, 0.5)
assert independent.mutual_information_bits == 0.0
assert one_row.mutual_information_bits == 0.0
assert 0.99 < skewed.mutual_information_bits < 1.01


def is_rejected(candidate: object) -> bool:
    try:
        mutual_information_from_counts(candidate)  # type: ignore[arg-type]
    except TypeError:
        return True
    except ValueError:
        return True
    return False


assert is_rejected(((0, 0),))
assert is_rejected(((1,), (1, 2)))
assert is_rejected(((True,),))
```

## Trade-offs and Limitations

For `r` rows and `c` columns, the function takes `O(r*c)` time. Marginals use
`O(r+c)` memory and the immutable evidence uses `O(z)` memory for `z` non-zero
cells. Empty rows or columns are retained in the marginal vectors but create no
terms, because no positive cell can have a zero marginal.

Integer counts, marginals, and each likelihood ratio's numerator and
denominator are exact. Logarithms, multiplication by empirical probability,
and summation are rounded binary64 operations. Exact-independence cells are
recognized before logarithms. Ratios near one use `log1p` on their relative
difference; ratios farther away use `log` directly so a small positive ratio
cannot first round to zero. A final negative value within a scale-aware
rounding tolerance is clamped to zero; a larger violation raises rather than
silently presenting an impossible result. Because the reported view is
binary64, sufficiently small positive information can still round to `0.0`
for a non-factorizing exact table; inspect the exact counts or use a
high-precision statistics library when that distinction matters.

Empirical mutual information is non-negative and invariant to row or column
reordering, but it is unsigned, grows with category resolution, and is biased
upward in finite samples. It is neither normalized agreement, a significance
test, an effect direction, nor evidence of causality. The caller owns binning,
missing-value policy and interpretation.

## Related Snippets

<!-- catalog:related:start -->
- [Build an Exact Bounded Multiclass Confusion Matrix](build-an-exact-bounded-multiclass-confusion-matrix.md)
- [Compute Exact Unweighted Cohen's Kappa from a Confusion Matrix](compute-exact-unweighted-cohens-kappa-from-a-confusion-matrix.md)
- [Measure Drift Between Two Fixed-Bin Count Distributions with PSI](measure-drift-between-two-fixed-bin-count-distributions-with-psi.md)
<!-- catalog:related:end -->
