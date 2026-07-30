---
title: "Report Exact Binary Matthews-Correlation Evidence from Four Confusion Counts"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - build-an-exact-bounded-multiclass-confusion-matrix.md
  - compute-exact-squared-pearson-correlation-with-direction.md
  - compute-exact-unweighted-cohens-kappa-from-a-confusion-matrix.md
---

# Report Exact Binary Matthews-Correlation Evidence from Four Confusion Counts

## Idea and Problem

Measure binary prediction association from four confusion counts while preserving exact sign, magnitude evidence, and undefined cases.

The Matthews correlation coefficient divides `TP*TN - FP*FN` by the square
root of four class and prediction marginals. The square root is usually
irrational, so the result records the numerator, denominator radicand, and
signed squared coefficient exactly. A float is derived only as a convenient
view.

For an admitted nonempty table, when any required marginal is zero, the
denominator vanishes. Returning an explicit undefined result avoids silently
replacing a mathematically missing coefficient with a policy value. An
all-zero table is rejected because it contains no observations.

## When to Use

Use this calculation after a binary threshold and positive-label convention
have already produced exact aggregate `TP`, `TN`, `FP`, and `FN` counts. It is
useful when both classes matter and one symmetric association measure is more
informative than accuracy alone on an imbalanced table.

Use precision, recall, or a cost-weighted measure when false positives and
false negatives have different operational meanings. Use a statistics library
when raw labels, sample weights, multiclass behavior, confidence intervals,
hypothesis tests, threshold selection, or a library-specific zero-denominator
policy is required.

## Implementation

```python
import math
from dataclasses import dataclass
from enum import Enum
from fractions import Fraction

_MAX_TOTAL = (1 << 63) - 1


class MatthewsDirection(Enum):
    NEGATIVE = "negative"
    ZERO = "zero"
    POSITIVE = "positive"
    UNDEFINED = "undefined"


@dataclass(frozen=True, slots=True)
class MatthewsEvidence:
    total: int
    numerator: int
    denominator_radicand: int
    signed_squared_coefficient: Fraction | None
    coefficient: float | None
    direction: MatthewsDirection


def binary_matthews_evidence(
    true_positive: int,
    true_negative: int,
    false_positive: int,
    false_negative: int,
) -> MatthewsEvidence:
    """Return exact evidence and a float view of binary MCC."""
    named_counts = (
        ("true_positive", true_positive),
        ("true_negative", true_negative),
        ("false_positive", false_positive),
        ("false_negative", false_negative),
    )
    for name, count in named_counts:
        if type(count) is not int:
            raise TypeError(f"{name} must be an exact integer")
        if not 0 <= count <= _MAX_TOTAL:
            raise ValueError(f"{name} is outside the supported range")

    total = sum(count for _, count in named_counts)
    if not 1 <= total <= _MAX_TOTAL:
        raise ValueError("the aggregate count is outside the supported range")

    numerator = true_positive * true_negative - false_positive * false_negative
    denominator_radicand = (
        (true_positive + false_positive)
        * (true_positive + false_negative)
        * (true_negative + false_positive)
        * (true_negative + false_negative)
    )
    if denominator_radicand == 0:
        return MatthewsEvidence(
            total=total,
            numerator=numerator,
            denominator_radicand=0,
            signed_squared_coefficient=None,
            coefficient=None,
            direction=MatthewsDirection.UNDEFINED,
        )

    signed_square = Fraction(
        numerator * abs(numerator),
        denominator_radicand,
    )
    if numerator < 0:
        direction = MatthewsDirection.NEGATIVE
    elif numerator > 0:
        direction = MatthewsDirection.POSITIVE
    else:
        direction = MatthewsDirection.ZERO

    magnitude = math.sqrt(float(abs(signed_square)))
    coefficient = -magnitude if numerator < 0 else magnitude
    return MatthewsEvidence(
        total=total,
        numerator=numerator,
        denominator_radicand=denominator_radicand,
        signed_squared_coefficient=signed_square,
        coefficient=coefficient,
        direction=direction,
    )
```

## Example

```python


measured = binary_matthews_evidence(30, 50, 10, 10)
perfect = binary_matthews_evidence(3, 4, 0, 0)
inverse = binary_matthews_evidence(0, 0, 3, 4)
independent = binary_matthews_evidence(1, 1, 1, 1)
undefined = binary_matthews_evidence(5, 0, 0, 0)

assert (
    measured.total,
    measured.numerator,
    measured.denominator_radicand,
    measured.signed_squared_coefficient,
    measured.direction,
) == (100, 1_400, 5_760_000, Fraction(49, 144), MatthewsDirection.POSITIVE)
assert math.isclose(measured.coefficient or 0.0, 7 / 12)
assert (
    perfect.signed_squared_coefficient,
    perfect.coefficient,
    inverse.signed_squared_coefficient,
    inverse.coefficient,
    independent.signed_squared_coefficient,
) == (Fraction(1), 1.0, Fraction(-1), -1.0, Fraction(0))
assert (
    undefined.denominator_radicand,
    undefined.signed_squared_coefficient,
    undefined.coefficient,
    undefined.direction,
) == (0, None, None, MatthewsDirection.UNDEFINED)
```

## Trade-offs and Limitations

Validation and arithmetic use `O(1)` stored values, but Python integer and
`Fraction` operations are not constant-cost with respect to operand bit
length. The aggregate signed-64-bit cap keeps conversion of the exact signed
square to a finite binary64 value; the float is rounded, while the numerator,
radicand, and signed square remain exact.

Binary MCC is the Pearson correlation, also called the phi coefficient, of two
binary indicator variables. This direct confusion-count boundary avoids
materializing raw labels but does not define a new statistic. Swapping both
label meanings preserves the coefficient; swapping only one side reverses its
sign.

An undefined result means at least one observed or predicted class has no
variation. It is not evidence of zero association. The function does not
apply smoothing, select a threshold, infer causality, compare models, quantify
uncertainty, or decide whether a coefficient is operationally sufficient.

## Related Snippets

<!-- catalog:related:start -->
- [Build an Exact Bounded Multiclass Confusion Matrix](build-an-exact-bounded-multiclass-confusion-matrix.md)
- [Compute Exact Squared Pearson Correlation with Direction](compute-exact-squared-pearson-correlation-with-direction.md)
- [Compute Exact Unweighted Cohen's Kappa from a Confusion Matrix](compute-exact-unweighted-cohens-kappa-from-a-confusion-matrix.md)
<!-- catalog:related:end -->
