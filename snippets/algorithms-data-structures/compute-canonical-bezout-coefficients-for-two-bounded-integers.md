---
title: "Compute Canonical Bézout Coefficients for Two Bounded Integers"
snippet_type: algorithm
use_cases:
  - data-transformation
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - solve-a-bounded-linear-congruence-as-a-canonical-residue-class.md
  - combine-a-bounded-system-of-possibly-non-coprime-integer-congruences.md
  - factor-a-bounded-positive-integer-by-deterministic-trial-division.md
---

# Compute Canonical Bézout Coefficients for Two Bounded Integers

## Idea and Problem

Return a reproducible exact certificate that the greatest common divisor of two bounded signed integers is their integer linear combination.

Extended Euclid tracks how each remainder is assembled from the original
values. Its first coefficient is not unique: adding a multiple of the second
value divided by the gcd produces another valid identity. Reducing that
coefficient to one explicit non-negative residue interval chooses one
canonical solution, after which the second coefficient is determined exactly.

## When to Use

Use this function when a gcd calculation also needs inspectable coefficients,
for example to prove divisibility, derive a modular inverse, or verify a small
integer transformation. The canonical range makes results stable across
equivalent coefficient families and input signs.

Use `math.gcd()` when only the divisor is needed. Use a dedicated congruence
solver when the desired output is a residue class rather than a Bézout
certificate, and use a linear-algebra or number-theory library for systems
with more than two variables.

## Implementation

```python
from dataclasses import dataclass

_MAX_BEZOUT_INTEGER_BITS = 4_096


@dataclass(frozen=True, slots=True)
class BezoutIdentity:
    gcd: int
    first_coefficient: int
    second_coefficient: int


def canonical_bezout_coefficients(first: int, second: int) -> BezoutIdentity:
    """Return the canonical exact identity first*x + second*y == gcd."""
    if type(first) is not int:
        raise TypeError("first must be an exact integer")
    if type(second) is not int:
        raise TypeError("second must be an exact integer")
    if abs(first).bit_length() > _MAX_BEZOUT_INTEGER_BITS:
        raise ValueError("first exceeds the supported bit length")
    if abs(second).bit_length() > _MAX_BEZOUT_INTEGER_BITS:
        raise ValueError("second exceeds the supported bit length")

    if first == 0 and second == 0:
        return BezoutIdentity(0, 0, 0)
    if second == 0:
        return BezoutIdentity(abs(first), 1 if first > 0 else -1, 0)

    previous_remainder = abs(first)
    remainder = abs(second)
    previous_first_coefficient = 1
    first_coefficient = 0

    while remainder:
        quotient, next_remainder = divmod(previous_remainder, remainder)
        previous_remainder, remainder = remainder, next_remainder
        previous_first_coefficient, first_coefficient = (
            first_coefficient,
            previous_first_coefficient - quotient * first_coefficient,
        )

    gcd = previous_remainder
    signed_first_coefficient = (
        previous_first_coefficient if first >= 0 else -previous_first_coefficient
    )
    coefficient_modulus = abs(second) // gcd
    canonical_first = signed_first_coefficient % coefficient_modulus
    numerator = gcd - first * canonical_first
    canonical_second, remainder = divmod(numerator, second)
    if remainder:
        raise AssertionError("canonical coefficient reduction must remain exact")

    return BezoutIdentity(gcd, canonical_first, canonical_second)
```

## Example

```python
def canonical_bezout_by_search(first: int, second: int) -> BezoutIdentity:
    from math import gcd

    divisor = gcd(first, second)
    if first == 0 and second == 0:
        return BezoutIdentity(0, 0, 0)
    if second == 0:
        return BezoutIdentity(divisor, 1 if first > 0 else -1, 0)

    coefficient_modulus = abs(second) // divisor
    for first_coefficient in range(coefficient_modulus):
        numerator = divisor - first * first_coefficient
        second_coefficient, remainder = divmod(numerator, second)
        if remainder == 0:
            return BezoutIdentity(
                divisor,
                first_coefficient,
                second_coefficient,
            )
    raise AssertionError("one canonical Bézout coefficient must exist")


def exercise_small_signed_pairs() -> int:
    checked = 0
    for first in range(-96, 97):
        for second in range(-96, 97):
            assert canonical_bezout_coefficients(
                first,
                second,
            ) == canonical_bezout_by_search(first, second)
            checked += 1
    return checked


def exercise_signed_boundaries() -> int:
    from math import gcd

    maximum = (1 << _MAX_BEZOUT_INTEGER_BITS) - 1
    cases = (
        (maximum, maximum - 2),
        (-maximum, 1 << (_MAX_BEZOUT_INTEGER_BITS - 1)),
        (1 << (_MAX_BEZOUT_INTEGER_BITS - 1), -maximum),
        (-maximum, -maximum),
    )
    for first, second in cases:
        identity = canonical_bezout_coefficients(first, second)
        assert identity.gcd == gcd(first, second)
        assert (
            first * identity.first_coefficient
            + second * identity.second_coefficient
            == identity.gcd
        )
        assert 0 <= identity.first_coefficient < abs(second) // identity.gcd
    return len(cases)


rejected = 0
for invalid_first, invalid_second in (
    (True, 1),
    (1, False),
    (1 << _MAX_BEZOUT_INTEGER_BITS, 1),
    (1, -(1 << _MAX_BEZOUT_INTEGER_BITS)),
):
    try:
        canonical_bezout_coefficients(invalid_first, invalid_second)
    except (TypeError, ValueError):
        rejected += 1

assert (
    exercise_small_signed_pairs(),
    exercise_signed_boundaries(),
    canonical_bezout_coefficients(240, 46),
    canonical_bezout_coefficients(-240, 46),
    canonical_bezout_coefficients(0, 0),
    canonical_bezout_coefficients(-7, 0),
    rejected,
) == (
    37_249,
    4,
    BezoutIdentity(2, 14, -73),
    BezoutIdentity(2, 9, 47),
    BezoutIdentity(0, 0, 0),
    BezoutIdentity(7, -1, 0),
    4,
)
```

## Trade-offs and Limitations

For non-zero operands, extended Euclid takes logarithmically many exact
division steps in the smaller magnitude. Division and coefficient arithmetic
have operand-dependent big-integer cost under the 4,096-bit input cap. The
algorithm retains only a constant number of Python integers, but their storage
is proportional to their bit lengths.

When the second value is non-zero, the first coefficient is the unique value
in `0 <= x < abs(second) // gcd`; this can make the derived second coefficient
larger than another valid, non-canonical choice. The special zero cases keep
the gcd non-negative and avoid an undefined residue interval.

The function does not enumerate the infinite coefficient family, solve a
multi-variable Diophantine system, return a congruence solution directly, or
promise constant-time behavior suitable for cryptographic side-channel
requirements.

## Related Snippets

<!-- catalog:related:start -->
- [Solve a Bounded Linear Congruence as a Canonical Residue Class](solve-a-bounded-linear-congruence-as-a-canonical-residue-class.md)
- [Combine a Bounded System of Possibly Non-Coprime Integer Congruences](combine-a-bounded-system-of-possibly-non-coprime-integer-congruences.md)
- [Factor a Bounded Positive Integer by Deterministic Trial Division](factor-a-bounded-positive-integer-by-deterministic-trial-division.md)
<!-- catalog:related:end -->
