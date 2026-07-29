---
title: "Approximate a Bounded Fraction under a Denominator Limit with Exact Error"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-exact-rational-reduced-row-echelon-form-and-rank-for-a-bounded-integer-matrix.md
  - interpolate-a-global-polynomial-exactly-from-bounded-integer-points.md
  - evaluate-a-bounded-float-polynomial-with-fused-horner-steps.md
---

# Approximate a Bounded Fraction under a Denominator Limit with Exact Error

## Idea and Problem

Find a closest bounded-denominator rational approximation while reporting its signed and absolute errors exactly.

`Fraction.limit_denominator` uses continued fractions rather than enumerating
every possible numerator and denominator. Keeping the original and result as
`Fraction` values means the approximation error can also remain rational, with
no binary floating-point conversion or tolerance decision.

## When to Use

Use this helper when a rational value must fit a denominator budget for a
ratio, compact display, protocol field, or test fixture, and the caller needs
to inspect the exact information lost. The input must already be an exact
`Fraction`; convert measurements or decimal text under a separate explicit
policy first.

The documented API returns a closest fraction with a denominator no greater
than the limit. If every possible tie must follow a portable application rule,
implement that rule separately instead of depending on one interpreter's tie
choice.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction

_MAX_INTEGER_BITS = 4_096
_MAX_DENOMINATOR = 1_000_000


@dataclass(frozen=True, slots=True)
class FractionApproximation:
    original: Fraction
    approximation: Fraction
    signed_error: Fraction
    absolute_error: Fraction
    exact: bool


def approximate_fraction(
    value: Fraction,
    max_denominator: int,
) -> FractionApproximation:
    if type(value) is not Fraction:
        raise TypeError("value must be an exact Fraction")
    if value.numerator.bit_length() > _MAX_INTEGER_BITS:
        raise ValueError("numerator exceeds the supported bit length")
    if value.denominator.bit_length() > _MAX_INTEGER_BITS:
        raise ValueError("denominator exceeds the supported bit length")
    if type(max_denominator) is not int:
        raise TypeError("max_denominator must be an exact integer")
    if not 1 <= max_denominator <= _MAX_DENOMINATOR:
        raise ValueError("max_denominator is outside the supported range")

    approximation = value.limit_denominator(max_denominator)
    signed_error = approximation - value
    return FractionApproximation(
        original=value,
        approximation=approximation,
        signed_error=signed_error,
        absolute_error=abs(signed_error),
        exact=approximation == value,
    )
```

## Example

```python
def exhaustive_small_oracle(value: Fraction, limit: int) -> Fraction:
    candidates: set[Fraction] = set()
    for denominator in range(1, limit + 1):
        scaled_numerator = value.numerator * denominator
        lower = scaled_numerator // value.denominator
        candidates.add(Fraction(lower, denominator))
        candidates.add(Fraction(lower + 1, denominator))
    return min(
        candidates,
        key=lambda candidate: (
            abs(candidate - value),
            candidate.denominator,
            candidate,
        ),
    )


pi_like = approximate_fraction(Fraction(3_141_592, 1_000_000), 113)
assert pi_like.approximation == Fraction(355, 113)
assert pi_like.signed_error == Fraction(13, 14_125_000)
assert pi_like.absolute_error == Fraction(13, 14_125_000)
assert not pi_like.exact

already_small = approximate_fraction(Fraction(7, 12), 12)
assert already_small.exact
assert already_small.absolute_error == 0

# CPython 3.14 chooses the lower value when equally close candidates have the
# same denominator. This is verified behavior, not a portable API guarantee.
assert approximate_fraction(Fraction(1, 2), 1).approximation == 0
assert approximate_fraction(Fraction(-1, 2), 1).approximation == -1

for target in (
    Fraction(-7, 5),
    Fraction(-1, 2),
    Fraction(0),
    Fraction(1, 2),
    Fraction(22, 7),
    Fraction(355, 113),
):
    for limit in range(1, 21):
        assert approximate_fraction(
            target,
            limit,
        ).approximation == exhaustive_small_oracle(target, limit)

bit_boundary = Fraction(1 << 4_095, (1 << 4_095) + 1)
assert approximate_fraction(bit_boundary, 1).approximation == 1

try:
    approximate_fraction(Fraction(1 << 4_096), 10)
except ValueError:
    pass
else:
    raise AssertionError("a 4,097-bit numerator must be rejected")

maximum_limit = approximate_fraction(Fraction(1, 1_000_001), 1_000_000)
assert maximum_limit.approximation.denominator <= 1_000_000
```

## Trade-offs and Limitations

Continued-fraction iteration is linear in the continued-fraction length, which
is bounded by the input integer bit lengths. The cost of each big-integer
operation is implementation-dependent; it is not constant merely because the
number of continued-fraction steps is bounded.

The documented `Fraction` contract promises a closest approximation under the
denominator limit. CPython 3.14's implementation resolves equal distances by
choosing the smaller denominator, then the lower value when equal denominators
remain. The Example verifies that behavior explicitly but does not present it
as a portable guarantee for every Python implementation or future version.

The helper limits representation size rather than application error. A small
denominator can still create an unacceptable absolute or relative error, and
the caller must decide that policy from the returned exact values. It does not
convert floats or decimals, choose a denominator limit, enumerate alternate
ties, or format the fraction for presentation.

## Related Snippets

<!-- catalog:related:start -->
- [Compute Exact Rational Reduced Row-Echelon Form and Rank for a Bounded Integer Matrix](compute-exact-rational-reduced-row-echelon-form-and-rank-for-a-bounded-integer-matrix.md)
- [Interpolate a Global Polynomial Exactly from Bounded Integer Points](interpolate-a-global-polynomial-exactly-from-bounded-integer-points.md)
- [Evaluate a Bounded Float Polynomial with Fused Horner Steps](evaluate-a-bounded-float-polynomial-with-fused-horner-steps.md)
<!-- catalog:related:end -->
