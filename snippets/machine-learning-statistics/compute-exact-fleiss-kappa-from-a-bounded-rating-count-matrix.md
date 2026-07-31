---
title: "Compute Exact Fleiss' Kappa from a Bounded Rating-Count Matrix"
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
  - build-an-exact-bounded-multiclass-confusion-matrix.md
  - compute-exact-pearson-chi-square-and-cramers-v-squared-from-a-bounded-contingency-table.md
---

# Compute Exact Fleiss' Kappa from a Bounded Rating-Count Matrix

## Idea and Problem

Measure nominal agreement among interchangeable raters while retaining the exact observed and chance evidence behind the adjustment.

Each matrix row represents one subject and each column one declared category.
A cell counts how many raters assigned that category to that subject. Fleiss'
kappa first measures how often two different ratings of the same subject agree,
then adjusts that proportion by agreement expected from the pooled category
frequencies.

All inputs are counts. Pair totals, pooled proportions, and the adjusted result
can therefore remain exact `Fraction` values instead of accumulating rounding
error. Category labels and individual rater identities are deliberately absent
from this representation.

## When to Use

Use this calculation when every subject receives the same fixed number of
nominal ratings, the raters are treated as interchangeable, and a transparent
chance-adjusted agreement summary is useful. A rating-count matrix is often a
natural aggregate when different members of a larger rater pool may evaluate
different subjects.

Use Cohen's kappa when two named raters and their separate marginals matter.
Use a weighted agreement measure when category order or distance carries
meaning, and use a statistics package when confidence intervals, hypothesis
tests, missing ratings, subject weights, or a sampling model are required.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction

_MAX_SUBJECT_COUNT = 4_096
_MAX_CATEGORY_COUNT = 64
_MAX_CELL_COUNT = 256
_MAX_RATINGS_PER_SUBJECT = 256


@dataclass(frozen=True, slots=True)
class FleissKappaEvidence:
    subject_count: int
    category_count: int
    ratings_per_subject: int
    category_totals: tuple[int, ...]
    agreeing_pair_count: int
    total_within_subject_pair_count: int
    observed_agreement: Fraction
    chance_agreement: Fraction
    kappa: Fraction | None


def exact_fleiss_kappa(
    matrix: tuple[tuple[int, ...], ...],
) -> FleissKappaEvidence:
    """Return exact Fleiss-kappa evidence for one rating-count matrix."""
    if type(matrix) is not tuple:
        raise TypeError("matrix must be an exact tuple")
    subject_count = len(matrix)
    if not 2 <= subject_count <= _MAX_SUBJECT_COUNT:
        raise ValueError("subject count is outside the supported range")

    first_row = matrix[0]
    if type(first_row) is not tuple:
        raise TypeError("matrix[0] must be an exact tuple")
    category_count = len(first_row)
    if not 2 <= category_count <= _MAX_CATEGORY_COUNT:
        raise ValueError("category count is outside the supported range")

    category_totals = [0] * category_count
    agreeing_pair_count = 0
    ratings_per_subject: int | None = None

    for subject_index, row in enumerate(matrix):
        if type(row) is not tuple:
            raise TypeError(f"matrix[{subject_index}] must be an exact tuple")
        if len(row) != category_count:
            raise ValueError("every subject row must have the same category count")

        row_total = 0
        for category_index, count in enumerate(row):
            if type(count) is not int:
                raise TypeError(
                    f"matrix[{subject_index}][{category_index}] must be an exact integer"
                )
            if not 0 <= count <= _MAX_CELL_COUNT:
                raise ValueError(
                    f"matrix[{subject_index}][{category_index}] is outside the supported range"
                )
            row_total += count
            category_totals[category_index] += count
            agreeing_pair_count += count * (count - 1) // 2

        if not 2 <= row_total <= _MAX_RATINGS_PER_SUBJECT:
            raise ValueError("ratings per subject are outside the supported range")
        if ratings_per_subject is None:
            ratings_per_subject = row_total
        elif row_total != ratings_per_subject:
            raise ValueError("every subject must have the same number of ratings")

    if ratings_per_subject is None:
        raise AssertionError("a validated matrix must contain subject rows")

    total_within_subject_pair_count = (
        subject_count * ratings_per_subject * (ratings_per_subject - 1) // 2
    )
    observed_agreement = Fraction(
        agreeing_pair_count,
        total_within_subject_pair_count,
    )
    total_rating_count = subject_count * ratings_per_subject
    chance_agreement = Fraction(
        sum(total * total for total in category_totals),
        total_rating_count * total_rating_count,
    )
    kappa = (
        None
        if chance_agreement == 1
        else (observed_agreement - chance_agreement) / (1 - chance_agreement)
    )

    return FleissKappaEvidence(
        subject_count=subject_count,
        category_count=category_count,
        ratings_per_subject=ratings_per_subject,
        category_totals=tuple(category_totals),
        agreeing_pair_count=agreeing_pair_count,
        total_within_subject_pair_count=total_within_subject_pair_count,
        observed_agreement=observed_agreement,
        chance_agreement=chance_agreement,
        kappa=kappa,
    )
```

## Example

```python
def direct_rating_pair_evidence(
    matrix: tuple[tuple[int, ...], ...],
) -> FleissKappaEvidence:
    expanded_rows = tuple(
        tuple(category for category, count in enumerate(row) for _ in range(count))
        for row in matrix
    )
    agreeing = sum(
        left == right
        for ratings in expanded_rows
        for left_index, left in enumerate(ratings)
        for right in ratings[left_index + 1 :]
    )
    pair_total = sum(len(ratings) * (len(ratings) - 1) // 2 for ratings in expanded_rows)
    global_ratings = tuple(rating for row in expanded_rows for rating in row)
    chance_matches = sum(left == right for left in global_ratings for right in global_ratings)
    observed = Fraction(agreeing, pair_total)
    chance = Fraction(chance_matches, len(global_ratings) ** 2)
    category_count = len(matrix[0])
    totals = tuple(sum(row[category] for row in matrix) for category in range(category_count))
    return FleissKappaEvidence(
        subject_count=len(matrix),
        category_count=category_count,
        ratings_per_subject=len(expanded_rows[0]),
        category_totals=totals,
        agreeing_pair_count=agreeing,
        total_within_subject_pair_count=pair_total,
        observed_agreement=observed,
        chance_agreement=chance,
        kappa=None if chance == 1 else (observed - chance) / (1 - chance),
    )


def compositions(total: int, width: int):
    from itertools import product

    return tuple(
        values for values in product(range(total + 1), repeat=width) if sum(values) == total
    )


def verify_small_matrices() -> int:
    from itertools import product

    checked = 0
    for subject_count in (2, 3):
        for category_count in (2, 3):
            for rating_count in (2, 3):
                rows = compositions(rating_count, category_count)
                for matrix in product(rows, repeat=subject_count):
                    assert exact_fleiss_kappa(matrix) == direct_rating_pair_evidence(matrix)
                    checked += 1
    return checked


perfect = exact_fleiss_kappa(((3, 0), (0, 3)))
negative = exact_fleiss_kappa(((1, 1), (1, 1)))
undefined = exact_fleiss_kappa(((3, 0, 0), (3, 0, 0)))
permuted = exact_fleiss_kappa(((0, 3), (3, 0)))


def rejected(candidate: object) -> bool:
    try:
        exact_fleiss_kappa(candidate)
    except (TypeError, ValueError):
        return True
    return False


invalid_matrices = (
    ((1, 0),),
    [(1, 1), (1, 1)],
    ((1, 1), (2, 0, 0)),
    ((1, 1), (1, 2)),
    ((True, 1), (1, 1)),
)

assert (
    perfect.kappa,
    negative.kappa,
    undefined.kappa,
    permuted.kappa,
    verify_small_matrices(),
    sum(rejected(candidate) for candidate in invalid_matrices),
) == (Fraction(1), Fraction(-1), None, Fraction(1), 1_468, 5)
```

## Trade-offs and Limitations

For `N` subjects and `K` categories, validation and aggregation take
`O(N * K)` time and retain `O(K)` auxiliary counts. Exact integers and
`Fraction` avoid numeric drift, although their arithmetic cost grows with the
bit length of intermediate totals. The explicit caps admit at most 262,144
cells and at most 1,048,576 ratings.

The statistic uses pooled category marginals and interchangeable ratings. Even
with exactly two ratings per subject, that is not generally Cohen's kappa for
two named raters with separate marginals. A zero chance-adjustment denominator
returns `None`, while a valid negative result is retained rather than clipped.

Kappa does not explain disagreements, establish rater accuracy, choose a
category, or imply that the subjects challenged the raters meaningfully. This
bounded calculation provides no weighting, missing-data policy, uncertainty,
significance test, confidence interval, sampling correction, or qualitative
interpretation threshold.

## Related Snippets

<!-- catalog:related:start -->
- [Compute Exact Unweighted Cohen's Kappa from a Confusion Matrix](compute-exact-unweighted-cohens-kappa-from-a-confusion-matrix.md)
- [Build an Exact Bounded Multiclass Confusion Matrix](build-an-exact-bounded-multiclass-confusion-matrix.md)
- [Compute Exact Pearson Chi-Square and Cramer's V Squared from a Bounded Contingency Table](compute-exact-pearson-chi-square-and-cramers-v-squared-from-a-bounded-contingency-table.md)
<!-- catalog:related:end -->
