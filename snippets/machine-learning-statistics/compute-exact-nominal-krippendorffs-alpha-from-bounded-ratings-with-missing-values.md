---
title: "Compute Exact Nominal Krippendorff's Alpha from Bounded Ratings with Missing Values"
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
  - compute-exact-unweighted-cohens-kappa-from-a-confusion-matrix.md
  - compute-exact-raw-cronbachs-alpha-from-a-bounded-integer-item-score-matrix.md
---

# Compute Exact Nominal Krippendorff's Alpha from Bounded Ratings with Missing Values

## Idea and Problem

Measure nominal inter-rater agreement exactly when units can have different numbers of observed ratings.

For a usable unit with `m` observed ratings, each ordered pair contributes
`1 / (m - 1)` to its coincidence cell. The unit therefore contributes exactly
`m` coincidences regardless of how many ratings are missing. Units with fewer
than two observations cannot form a pair and are excluded from both observed
coincidences and pooled category marginals.

Nominal disagreement is one when categories differ and zero when they match.
Accumulating disagreeing ordered pairs directly yields exact observed and
chance-expected disagreement as `Fraction` values, without materializing a
dense coincidence matrix in the implementation.

## When to Use

Use nominal Krippendorff's alpha for a bounded unit-by-rater matrix when
categories have no meaningful order, `None` is the declared missing marker,
and usable units may contain different numbers of ratings. The returned counts
and disagreements make excluded units and undefined chance correction visible.

Use Cohen's kappa for exactly two complete raters or Fleiss' kappa for
aggregated subject counts with one common rating count. Use a statistical
package when ordinal or interval distances, sampling weights, uncertainty,
bootstrap intervals, or a missing-data model are required.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction
from itertools import product

_MAX_ALPHA_UNITS = 4_096
_MAX_ALPHA_RATERS = 64
_MAX_ALPHA_CELLS = 262_144
_MAX_ALPHA_CATEGORIES = 64


@dataclass(frozen=True, slots=True)
class NominalAlphaEvidence:
    usable_unit_count: int
    usable_rating_count: int
    category_totals: tuple[int, ...]
    observed_disagreement: Fraction
    expected_disagreement: Fraction
    alpha: Fraction | None


def exact_nominal_krippendorff_alpha(
    ratings: tuple[tuple[int | None, ...], ...],
    *,
    category_count: int,
) -> NominalAlphaEvidence:
    """Return exact nominal-alpha evidence for bounded incomplete ratings."""
    if type(category_count) is not int:
        raise TypeError("category_count must be an exact integer")
    if not 2 <= category_count <= _MAX_ALPHA_CATEGORIES:
        raise ValueError("category_count is outside 2..64")
    if type(ratings) is not tuple:
        raise TypeError("ratings must be an exact tuple")
    unit_count = len(ratings)
    if not 2 <= unit_count <= _MAX_ALPHA_UNITS:
        raise ValueError("unit count is outside 2..4096")

    first_row = ratings[0]
    if type(first_row) is not tuple:
        raise TypeError("every unit row must be an exact tuple")
    rater_count = len(first_row)
    if not 2 <= rater_count <= _MAX_ALPHA_RATERS:
        raise ValueError("rater count is outside 2..64")
    if unit_count * rater_count > _MAX_ALPHA_CELLS:
        raise ValueError("ratings exceed the cell limit")

    usable_unit_count = 0
    usable_rating_count = 0
    category_totals = [0] * category_count
    weighted_disagreeing_pairs = Fraction()

    for row in ratings:
        if type(row) is not tuple:
            raise TypeError("every unit row must be an exact tuple")
        if len(row) != rater_count:
            raise ValueError("ratings must be rectangular")

        observed: list[int] = []
        for rating in row:
            if rating is None:
                continue
            if type(rating) is not int:
                raise TypeError("ratings must be exact integers or None")
            if not 0 <= rating < category_count:
                raise ValueError("rating is outside the declared category range")
            observed.append(rating)

        if len(observed) < 2:
            continue

        usable_unit_count += 1
        usable_rating_count += len(observed)
        counts = [0] * category_count
        for rating in observed:
            counts[rating] += 1
            category_totals[rating] += 1
        ordered_pairs = len(observed) * (len(observed) - 1)
        agreeing_pairs = sum(count * (count - 1) for count in counts)
        weighted_disagreeing_pairs += Fraction(
            ordered_pairs - agreeing_pairs,
            len(observed) - 1,
        )

    if usable_unit_count == 0:
        raise ValueError("at least one unit must contain two observed ratings")

    observed_disagreement = weighted_disagreeing_pairs / usable_rating_count
    expected_disagreement = Fraction(
        usable_rating_count * usable_rating_count - sum(total * total for total in category_totals),
        usable_rating_count * (usable_rating_count - 1),
    )
    alpha = None
    if expected_disagreement != 0:
        alpha = 1 - observed_disagreement / expected_disagreement

    return NominalAlphaEvidence(
        usable_unit_count=usable_unit_count,
        usable_rating_count=usable_rating_count,
        category_totals=tuple(category_totals),
        observed_disagreement=observed_disagreement,
        expected_disagreement=expected_disagreement,
        alpha=alpha,
    )
```

## Example

```python

def coincidence_oracle(
    ratings: tuple[tuple[int | None, ...], ...],
    category_count: int,
) -> NominalAlphaEvidence | None:
    coincidence = [[Fraction() for _ in range(category_count)] for _ in range(category_count)]
    usable_units = 0
    usable_ratings = 0
    totals = [0] * category_count

    for row in ratings:
        observed = tuple(rating for rating in row if rating is not None)
        if len(observed) < 2:
            continue
        usable_units += 1
        usable_ratings += len(observed)
        for category in observed:
            totals[category] += 1
        for left_index, left in enumerate(observed):
            for right_index, right in enumerate(observed):
                if left_index != right_index:
                    coincidence[left][right] += Fraction(1, len(observed) - 1)

    if usable_units == 0:
        return None
    observed_disagreement = (
        sum(
            coincidence[left][right]
            for left in range(category_count)
            for right in range(category_count)
            if left != right
        )
        / usable_ratings
    )
    expected_disagreement = sum(
        totals[left] * totals[right]
        for left in range(category_count)
        for right in range(category_count)
        if left != right
    ) / Fraction(usable_ratings * (usable_ratings - 1))
    alpha = None
    if expected_disagreement:
        alpha = 1 - observed_disagreement / expected_disagreement
    return NominalAlphaEvidence(
        usable_units,
        usable_ratings,
        tuple(totals),
        observed_disagreement,
        expected_disagreement,
        alpha,
    )


ratings = (
    (0, 0, None),
    (1, 1, 1),
    (0, 1, None),
    (None, None, 0),
)
assert exact_nominal_krippendorff_alpha(ratings, category_count=2) == NominalAlphaEvidence(
    usable_unit_count=3,
    usable_rating_count=7,
    category_totals=(3, 4),
    observed_disagreement=Fraction(2, 7),
    expected_disagreement=Fraction(4, 7),
    alpha=Fraction(1, 2),
)
assert exact_nominal_krippendorff_alpha(
    ((0, 1), (0, 1)),
    category_count=2,
).alpha == Fraction(-1, 2)
assert (
    exact_nominal_krippendorff_alpha(
        ((0, 0), (0, None)),
        category_count=2,
    ).alpha
    is None
)

checked = 0
values = (None, 0, 1)
for unit_count, rater_count in ((2, 2), (2, 3), (3, 2)):
    for cells in product(values, repeat=unit_count * rater_count):
        matrix = tuple(
            tuple(cells[index : index + rater_count]) for index in range(0, len(cells), rater_count)
        )
        expected = coincidence_oracle(matrix, 2)
        if expected is None:
            continue
        assert exact_nominal_krippendorff_alpha(matrix, category_count=2) == expected
        checked += 1

assert checked == 1_340
```

## Trade-offs and Limitations

For `U` units, `R` rater columns, and `C` declared categories, validation and
aggregation take `O(U * R + U * C)` arithmetic operations and retain `O(C)`
state. At most 262,144 cells are admitted. Exact rational arithmetic avoids
rounding, although numerator and denominator costs still depend on their
bounded intermediate sizes.

Units with zero or one observed rating are deliberately absent from both
observed disagreement and category totals. If no usable unit remains the input
has no pairwise evidence and is rejected. If every usable rating has one
category, expected disagreement is zero and `alpha` is `None`; negative values
are retained rather than clamped.

This is nominal alpha only. It does not assign distances between categories,
impute missing cells, estimate uncertainty, test hypotheses, diagnose rater
bias, or show that the rating construct is valid.

## Related Snippets

<!-- catalog:related:start -->
- [Compute Exact Fleiss' Kappa from a Bounded Rating-Count Matrix](compute-exact-fleiss-kappa-from-a-bounded-rating-count-matrix.md)
- [Compute Exact Unweighted Cohen's Kappa from a Confusion Matrix](compute-exact-unweighted-cohens-kappa-from-a-confusion-matrix.md)
- [Compute Exact Raw Cronbach's Alpha from a Bounded Integer Item-Score Matrix](compute-exact-raw-cronbachs-alpha-from-a-bounded-integer-item-score-matrix.md)
<!-- catalog:related:end -->
