---
title: "Compute an Exact Paired Two-Sided Sign Test"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - adjust-bounded-exact-p-values-with-holms-step-down-method.md
  - compute-a-wilson-score-interval-for-a-binomial-proportion.md
  - compute-exact-binary-roc-auc-with-tied-integer-scores.md
---

# Compute an Exact Paired Two-Sided Sign Test

## Idea and Problem

Test whether paired integer observations show a directional imbalance while preserving the finite-sample two-sided p-value as an exact fraction.

Each pair contributes only an increase, a decrease, or a tie. Ties are removed,
and under the null hypothesis the remaining signs are modeled as independent
fair coin flips. Doubling the smaller exact binomial tail gives one explicit
two-sided convention without converting large integer counts to floats.

## When to Use

Use this test for bounded before-and-after observations when direction is
meaningful but difference magnitudes are unavailable, unreliable, or
deliberately ignored. Pairs must be independent of one another, and the null
model must make an increase and a decrease equally likely among non-tied pairs.

Choose the pairs and the two-sided test before inspecting their signs. Use a
statistical library or a domain-specific analysis when magnitudes carry useful
information, observations are clustered, a one-sided or mid-p convention is
required, or the analysis also needs an effect estimate or confidence interval.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_PAIRS = 4_096
_ONE = Fraction(1, 1)


@dataclass(frozen=True, slots=True)
class PairedSignTestResult:
    increases: int
    decreases: int
    ties: int
    effective_pairs: int
    two_sided_p_value: Fraction


def _validate_signed_64_tuple(values: object, *, name: str) -> tuple[int, ...]:
    if type(values) is not tuple:
        raise TypeError(f"{name} must be an exact tuple")
    if not 1 <= len(values) <= _MAX_PAIRS:
        raise ValueError(f"{name} count is outside the supported range")
    for index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"{name}[{index}] must be an exact integer")
        if not _MIN_INT64 <= value <= _MAX_INT64:
            raise ValueError(f"{name}[{index}] is outside the signed 64-bit range")
    return values


def paired_two_sided_sign_test(
    before: tuple[int, ...],
    after: tuple[int, ...],
) -> PairedSignTestResult:
    """Return an exact doubled-smaller-tail paired sign test."""
    before = _validate_signed_64_tuple(before, name="before")
    after = _validate_signed_64_tuple(after, name="after")
    if len(before) != len(after):
        raise ValueError("before and after must contain the same number of values")

    increases = 0
    decreases = 0
    for before_value, after_value in zip(before, after, strict=True):
        if after_value > before_value:
            increases += 1
        elif after_value < before_value:
            decreases += 1

    ties = len(before) - increases - decreases
    effective_pairs = increases + decreases
    smaller_tail = min(increases, decreases)

    binomial_term = 1
    tail_sum = 1
    for index in range(1, smaller_tail + 1):
        binomial_term = binomial_term * (effective_pairs - index + 1) // index
        tail_sum += binomial_term

    doubled_tail = Fraction(2 * tail_sum, 1 << effective_pairs)
    return PairedSignTestResult(
        increases=increases,
        decreases=decreases,
        ties=ties,
        effective_pairs=effective_pairs,
        two_sided_p_value=min(_ONE, doubled_tail),
    )
```

## Example

```python
before = (10, 20, 30, 40, 50, 60, 70, 80)
after = (11, 19, 31, 41, 50, 62, 69, 82)

result = paired_two_sided_sign_test(before, after)
swapped = paired_two_sided_sign_test(after, before)
all_tied = paired_two_sided_sign_test((3, 5, 8), (3, 5, 8))

assert (result, swapped.two_sided_p_value, all_tied) == (
    PairedSignTestResult(5, 2, 1, 7, Fraction(29, 64)),
    Fraction(29, 64),
    PairedSignTestResult(0, 0, 3, 0, Fraction(1, 1)),
)
```

## Trade-offs and Limitations

For `n` pairs and `s = min(increases, decreases)`, validation and sign counting
take `O(n)` comparisons and the binomial tail takes `O(s)` recurrence steps.
The result uses `O(1)` stored counters beyond the inputs. These operation counts
treat integer arithmetic as unit cost; binomial coefficients and the final
`Fraction` can contain `O(n)` bits, so multiplication, division, and reduction
costs grow with operand size.

Discarding ties and magnitudes makes the test robust to the scale of nonzero
changes, but it can also discard substantial evidence and have less power than
a justified magnitude-aware analysis. The doubled-smaller-tail convention is
discrete and can be conservative. An all-tied sample returns p-value one as an
explicit no-evidence result.

An exact p-value is exact only for this finite binomial calculation under its
assumptions. It is not the probability that the null hypothesis is true, an
effect size, or a confidence interval. This function does not choose an alpha
level, adjust a family of tests, support one-sided or mid-p variants, apply a
Wilcoxon procedure, or analyze independent samples.

## Related Snippets

<!-- catalog:related:start -->
- [Adjust Bounded Exact P-Values with Holm's Step-Down Method](adjust-bounded-exact-p-values-with-holms-step-down-method.md)
- [Compute a Wilson Score Interval for a Binomial Proportion](compute-a-wilson-score-interval-for-a-binomial-proportion.md)
- [Compute Exact Binary ROC AUC with Tied Integer Scores](compute-exact-binary-roc-auc-with-tied-integer-scores.md)
<!-- catalog:related:end -->
