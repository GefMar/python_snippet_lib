---
title: "Adjust Bounded Exact P-Values with Holm's Step-Down Method"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-a-wilson-score-interval-for-a-binomial-proportion.md
  - score-feature-importances-against-bounded-null-runs.md
  - calculate-a-symmetrically-trimmed-mean.md
---

# Adjust Bounded Exact P-Values with Holm's Step-Down Method

## Idea and Problem

Convert a bounded family of exact p-values into Holm-adjusted values that can be compared with one family-wise significance level.

Sort the raw values from smallest to largest, multiply each by the number of
hypotheses remaining at that position, cap at one, and take a cumulative
maximum. The cumulative maximum preserves the method's step-down ordering;
restoring original positions keeps each adjusted value aligned with its input.

## When to Use

Use this algorithm after defining one finite hypothesis family and obtaining a
valid exact `Fraction` p-value for every member. Holm's method controls the
family-wise error rate without requiring independence among the tests, and the
adjusted values can be compared directly with a chosen significance level.

Choose the family before inspecting outcomes. Use a statistical library when
raw p-values, missing-value policy, weighted hypotheses, confidence intervals,
or a different multiplicity procedure must be calculated as part of the same
analysis.

## Implementation

```python
from fractions import Fraction

_MAX_P_VALUE_COUNT = 10_000
_MAX_FRACTION_COMPONENT_BITS = 256
_ZERO = Fraction(0, 1)
_ONE = Fraction(1, 1)


def holm_adjusted_p_values(
    p_values: tuple[Fraction, ...],
) -> tuple[Fraction, ...]:
    """Return exact Holm-adjusted p-values in original input order."""
    if type(p_values) is not tuple:
        raise TypeError("p_values must be an exact tuple")
    if not 1 <= len(p_values) <= _MAX_P_VALUE_COUNT:
        raise ValueError("p-value count is outside the supported range")

    checked: list[Fraction] = []
    for p_value in p_values:
        if type(p_value) is not Fraction:
            raise TypeError("p_values must contain exact Fraction values")
        if (
            p_value.numerator.bit_length() > _MAX_FRACTION_COMPONENT_BITS
            or p_value.denominator.bit_length() > _MAX_FRACTION_COMPONENT_BITS
        ):
            raise ValueError("a p-value fraction component is too large")
        if not _ZERO <= p_value <= _ONE:
            raise ValueError("p-values must be between zero and one")
        checked.append(p_value)

    ordered = sorted(enumerate(checked), key=lambda item: (item[1], item[0]))
    adjusted = [_ZERO] * len(ordered)
    running_maximum = _ZERO

    for sorted_index, (original_index, p_value) in enumerate(ordered):
        remaining = len(ordered) - sorted_index
        candidate = min(_ONE, p_value * remaining)
        running_maximum = max(running_maximum, candidate)
        adjusted[original_index] = running_maximum

    return tuple(adjusted)
```

## Example

```python
raw = (Fraction(1, 100), Fraction(4, 100), Fraction(3, 100))
adjusted = holm_adjusted_p_values(raw)

try:
    holm_adjusted_p_values((Fraction(1, 10), 0.2))  # type: ignore[arg-type]
except TypeError:
    float_rejected = True
else:
    float_rejected = False

assert (adjusted, float_rejected) == (
    (Fraction(3, 100), Fraction(6, 100), Fraction(6, 100)),
    True,
)
```

## Trade-offs and Limitations

Sorting takes `O(m log m)` comparisons for `m` hypotheses; the reordered and
restored values use `O(m)` memory. Fraction arithmetic is exact rather than
constant-cost: its work depends on numerator and denominator sizes. Input
component bit limits and the family-size cap bound that cost, while the
rank-scaled intermediate values remain small extensions of admitted inputs.

Stable value-and-position sorting makes tied inputs deterministic. The function
returns adjusted values only; it does not choose an alpha level, accept or
reject hypotheses, compute raw p-values, estimate effect sizes, or interpret
scientific importance. Family-wise error control still depends on valid raw
p-values and a family defined independently of the observed results.

## Related Snippets

<!-- catalog:related:start -->
- [Compute a Wilson Score Interval for a Binomial Proportion](compute-a-wilson-score-interval-for-a-binomial-proportion.md)
- [Score Feature Importances Against Bounded Null Runs](score-feature-importances-against-bounded-null-runs.md)
- [Calculate a Symmetrically Trimmed Mean](calculate-a-symmetrically-trimmed-mean.md)
<!-- catalog:related:end -->
