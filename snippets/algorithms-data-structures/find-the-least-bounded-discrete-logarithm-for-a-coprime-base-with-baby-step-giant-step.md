---
title: "Find the Least Bounded Discrete Logarithm for a Coprime Base with Baby-Step Giant-Step"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - solve-a-bounded-linear-congruence-as-a-canonical-residue-class.md
  - compute-canonical-bezout-coefficients-for-two-bounded-integers.md
  - combine-a-bounded-system-of-possibly-non-coprime-integer-congruences.md
---

# Find the Least Bounded Discrete Logarithm for a Coprime Base with Baby-Step Giant-Step

## Idea and Problem

Find the least exponent within an inclusive bound whose modular power equals a target when the base is invertible modulo the declared modulus.

For step size `s`, write each candidate as `x = i * s + j`. A table maps
every baby step `base**j` to its least exponent `j`. Successive giant steps
multiply the target by the inverse of `base**s`, so a table match reconstructs
one candidate without scanning every preceding exponent.

Giant-step blocks and baby exponents are both visited in increasing order.
The first in-bound match is therefore the least valid exponent even when
powers repeat before the inclusive search bound.

## When to Use

Use this function when the base is known to be coprime to a bounded modulus,
the exponent has a finite public upper bound, and the least matching exponent
is required. It is useful for exact modular-state recovery, bounded cycle
analysis, and independent checks where a linear scan across the complete
exponent interval would be too slow.

Use direct exponentiation when only a proposed exponent needs verification.
Use a linear scan for very small bounds, and choose a number-theory library for
large groups, unbounded discrete logarithms, non-coprime bases, subgroup-order
analysis, or algorithms intended for cryptographic-size operands.

## Implementation

```python
from math import gcd, isqrt

_MAX_DISCRETE_LOGARITHM_MODULUS = (1 << 63) - 1
_MAX_DISCRETE_LOGARITHM_EXPONENT = 1_000_000


def least_bounded_discrete_logarithm(
    base: int,
    target: int,
    modulus: int,
    maximum_exponent: int,
) -> int | None:
    """Return the least in-bound exponent, or None when no match exists."""
    if type(base) is not int:
        raise TypeError("base must be an exact integer")
    if type(target) is not int:
        raise TypeError("target must be an exact integer")
    if type(modulus) is not int:
        raise TypeError("modulus must be an exact integer")
    if type(maximum_exponent) is not int:
        raise TypeError("maximum_exponent must be an exact integer")
    if not 2 <= modulus <= _MAX_DISCRETE_LOGARITHM_MODULUS:
        raise ValueError("modulus is outside the supported range")
    if not 0 <= base < modulus:
        raise ValueError("base must be normalized into the modulus range")
    if not 0 <= target < modulus:
        raise ValueError("target must be normalized into the modulus range")
    if not 0 <= maximum_exponent <= _MAX_DISCRETE_LOGARITHM_EXPONENT:
        raise ValueError("maximum_exponent is outside the supported range")
    if gcd(base, modulus) != 1:
        raise ValueError("base must be coprime to modulus")

    if target == 1:
        return 0

    step_size = isqrt(maximum_exponent) + 1
    baby_exponents: dict[int, int] = {}
    baby_value = 1
    for exponent in range(step_size):
        baby_exponents.setdefault(baby_value, exponent)
        baby_value = baby_value * base % modulus

    inverse_giant_step = pow(baby_value, -1, modulus)
    reduced_target = target
    for giant_index in range(maximum_exponent // step_size + 1):
        baby_exponent = baby_exponents.get(reduced_target)
        if baby_exponent is not None:
            candidate = giant_index * step_size + baby_exponent
            if (
                candidate <= maximum_exponent
                and pow(base, candidate, modulus) == target
            ):
                return candidate
        reduced_target = reduced_target * inverse_giant_step % modulus

    return None
```

## Example

```python
def least_discrete_logarithm_by_search(
    base: int,
    target: int,
    modulus: int,
    maximum_exponent: int,
) -> int | None:
    value = 1
    for exponent in range(maximum_exponent + 1):
        if value == target:
            return exponent
        value = value * base % modulus
    return None


def exercise_small_moduli() -> int:
    from math import gcd

    checked = 0
    for modulus in range(2, 13):
        for base in range(modulus):
            if gcd(base, modulus) != 1:
                continue
            for target in range(modulus):
                for maximum_exponent in range(25):
                    assert least_bounded_discrete_logarithm(
                        base,
                        target,
                        modulus,
                        maximum_exponent,
                    ) == least_discrete_logarithm_by_search(
                        base,
                        target,
                        modulus,
                        maximum_exponent,
                    )
                    checked += 1
    return checked


def exercise_seeded_inputs() -> int:
    from math import gcd
    from random import Random

    generator = Random(20_260_730)
    checked = 0
    while checked < 500:
        modulus = generator.randint(2, 500)
        base = generator.randrange(modulus)
        if gcd(base, modulus) != 1:
            continue
        target = generator.randrange(modulus)
        maximum_exponent = generator.randint(0, 200)
        assert least_bounded_discrete_logarithm(
            base,
            target,
            modulus,
            maximum_exponent,
        ) == least_discrete_logarithm_by_search(
            base,
            target,
            modulus,
            maximum_exponent,
        )
        checked += 1
    return checked


def raises(error_type: type[Exception], operation) -> bool:
    try:
        operation()
    except error_type:
        return True
    return False


maximum_modulus = _MAX_DISCRETE_LOGARITHM_MODULUS
invalid_calls = (
    lambda: least_bounded_discrete_logarithm(True, 1, 2, 0),
    lambda: least_bounded_discrete_logarithm(1, False, 2, 0),
    lambda: least_bounded_discrete_logarithm(1, 1, True, 0),
    lambda: least_bounded_discrete_logarithm(1, 1, 2, False),
    lambda: least_bounded_discrete_logarithm(0, 0, 1, 0),
    lambda: least_bounded_discrete_logarithm(2, 1, 2, 0),
    lambda: least_bounded_discrete_logarithm(1, 2, 2, 0),
    lambda: least_bounded_discrete_logarithm(1, 1, 2, -1),
    lambda: least_bounded_discrete_logarithm(
        1,
        1,
        2,
        _MAX_DISCRETE_LOGARITHM_EXPONENT + 1,
    ),
    lambda: least_bounded_discrete_logarithm(2, 1, 4, 4),
)

assert (
    exercise_small_moduli(),
    exercise_seeded_inputs(),
    least_bounded_discrete_logarithm(2, 1, 7, 6),
    least_bounded_discrete_logarithm(2, 4, 7, 1),
    least_bounded_discrete_logarithm(2, 4, 7, 2),
    least_bounded_discrete_logarithm(2, 0, 5, _MAX_DISCRETE_LOGARITHM_EXPONENT),
    least_bounded_discrete_logarithm(
        maximum_modulus - 1,
        maximum_modulus - 1,
        maximum_modulus,
        _MAX_DISCRETE_LOGARITHM_EXPONENT,
    ),
    sum(raises((TypeError, ValueError), call) for call in invalid_calls),
) == (
    9_350,
    500,
    0,
    None,
    2,
    None,
    1,
    10,
)
```

## Trade-offs and Limitations

For inclusive bound `B`, the step size is `floor(sqrt(B)) + 1`. Building the
baby table and scanning giant blocks use `O(sqrt(B))` expected dictionary
operations and modular multiplications, while the table retains
`O(sqrt(B))` Python-integer entries. Modular multiplication, inversion, and
dictionary hashing have operand-dependent costs under the signed-63-bit
modulus cap.

Inputs are canonical residues: `0 <= base, target < modulus`. The base must be
coprime to the modulus, but the target need not be; a non-coprime target simply
cannot match a power of the accepted base and returns `None`. Exponent zero is
always considered, the upper bound is inclusive, and repeated powers still
return the least matching exponent.

The routine searches only through the declared maximum exponent. `None` means
there is no match in that finite interval, not that no discrete logarithm
exists for every larger exponent. The implementation does not handle
non-coprime bases, derive a group order, prove subgroup membership separately,
enumerate later solutions, resist timing analysis, or target cryptographic-size
discrete-logarithm problems.

## Related Snippets

<!-- catalog:related:start -->
- [Solve a Bounded Linear Congruence as a Canonical Residue Class](solve-a-bounded-linear-congruence-as-a-canonical-residue-class.md)
- [Compute Canonical Bézout Coefficients for Two Bounded Integers](compute-canonical-bezout-coefficients-for-two-bounded-integers.md)
- [Combine a Bounded System of Possibly Non-Coprime Integer Congruences](combine-a-bounded-system-of-possibly-non-coprime-integer-congruences.md)
<!-- catalog:related:end -->
