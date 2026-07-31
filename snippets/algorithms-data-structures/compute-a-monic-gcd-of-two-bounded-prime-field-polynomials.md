---
title: "Compute a Monic GCD of Two Bounded Prime-Field Polynomials"
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
  - convolve-bounded-integer-polynomials-modulo-998244353-with-an-iterative-ntt.md
  - compute-a-distant-linear-recurrence-term-modulo-an-integer-by-polynomial-reduction.md
  - find-a-shortest-linear-recurrence-fitting-a-bounded-prime-field-sequence-with-berlekamp-massey.md
---

# Compute a Monic GCD of Two Bounded Prime-Field Polynomials

## Idea and Problem

Find one canonical greatest common divisor of two bounded univariate polynomials over an explicitly validated prime field.

Polynomial Euclid repeatedly replaces a pair with the divisor and the
remainder from long division. Because every nonzero coefficient has an inverse
modulo a prime, each leading term can be cancelled exactly. Scaling the final
nonzero remainder by the inverse of its leading coefficient makes the answer
monic, eliminating the otherwise arbitrary nonzero constant factor.

Coefficients are stored constant-first as canonical residues. The zero
polynomial is exactly `(0,)`, and every other tuple ends in a nonzero leading
coefficient. These representation rules make equality and the returned GCD
unambiguous.

## When to Use

Use this operation to detect or remove common polynomial factors when all
coefficients belong to one small prime field. It is useful in finite-field
recurrences, algebraic validation, and preprocessing before modular polynomial
operations.

The caller must already know the field modulus and must represent dense
polynomials of degree at most 256. Use a specialized computer-algebra library
for large degrees, sparse multivariate inputs, factorization, or coefficient
domains other than a prime field.

## Implementation

```python
from math import isqrt

_MAX_POLYNOMIAL_PRIME = 65_521
_MAX_POLYNOMIAL_COEFFICIENTS = 257
_ZERO_POLYNOMIAL = (0,)


def _validate_polynomial_prime(prime: int) -> None:
    if type(prime) is not int:
        raise TypeError("prime must be an exact integer")
    if not 2 <= prime <= _MAX_POLYNOMIAL_PRIME:
        raise ValueError("prime is outside 2..65,521")
    if prime % 2 == 0:
        if prime != 2:
            raise ValueError("prime must be prime")
        return
    for divisor in range(3, isqrt(prime) + 1, 2):
        if prime % divisor == 0:
            raise ValueError("prime must be prime")


def _validate_field_polynomial(
    polynomial: tuple[int, ...],
    prime: int,
    name: str,
) -> None:
    if type(polynomial) is not tuple:
        raise TypeError(f"{name} must be an exact tuple")
    if not 1 <= len(polynomial) <= _MAX_POLYNOMIAL_COEFFICIENTS:
        raise ValueError(f"{name} must contain 1..257 coefficients")
    if any(type(coefficient) is not int for coefficient in polynomial):
        raise TypeError(f"every {name} coefficient must be an exact integer")
    if any(not 0 <= coefficient < prime for coefficient in polynomial):
        raise ValueError(f"every {name} coefficient must be a canonical residue")
    if len(polynomial) > 1 and polynomial[-1] == 0:
        raise ValueError(f"{name} has a trailing zero coefficient")


def _polynomial_remainder(
    dividend: tuple[int, ...],
    divisor: tuple[int, ...],
    prime: int,
) -> tuple[int, ...]:
    remainder = list(dividend)
    inverse_leading = pow(divisor[-1], -1, prime)

    while remainder != [0] and len(remainder) >= len(divisor):
        shift = len(remainder) - len(divisor)
        factor = remainder[-1] * inverse_leading % prime
        for index, coefficient in enumerate(divisor):
            target = shift + index
            remainder[target] = (remainder[target] - factor * coefficient) % prime
        while len(remainder) > 1 and remainder[-1] == 0:
            remainder.pop()
    return tuple(remainder)


def monic_polynomial_gcd(
    left: tuple[int, ...],
    right: tuple[int, ...],
    prime: int,
) -> tuple[int, ...]:
    """Return the canonical monic GCD over the field with order prime."""
    _validate_polynomial_prime(prime)
    _validate_field_polynomial(left, prime, "left")
    _validate_field_polynomial(right, prime, "right")
    if left == _ZERO_POLYNOMIAL and right == _ZERO_POLYNOMIAL:
        raise ValueError("the GCD of two zero polynomials is undefined")

    while right != _ZERO_POLYNOMIAL:
        left, right = right, _polynomial_remainder(left, right, prime)

    inverse_leading = pow(left[-1], -1, prime)
    return tuple(coefficient * inverse_leading % prime for coefficient in left)
```

## Example

```python
from itertools import product


def multiply_polynomials(
    left: tuple[int, ...],
    right: tuple[int, ...],
    prime: int,
) -> tuple[int, ...]:
    if left == (0,) or right == (0,):
        return (0,)
    result = [0] * (len(left) + len(right) - 1)
    for left_index, left_value in enumerate(left):
        for right_index, right_value in enumerate(right):
            result[left_index + right_index] = (
                result[left_index + right_index] + left_value * right_value
            ) % prime
    while len(result) > 1 and result[-1] == 0:
        result.pop()
    return tuple(result)


gf2_polynomials = [(0,)]
for degree in range(7):
    gf2_polynomials.extend((*lower, 1) for lower in product(range(2), repeat=degree))


def oracle_divisors(polynomial: tuple[int, ...]) -> frozenset[tuple[int, ...]]:
    found: set[tuple[int, ...]] = set()
    for divisor in gf2_polynomials[1:]:
        for quotient in gf2_polynomials:
            if multiply_polynomials(divisor, quotient, 2) == polynomial:
                found.add(divisor)
                break
    return frozenset(found)


divisors_by_polynomial = {polynomial: oracle_divisors(polynomial) for polynomial in gf2_polynomials}
examined_pairs = 0
for first in gf2_polynomials:
    for second in gf2_polynomials:
        if first == second == (0,):
            continue
        common = divisors_by_polynomial[first] & divisors_by_polynomial[second]
        expected = max(common, key=len)
        assert monic_polynomial_gcd(first, second, 2) == expected
        assert monic_polynomial_gcd(second, first, 2) == expected
        examined_pairs += 1

shared = (2, 1)
left_factor = multiply_polynomials(shared, (1, 1, 1), 5)
right_factor = multiply_polynomials(shared, (3, 1), 5)
scaled_left = tuple(coefficient * 3 % 5 for coefficient in left_factor)
shared_over_three = (1, 1)
left_over_three = multiply_polynomials(shared_over_three, (2, 1), 3)
right_over_three = multiply_polynomials(shared_over_three, (1, 0, 1), 3)
shared_over_257 = (5, 0, 1)
left_over_257 = multiply_polynomials(shared_over_257, (3, 1), 257)
right_over_257 = multiply_polynomials(shared_over_257, (7, 1), 257)
maximum_degree = (1,) + (0,) * 255 + (1,)


def rejects(left: object, right: object, prime: object) -> bool:
    try:
        monic_polynomial_gcd(left, right, prime)  # type: ignore[arg-type]
    except (TypeError, ValueError):
        return True
    return False


assert (
    examined_pairs,
    monic_polynomial_gcd(left_factor, right_factor, 5),
    monic_polynomial_gcd(scaled_left, right_factor, 5),
    monic_polynomial_gcd(left_over_three, right_over_three, 3),
    monic_polynomial_gcd(left_over_257, right_over_257, 257),
    monic_polynomial_gcd((0,), (6, 3), 7),
    monic_polynomial_gcd(maximum_degree, maximum_degree, 257),
    rejects((0,), (0,), 2),
    rejects((1,), (1,), 15),
    rejects((1, True), (1,), 2),
    rejects((1, 0), (1,), 2),
    rejects((1,) * 258, (1,), 2),
) == (
    16_383,
    shared,
    shared,
    shared_over_three,
    shared_over_257,
    (2, 1),
    maximum_degree,
    True,
    True,
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

For maximum input degree `D`, dense Euclidean division uses `O(D**2)` field
operations in the worst case. Trial division validates the modulus in
`O(sqrt(prime))` bounded integer steps, and up to `O(D log prime)` modular work
is spent finding leading-coefficient inverses. The live coefficient state is
`O(D)`. Python integer multiplication and remainder costs are not constant in
the operand bit length.

Rejecting noncanonical coefficients and trailing zeros is intentional: silent
normalization could conceal a field mismatch or a malformed serialized value.
The `(0,), (0,)` pair has no distinguished monic GCD and is therefore rejected;
one zero input still returns the monic normalization of the other polynomial.

This computes only a GCD. It does not return Bézout coefficients, factor the
inputs, find roots, handle multivariate expressions, or define division over
composite coefficient rings where leading coefficients may not be invertible.

## Related Snippets

<!-- catalog:related:start -->
- [Convolve Bounded Integer Polynomials Modulo 998244353 with an Iterative NTT](convolve-bounded-integer-polynomials-modulo-998244353-with-an-iterative-ntt.md)
- [Compute a Distant Linear-Recurrence Term Modulo an Integer by Polynomial Reduction](compute-a-distant-linear-recurrence-term-modulo-an-integer-by-polynomial-reduction.md)
- [Find a Shortest Linear Recurrence Fitting a Bounded Prime-Field Sequence with Berlekamp-Massey](find-a-shortest-linear-recurrence-fitting-a-bounded-prime-field-sequence-with-berlekamp-massey.md)
<!-- catalog:related:end -->
