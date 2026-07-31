---
title: "Reconstruct a Unique Bounded Rational from One Modular Residue"
snippet_type: algorithm
use_cases:
  - data-transformation
  - interoperability
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - approximate-a-bounded-fraction-under-a-denominator-limit-with-exact-error.md
  - compute-canonical-bezout-coefficients-for-two-bounded-integers.md
  - solve-a-bounded-linear-congruence-as-a-canonical-residue-class.md
---

# Reconstruct a Unique Bounded Rational from One Modular Residue

## Idea and Problem

Recover the only sufficiently small reduced rational number that can represent one residue modulo a bounded integer.

For a residue `r` modulo `m`, the desired fraction `n / d` must satisfy
`n == r * d (mod m)`. Extended Euclid generates progressively smaller
remainders together with their coefficients of `r`. The first remainder inside
the numerator bound supplies the only possible candidate denominator.

The strict precondition `2 * numerator_bound * denominator_bound < modulus`
makes two different admitted reduced fractions impossible: their cross-product
difference would be a nonzero multiple of the modulus with smaller absolute
value. The implementation still verifies every candidate condition instead of
assuming that each residue has a reconstruction.

## When to Use

Use rational reconstruction after an exact calculation modulo one sufficiently
large integer when the true answer is known in advance to have a small signed
numerator and a small positive denominator. It is useful for lifting modular
intermediate results without introducing floating-point approximation.

Choose the bounds from independent knowledge of the problem, and require the
denominator to be invertible modulo the selected modulus. Use ordinary
`Fraction` construction when numerator and denominator are already available,
or bounded-denominator approximation when starting from an approximate value.

## Implementation

```python
from fractions import Fraction
from math import gcd

_MAX_MODULUS = (1 << 63) - 1
_MAX_RATIONAL_BOUND = (1 << 31) - 1


def reconstruct_bounded_rational(
    residue: int,
    modulus: int,
    numerator_bound: int,
    denominator_bound: int,
) -> Fraction | None:
    """Return the unique admitted rational congruent to residue, if it exists."""
    for name, value in (
        ("residue", residue),
        ("modulus", modulus),
        ("numerator_bound", numerator_bound),
        ("denominator_bound", denominator_bound),
    ):
        if type(value) is not int:
            raise TypeError(f"{name} must be an exact integer")

    if not 2 <= modulus <= _MAX_MODULUS:
        raise ValueError("modulus is outside 2..2^63-1")
    if not 0 <= residue < modulus:
        raise ValueError("residue is not canonical for modulus")
    if not 0 <= numerator_bound <= _MAX_RATIONAL_BOUND:
        raise ValueError("numerator_bound is outside 0..2^31-1")
    if not 1 <= denominator_bound <= _MAX_RATIONAL_BOUND:
        raise ValueError("denominator_bound is outside 1..2^31-1")
    if 2 * numerator_bound * denominator_bound >= modulus:
        raise ValueError("bounds do not guarantee a unique reconstruction")

    previous_remainder, remainder = modulus, residue
    previous_denominator, denominator = 0, 1
    while remainder > numerator_bound:
        quotient = previous_remainder // remainder
        previous_remainder, remainder = (
            remainder,
            previous_remainder - quotient * remainder,
        )
        previous_denominator, denominator = (
            denominator,
            previous_denominator - quotient * denominator,
        )

    numerator = remainder
    if denominator < 0:
        numerator = -numerator
        denominator = -denominator

    if not (
        abs(numerator) <= numerator_bound
        and 1 <= denominator <= denominator_bound
        and gcd(abs(numerator), denominator) == 1
        and gcd(denominator, modulus) == 1
        and (residue * denominator - numerator) % modulus == 0
    ):
        return None
    return Fraction(numerator, denominator)
```

## Example

```python
def enumerated_reconstructions(
    modulus: int,
    numerator_bound: int,
    denominator_bound: int,
) -> dict[int, Fraction]:
    found: dict[int, Fraction] = {}
    for denominator in range(1, denominator_bound + 1):
        if gcd(denominator, modulus) != 1:
            continue
        inverse = pow(denominator, -1, modulus)
        for numerator in range(-numerator_bound, numerator_bound + 1):
            if gcd(abs(numerator), denominator) != 1:
                continue
            residue = numerator * inverse % modulus
            candidate = Fraction(numerator, denominator)
            assert residue not in found or found[residue] == candidate
            found[residue] = candidate
    return found


exhaustive_cases = 0
for small_modulus in range(2, 128):
    for small_numerator_bound in range(9):
        for small_denominator_bound in range(1, 9):
            if 2 * small_numerator_bound * small_denominator_bound >= small_modulus:
                continue
            expected_by_residue = enumerated_reconstructions(
                small_modulus,
                small_numerator_bound,
                small_denominator_bound,
            )
            for small_residue in range(small_modulus):
                assert reconstruct_bounded_rational(
                    small_residue,
                    small_modulus,
                    small_numerator_bound,
                    small_denominator_bound,
                ) == expected_by_residue.get(small_residue)
                exhaustive_cases += 1

large_modulus = (1 << 63) - 25
large_expected = Fraction(-12_345, 6_789)
large_residue = (
    large_expected.numerator * pow(large_expected.denominator, -1, large_modulus) % large_modulus
)
large_result = reconstruct_bounded_rational(
    large_residue,
    large_modulus,
    abs(large_expected.numerator),
    large_expected.denominator,
)

try:
    reconstruct_bounded_rational(1, 12, 2, 3)
except ValueError:
    strict_boundary_rejected = True
else:
    strict_boundary_rejected = False

assert (
    reconstruct_bounded_rational(8, 17, 1, 2),
    reconstruct_bounded_rational(0, 97, 0, 12),
    large_result,
    strict_boundary_rejected,
    exhaustive_cases,
) == (Fraction(-1, 2), Fraction(0, 1), large_expected, True, 500_808)
```

## Trade-offs and Limitations

Extended Euclid performs `O(log m)` divisions in the worst case and retains
`O(1)` integers. Those integers have up to the modulus bit length, so Python's
arithmetic cost is not constant even though the number of live values is.

`None` means no fraction satisfies all declared conditions; it does not mean
that the residue lacks larger rational representatives. The strict inequality
guarantees uniqueness only inside the supplied rectangle of numerator and
denominator bounds. The function deliberately rejects wider bounds instead of
choosing among possible answers.

This is exact reconstruction from one canonical residue. It does not combine
several moduli, tolerate corrupt or approximate residues, infer suitable
bounds, perform lattice reduction, or prove that a recovered value is the
intended result of an external computation.

## Related Snippets

<!-- catalog:related:start -->
- [Approximate a Bounded Fraction under a Denominator Limit with Exact Error](approximate-a-bounded-fraction-under-a-denominator-limit-with-exact-error.md)
- [Compute Canonical Bézout Coefficients for Two Bounded Integers](compute-canonical-bezout-coefficients-for-two-bounded-integers.md)
- [Solve a Bounded Linear Congruence as a Canonical Residue Class](solve-a-bounded-linear-congruence-as-a-canonical-residue-class.md)
<!-- catalog:related:end -->
