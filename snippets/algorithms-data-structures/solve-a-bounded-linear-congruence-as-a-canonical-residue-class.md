---
title: "Solve a Bounded Linear Congruence as a Canonical Residue Class"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - combine-a-bounded-system-of-possibly-non-coprime-integer-congruences.md
  - compute-a-distant-linear-recurrence-term-modulo-an-integer-by-polynomial-reduction.md
---

# Solve a Bounded Linear Congruence as a Canonical Residue Class

## Idea and Problem

Solve one bounded linear congruence as a canonical class containing every integer solution.

Let `g` be the greatest common divisor of the coefficient and modulus. A
solution exists exactly when `g` divides the right-hand side. Dividing all
three values by `g` leaves a coefficient that is invertible modulo the reduced
modulus, so Python's modular `pow` can recover the least non-negative residue.

The immutable result `(residue, period)` denotes every integer
`x = residue + k * period`. This also represents equations with several
solutions modulo the original modulus without enumerating equivalent residues.

## When to Use

Use this function for one bounded modular equation with a coefficient that is
not necessarily coprime to its modulus. The canonical class is useful before
combining constraints or when a caller needs a compact description of every
integer solution.

Use the Chinese remainder construction for a system of already normalized
congruences. Use a general Diophantine solver when several variables or
inequalities are involved. This arithmetic helper makes no cryptographic or
constant-time claim.

## Implementation

```python
from math import gcd

_MIN_LINEAR_CONGRUENCE_INT = -(1 << 63)
_MAX_LINEAR_CONGRUENCE_INT = (1 << 63) - 1
_MAX_LINEAR_CONGRUENCE_MODULUS = 1_000_000_000


def solve_linear_congruence(
    coefficient: int,
    right_hand_side: int,
    modulus: int,
) -> tuple[int, int] | None:
    """Return the canonical residue and period, or None when no solution exists."""
    if type(coefficient) is not int:
        raise TypeError("coefficient must be an exact non-boolean integer")
    if not _MIN_LINEAR_CONGRUENCE_INT <= coefficient <= _MAX_LINEAR_CONGRUENCE_INT:
        raise ValueError("coefficient is outside the signed 64-bit range")
    if type(right_hand_side) is not int:
        raise TypeError("right_hand_side must be an exact non-boolean integer")
    if not _MIN_LINEAR_CONGRUENCE_INT <= right_hand_side <= _MAX_LINEAR_CONGRUENCE_INT:
        raise ValueError("right_hand_side is outside the signed 64-bit range")
    if type(modulus) is not int:
        raise TypeError("modulus must be an exact non-boolean integer")
    if not 1 <= modulus <= _MAX_LINEAR_CONGRUENCE_MODULUS:
        raise ValueError("modulus is outside the supported range")

    common_divisor = gcd(coefficient, modulus)
    if right_hand_side % common_divisor:
        return None

    period = modulus // common_divisor
    if period == 1:
        return 0, 1

    reduced_coefficient = coefficient // common_divisor
    reduced_right_hand_side = right_hand_side // common_divisor
    inverse = pow(reduced_coefficient % period, -1, period)
    residue = (reduced_right_hand_side * inverse) % period
    return residue, period
```

## Example

```python
def solve_linear_congruence_by_search(
    coefficient: int,
    right_hand_side: int,
    modulus: int,
) -> tuple[int, int] | None:
    solutions = tuple(
        candidate
        for candidate in range(modulus)
        if (coefficient * candidate - right_hand_side) % modulus == 0
    )
    if not solutions:
        return None
    period = modulus // len(solutions)
    return solutions[0] % period, period


checked_equations = 0
for small_coefficient in range(-4, 5):
    for small_right_hand_side in range(-4, 5):
        for small_modulus in range(1, 11):
            assert solve_linear_congruence(
                small_coefficient,
                small_right_hand_side,
                small_modulus,
            ) == solve_linear_congruence_by_search(
                small_coefficient,
                small_right_hand_side,
                small_modulus,
            )
            checked_equations += 1

rejected = 0
invalid_calls = (
    lambda: solve_linear_congruence(True, 0, 1),
    lambda: solve_linear_congruence(0, False, 1),
    lambda: solve_linear_congruence(0, 0, True),
    lambda: solve_linear_congruence(_MIN_LINEAR_CONGRUENCE_INT - 1, 0, 1),
    lambda: solve_linear_congruence(0, _MAX_LINEAR_CONGRUENCE_INT + 1, 1),
    lambda: solve_linear_congruence(1, 0, 0),
    lambda: solve_linear_congruence(1, 0, _MAX_LINEAR_CONGRUENCE_MODULUS + 1),
)
for invalid_call in invalid_calls:
    try:
        invalid_call()
    except (TypeError, ValueError):
        rejected += 1

assert (
    checked_equations,
    solve_linear_congruence(6, 9, 15),
    solve_linear_congruence(-6, 9, 15),
    solve_linear_congruence(0, 14, 7),
    solve_linear_congruence(0, 1, 7),
    solve_linear_congruence(1, _MIN_LINEAR_CONGRUENCE_INT, 1_000_000_000),
    rejected,
) == (
    810,
    (4, 5),
    (1, 5),
    (0, 1),
    None,
    (_MIN_LINEAR_CONGRUENCE_INT % 1_000_000_000, 1_000_000_000),
    7,
)
```

## Trade-offs and Limitations

The greatest-common-divisor calculation and modular inverse use `O(log M)`
integer-arithmetic steps for modulus `M` and retain `O(1)` Python-integer
objects. The bit cost of those operations still grows with the operands; this
is not a constant-time routine.

The function accepts exact signed-64-bit non-Boolean coefficients and
right-hand sides, with an exact modulus from one through 1,000,000,000. A
solvable equation returns the unique residue from zero through `period - 1`.
The class `(0, 1)` denotes every integer, including all equations modulo one
and equations such as `0 * x == 0 (mod m)`.

`None` means the greatest common divisor does not divide the right-hand side.
Malformed or out-of-range inputs raise an exception instead. The function does
not enumerate residues modulo the original modulus, combine several equations,
accept non-positive moduli, solve inequalities, or provide cryptographic
guarantees.

## Related Snippets

<!-- catalog:related:start -->
- [Combine a Bounded System of Possibly Non-Coprime Integer Congruences](combine-a-bounded-system-of-possibly-non-coprime-integer-congruences.md)
- [Compute a Distant Linear-Recurrence Term Modulo an Integer by Polynomial Reduction](compute-a-distant-linear-recurrence-term-modulo-an-integer-by-polynomial-reduction.md)
<!-- catalog:related:end -->
