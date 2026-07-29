---
title: "Compute Exact Unweighted Cohen's Kappa from a Confusion Matrix"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - build-an-exact-bounded-multiclass-confusion-matrix.md
  - compute-exact-binary-roc-auc-with-tied-integer-scores.md
  - measure-drift-between-two-fixed-bin-count-distributions-with-psi.md
---

# Compute Exact Unweighted Cohen's Kappa from a Confusion Matrix

## Idea and Problem

Measure agreement between two raters while correcting observed agreement by the agreement expected from their category marginals.

The confusion-matrix diagonal counts exact observed matches. Row and column
marginals define the chance agreement model for the two raters. Keeping both
probabilities and their correction as `Fraction` values avoids rounding and
makes the undefined single-category degeneracy explicit.

## When to Use

Use unweighted Cohen's kappa when exactly two raters classify the same items
into the same nominal categories and every item contributes one complete pair
of ratings. Apply the same category order to rows and columns before calling
the function.

Use a weighted agreement measure for ordered categories, a multi-rater measure
for more than two raters, or a specialized statistics package when uncertainty,
hypothesis tests, missing-data policies, or sampling weights are required.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction

_MAX_CATEGORIES = 64
_MAX_INT64 = (1 << 63) - 1


@dataclass(frozen=True, slots=True)
class CohenKappaResult:
    total: int
    observed_agreement: Fraction
    chance_agreement: Fraction
    kappa: Fraction | None


def exact_unweighted_cohens_kappa(
    matrix: tuple[tuple[int, ...], ...],
) -> CohenKappaResult:
    """Return exact observed, chance, and chance-corrected agreement."""
    if type(matrix) is not tuple:
        raise TypeError("matrix must be an exact tuple")
    category_count = len(matrix)
    if not 2 <= category_count <= _MAX_CATEGORIES:
        raise ValueError("matrix size is outside the supported range")

    total = 0
    for row_index, row in enumerate(matrix):
        if type(row) is not tuple:
            raise TypeError(f"matrix[{row_index}] must be an exact tuple")
        if len(row) != category_count:
            raise ValueError("matrix must be square")
        for column_index, count in enumerate(row):
            if type(count) is not int:
                raise TypeError(f"matrix[{row_index}][{column_index}] must be an exact integer")
            if not 0 <= count <= _MAX_INT64:
                raise ValueError(
                    f"matrix[{row_index}][{column_index}] is outside the supported range"
                )
            total += count

    if not 1 <= total <= _MAX_INT64:
        raise ValueError("matrix aggregate is outside the supported range")

    row_totals = tuple(sum(row) for row in matrix)
    column_totals = tuple(
        sum(matrix[row][column] for row in range(category_count))
        for column in range(category_count)
    )
    observed_matches = sum(matrix[index][index] for index in range(category_count))
    chance_numerator = sum(
        row_total * column_total
        for row_total, column_total in zip(row_totals, column_totals, strict=True)
    )

    observed_agreement = Fraction(observed_matches, total)
    chance_agreement = Fraction(chance_numerator, total * total)
    kappa = (
        None
        if chance_agreement == 1
        else (observed_agreement - chance_agreement) / (1 - chance_agreement)
    )
    return CohenKappaResult(
        total=total,
        observed_agreement=observed_agreement,
        chance_agreement=chance_agreement,
        kappa=kappa,
    )
```

## Example

```python
matrix = ((20, 5), (10, 15))
permuted = ((15, 10), (5, 20))
degenerate = ((7, 0), (0, 0))

measured = exact_unweighted_cohens_kappa(matrix)
reordered = exact_unweighted_cohens_kappa(permuted)
undefined = exact_unweighted_cohens_kappa(degenerate)

assert (measured, reordered, undefined) == (
    CohenKappaResult(50, Fraction(7, 10), Fraction(1, 2), Fraction(2, 5)),
    CohenKappaResult(50, Fraction(7, 10), Fraction(1, 2), Fraction(2, 5)),
    CohenKappaResult(7, Fraction(1, 1), Fraction(1, 1), None),
)
```

## Trade-offs and Limitations

For `K` categories, validation and marginal calculation take `O(K^2)` time.
The row and column marginals use `O(K)` auxiliary memory. Python integer and
`Fraction` arithmetic is exact, but multiplication and fraction reduction are
not constant-cost as operand bit lengths grow.

Chance agreement is the sum of the products of the two raters' marginal
category proportions. If all observations occupy one shared category, observed
and chance agreement are both one, so the correction has a zero denominator
and `kappa=None` is returned. Negative kappa values remain meaningful outputs
under this definition rather than validation failures.

Kappa is sensitive to category prevalence and rater marginals; it is not a
standalone measure of practical importance or rater quality. This function does
not support ordinal weights, more than two raters, missing ratings, confidence
intervals, hypothesis tests, alpha decisions, or causal interpretation.

## Related Snippets

<!-- catalog:related:start -->
- [Build an Exact Bounded Multiclass Confusion Matrix](build-an-exact-bounded-multiclass-confusion-matrix.md)
- [Compute Exact Binary ROC AUC with Tied Integer Scores](compute-exact-binary-roc-auc-with-tied-integer-scores.md)
- [Measure Drift Between Two Fixed-Bin Count Distributions with PSI](measure-drift-between-two-fixed-bin-count-distributions-with-psi.md)
<!-- catalog:related:end -->
