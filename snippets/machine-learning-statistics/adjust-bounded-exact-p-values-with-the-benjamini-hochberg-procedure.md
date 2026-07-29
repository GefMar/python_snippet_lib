---
title: "Adjust Bounded Exact P-Values with the Benjamini-Hochberg Procedure"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - adjust-bounded-exact-p-values-with-holms-step-down-method.md
  - compute-an-exact-two-sample-permutation-test-for-a-mean-difference.md
  - compute-fishers-exact-test-with-a-probability-ordered-two-sided-p-value.md
---

# Adjust Bounded Exact P-Values with the Benjamini-Hochberg Procedure

## Idea and Problem

Convert one bounded family of exact p-values into Benjamini-Hochberg adjusted values and inclusive decisions at an explicit false-discovery-rate level.

The procedure sorts p-values, multiplies each by the family size divided by
its one-based rank, and applies a reverse cumulative minimum capped at one.
Restoring original positions keeps every adjusted value and decision aligned
with its hypothesis. `Fraction` arithmetic avoids binary floating-point drift
at the significance boundary.

## When to Use

Use this algorithm after defining one finite hypothesis family and obtaining a
valid exact p-value for every member. The advertised Benjamini-Hochberg
false-discovery-rate guarantee assumes independent p-values or positive
regression dependence on the subset belonging to true null hypotheses (PRDS).

Choose the family and alpha before inspecting results. Use a statistical
library or a procedure with an appropriate dependence correction when tests
have arbitrary dependence, p-values are approximate or missing, hypotheses
are weighted, or the analysis also needs confidence intervals and effect
sizes.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction

_MAX_P_VALUE_COUNT = 4_096
_MAX_FRACTION_COMPONENT_BITS = 256
_ZERO = Fraction(0, 1)
_ONE = Fraction(1, 1)


@dataclass(frozen=True, slots=True)
class BenjaminiHochbergAdjustment:
    adjusted_p_values: tuple[Fraction, ...]
    rejected: tuple[bool, ...]


def _bounded_probability(value: object, *, name: str) -> Fraction:
    if type(value) is not Fraction:
        raise TypeError(f"{name} must be an exact Fraction")
    if (
        value.numerator.bit_length() > _MAX_FRACTION_COMPONENT_BITS
        or value.denominator.bit_length() > _MAX_FRACTION_COMPONENT_BITS
    ):
        raise ValueError(f"{name} has an oversized fraction component")
    if not _ZERO <= value <= _ONE:
        raise ValueError(f"{name} must be between zero and one")
    return value


def adjust_benjamini_hochberg(
    p_values: tuple[Fraction, ...],
    *,
    alpha: Fraction,
) -> BenjaminiHochbergAdjustment:
    if type(p_values) is not tuple:
        raise TypeError("p_values must be an exact tuple")
    if not 1 <= len(p_values) <= _MAX_P_VALUE_COUNT:
        raise ValueError("p-value count is outside the supported range")

    checked = tuple(
        _bounded_probability(p_value, name="a p-value")
        for p_value in p_values
    )
    checked_alpha = _bounded_probability(alpha, name="alpha")
    ordered = sorted(enumerate(checked), key=lambda item: (item[1], item[0]))

    adjusted = [_ONE] * len(ordered)
    running_minimum = _ONE
    for rank in range(len(ordered), 0, -1):
        original_index, p_value = ordered[rank - 1]
        candidate = min(_ONE, p_value * len(ordered) / rank)
        running_minimum = min(running_minimum, candidate)
        adjusted[original_index] = running_minimum

    adjusted_tuple = tuple(adjusted)
    return BenjaminiHochbergAdjustment(
        adjusted_p_values=adjusted_tuple,
        rejected=tuple(value <= checked_alpha for value in adjusted_tuple),
    )
```

## Example

```python
raw = (Fraction(1, 100), Fraction(4, 100), Fraction(3, 100))
adjustment = adjust_benjamini_hochberg(raw, alpha=Fraction(35, 1_000))

tied = adjust_benjamini_hochberg(
    (Fraction(1, 20), Fraction(1, 20)),
    alpha=Fraction(1, 20),
)

try:
    adjust_benjamini_hochberg(
        (Fraction(1, 10), 0.2),  # type: ignore[arg-type]
        alpha=Fraction(1, 20),
    )
except TypeError:
    float_rejected = True
else:
    float_rejected = False

assert (
    adjustment.adjusted_p_values,
    adjustment.rejected,
    tied,
    float_rejected,
) == (
    (Fraction(3, 100), Fraction(1, 25), Fraction(1, 25)),
    (True, False, False),
    BenjaminiHochbergAdjustment(
        adjusted_p_values=(Fraction(1, 20), Fraction(1, 20)),
        rejected=(True, True),
    ),
    True,
)
```

## Trade-offs and Limitations

Sorting takes `O(m log m)` comparisons for `m` hypotheses and the reordered
state uses `O(m)` memory. Fraction operations are exact rather than
constant-cost; the family-size and component-bit limits bound their admitted
cost. The direct decision rule is inclusive: an adjusted value equal to alpha
is rejected.

Stable value-and-index sorting makes ties deterministic, while the reverse
cumulative minimum gives tied p-values consistent adjusted values. The
function does not calculate raw p-values, select a hypothesis family, estimate
effect size, or establish scientific importance. False-discovery-rate control
still depends on valid raw p-values, a predeclared family, and independence or
the stated PRDS condition; it is not an arbitrary-dependence guarantee.

## Related Snippets

<!-- catalog:related:start -->
- [Adjust Bounded Exact P-Values with Holm's Step-Down Method](adjust-bounded-exact-p-values-with-holms-step-down-method.md)
- [Compute an Exact Two-Sample Permutation Test for a Mean Difference](compute-an-exact-two-sample-permutation-test-for-a-mean-difference.md)
- [Compute Fisher's Exact Test with a Probability-Ordered Two-Sided P-Value](compute-fishers-exact-test-with-a-probability-ordered-two-sided-p-value.md)
<!-- catalog:related:end -->
