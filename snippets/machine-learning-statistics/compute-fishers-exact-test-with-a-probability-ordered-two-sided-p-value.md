---
title: "Compute Fisher's Exact Test with a Probability-Ordered Two-Sided P-Value"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-an-exact-paired-two-sided-sign-test.md
  - adjust-bounded-exact-p-values-with-holms-step-down-method.md
  - compute-a-wilson-score-interval-for-a-binomial-proportion.md
---

# Compute Fisher's Exact Test with a Probability-Ordered Two-Sided P-Value

## Idea and Problem

Compute exact tail and probability-ordered two-sided p-values for a bounded 2-by-2 count table while conditioning on both observed margins.

For a table `((a, b), (c, d))`, hold its row and column totals fixed and let
`X` be the top-left count. Under the null model of no association, every
feasible `X` has one hypergeometric probability. Enumerating that finite
support gives inclusive lower and upper tails without floating-point rounding.

This page defines the two-sided p-value by summing every feasible table whose
exact null probability is less than or equal to the observed table's
probability. That probability ordering is explicit because other two-sided
conventions can return different values for discrete data.

## When to Use

Use this calculation for a pre-specified 2-by-2 contingency table when exact
conditional inference is appropriate, the four cells are non-negative counts,
and both margins are treated as fixed. The exact `Fraction` results are useful
for reproducible validation and for later multiple-test adjustment without an
intermediate float conversion.

Use a statistical library or a designed analysis when sampling assumptions,
stratification, effect estimation, confidence intervals, larger tables,
unconditional tests, weighting, missing observations, or power calculations
also matter. Choose the tail direction and two-sided convention before looking
at the result.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction
from math import comb

_MAX_FISHER_TOTAL = 20_000


@dataclass(frozen=True, slots=True)
class FisherExactResult:
    observed_probability: Fraction
    lower_tail: Fraction
    upper_tail: Fraction
    probability_ordered_two_sided: Fraction


def fishers_exact_probabilities(
    table: tuple[tuple[int, int], tuple[int, int]],
) -> FisherExactResult:
    """Return exact fixed-margin probabilities for one 2-by-2 table."""
    if type(table) is not tuple:
        raise TypeError("table must be an exact tuple")
    if len(table) != 2:
        raise ValueError("table must contain two rows")
    for row_index, row in enumerate(table):
        if type(row) is not tuple:
            raise TypeError(f"table[{row_index}] must be an exact tuple")
        if len(row) != 2:
            raise ValueError(f"table[{row_index}] must contain two counts")
        for column_index, count in enumerate(row):
            if type(count) is not int:
                raise TypeError(f"table[{row_index}][{column_index}] must be an exact integer")
            if count < 0:
                raise ValueError(f"table[{row_index}][{column_index}] must be non-negative")

    (a, b), (c, d) = table
    total = a + b + c + d
    if not 1 <= total <= _MAX_FISHER_TOTAL:
        raise ValueError("table total is outside the supported range")

    row_one = a + b
    column_one = a + c
    column_two = b + d
    support_start = max(0, row_one - column_two)
    support_stop = min(row_one, column_one)
    denominator = comb(total, row_one)
    observed_weight = comb(column_one, a) * comb(column_two, row_one - a)
    weight = comb(column_one, support_start) * comb(column_two, row_one - support_start)

    lower_weight = 0
    upper_weight = 0
    two_sided_weight = 0

    for top_left in range(support_start, support_stop + 1):
        if top_left <= a:
            lower_weight += weight
        if top_left >= a:
            upper_weight += weight
        if weight <= observed_weight:
            two_sided_weight += weight

        if top_left < support_stop:
            numerator = weight * (column_one - top_left) * (row_one - top_left)
            divisor = (top_left + 1) * (column_two - row_one + top_left + 1)
            weight, remainder = divmod(numerator, divisor)
            if remainder:
                raise AssertionError("hypergeometric weight recurrence must be exact")

    return FisherExactResult(
        observed_probability=Fraction(observed_weight, denominator),
        lower_tail=Fraction(lower_weight, denominator),
        upper_tail=Fraction(upper_weight, denominator),
        probability_ordered_two_sided=Fraction(two_sided_weight, denominator),
    )
```

## Example

```python
ordinary = fishers_exact_probabilities(((6, 2), (1, 4)))
balanced = fishers_exact_probabilities(((1, 1), (1, 1)))
zero_margin = fishers_exact_probabilities(((0, 0), (3, 5)))
swapped_columns = fishers_exact_probabilities(((2, 6), (4, 1)))

assert (
    ordinary,
    ordinary.lower_tail + ordinary.upper_tail,
    balanced,
    zero_margin,
    swapped_columns.probability_ordered_two_sided,
) == (
    FisherExactResult(Fraction(35, 429), Fraction(427, 429), Fraction(37, 429), Fraction(4, 39)),
    1 + Fraction(35, 429),
    FisherExactResult(Fraction(2, 3), Fraction(5, 6), Fraction(5, 6), Fraction(1, 1)),
    FisherExactResult(Fraction(1, 1), Fraction(1, 1), Fraction(1, 1), Fraction(1, 1)),
    Fraction(4, 39),
)
```

## Trade-offs and Limitations

If the feasible support contains `w` top-left counts, the sweep performs
`O(w)` big-integer recurrence steps and keeps `O(1)` auxiliary integer objects.
Under the admitted total, `w` is at most 10,001. The constant number of
`math.comb` calls and each multiplication, exact division, and `Fraction`
reduction are not constant-time: operands can contain `Theta(total)` bits.

Both one-sided tails include the observed table. Consequently,
`lower_tail + upper_tail` equals one plus `observed_probability`. The two-sided
field includes all probability ties. It is not twice the smaller tail, a mid-p
value, an odds ratio, an effect size, or a confidence interval.

An exact calculation does not make the fixed-margin null model or the sampling
design appropriate. This function handles only one 2-by-2 table, does not
choose a significance threshold, and does not adjust a family of p-values.
Large or repeated analyses may be better served by established statistical
software with broader diagnostics and optimized numerical methods.

## Related Snippets

<!-- catalog:related:start -->
- [Compute an Exact Paired Two-Sided Sign Test](compute-an-exact-paired-two-sided-sign-test.md)
- [Adjust Bounded Exact P-Values with Holm's Step-Down Method](adjust-bounded-exact-p-values-with-holms-step-down-method.md)
- [Compute a Wilson Score Interval for a Binomial Proportion](compute-a-wilson-score-interval-for-a-binomial-proportion.md)
<!-- catalog:related:end -->
