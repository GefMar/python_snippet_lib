---
title: "Find the Canonical Square Root Modulo a Bounded Odd Prime with Tonelli-Shanks"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - test-unsigned-64-bit-integers-for-primality-with-deterministic-miller-rabin.md
  - find-the-least-bounded-discrete-logarithm-for-a-coprime-base-with-baby-step-giant-step.md
  - solve-a-bounded-linear-congruence-as-a-canonical-residue-class.md
---

# Find the Canonical Square Root Modulo a Bounded Odd Prime with Tonelli-Shanks

## Idea and Problem

Find a reproducible square root of one canonical residue modulo an odd prime, or report that no root exists.

Tonelli-Shanks factors `prime - 1` into an odd part times a power of two. A
supplied quadratic non-residue generates the required two-power subgroup, and
successive corrections reduce the order of the remaining error. When two
roots exist, returning the smaller of `root` and `prime - root` removes the
otherwise arbitrary sign choice.

## When to Use

Use this algorithm for bounded prime-field calculations that need the same
representative every time and already have a known quadratic non-residue for
the modulus. Validating both the prime and witness keeps a malformed arithmetic
precondition from silently producing a plausible value.

Use a number-theory library when the witness must be discovered, composite or
prime-power moduli are required, or inputs exceed the unsigned 64-bit range.
Use audited cryptographic code for secret values: Python's modular
exponentiation and this control flow are not constant-time.

## Implementation

```python
_MAX_UINT64 = (1 << 64) - 1
_PRIMALITY_TRIAL_DIVISORS = (2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37)
_UINT64_PRIMALITY_WITNESSES = (
    2,
    325,
    9_375,
    28_178,
    450_775,
    9_780_504,
    1_795_265_022,
)


def _is_prime_in_uint64_range(candidate: int) -> bool:
    if candidate < 2:
        return False
    for divisor in _PRIMALITY_TRIAL_DIVISORS:
        if candidate == divisor:
            return True
        if candidate % divisor == 0:
            return False

    odd_part = candidate - 1
    power_of_two = 0
    while odd_part % 2 == 0:
        odd_part //= 2
        power_of_two += 1

    for raw_witness in _UINT64_PRIMALITY_WITNESSES:
        witness = raw_witness % candidate
        if witness in (0, 1):
            continue
        residue = pow(witness, odd_part, candidate)
        if residue in (1, candidate - 1):
            continue
        for _ in range(power_of_two - 1):
            residue = residue * residue % candidate
            if residue == candidate - 1:
                break
        else:
            return False
    return True


def canonical_square_root_mod_prime(
    residue: int,
    prime: int,
    quadratic_non_residue: int,
) -> int | None:
    """Return the least modular square root, or None when none exists."""
    if type(residue) is not int:
        raise TypeError("residue must be an exact integer")
    if type(prime) is not int:
        raise TypeError("prime must be an exact integer")
    if type(quadratic_non_residue) is not int:
        raise TypeError("quadratic_non_residue must be an exact integer")
    if not 3 <= prime <= _MAX_UINT64 or prime % 2 == 0:
        raise ValueError("prime must be an odd unsigned 64-bit value")
    if not _is_prime_in_uint64_range(prime):
        raise ValueError("prime must pass deterministic primality validation")
    if not 0 <= residue < prime:
        raise ValueError("residue must be canonical modulo prime")
    if not 2 <= quadratic_non_residue < prime:
        raise ValueError("quadratic_non_residue must be in 2..prime-1")
    if pow(quadratic_non_residue, (prime - 1) // 2, prime) != prime - 1:
        raise ValueError("quadratic_non_residue must fail Euler's residue criterion")

    if residue == 0:
        return 0
    if pow(residue, (prime - 1) // 2, prime) != 1:
        return None
    if prime % 4 == 3:
        root = pow(residue, (prime + 1) // 4, prime)
        return min(root, prime - root)

    odd_part = prime - 1
    power_of_two = 0
    while odd_part % 2 == 0:
        odd_part //= 2
        power_of_two += 1

    root = pow(residue, (odd_part + 1) // 2, prime)
    error = pow(residue, odd_part, prime)
    correction = pow(quadratic_non_residue, odd_part, prime)

    while error != 1:
        least_power = 1
        squared_error = error * error % prime
        while squared_error != 1:
            squared_error = squared_error * squared_error % prime
            least_power += 1

        factor = pow(correction, 1 << (power_of_two - least_power - 1), prime)
        factor_squared = factor * factor % prime
        root = root * factor % prime
        error = error * factor_squared % prime
        correction = factor_squared
        power_of_two = least_power

    return min(root, prime - root)
```

## Example

```python
def first_quadratic_non_residue(prime: int) -> int:
    return next(
        candidate
        for candidate in range(2, prime)
        if pow(candidate, (prime - 1) // 2, prime) == prime - 1
    )


def exercise_small_primes() -> int:
    checked = 0
    for prime in (3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43):
        witness = first_quadratic_non_residue(prime)
        for residue in range(prime):
            roots = [candidate for candidate in range(prime) if candidate**2 % prime == residue]
            expected = min(roots) if roots else None
            assert canonical_square_root_mod_prime(residue, prime, witness) == expected
            checked += 1
    return checked


def raises(error_type: type[Exception], operation) -> bool:
    try:
        operation()
    except error_type:
        return True
    return False


large_prime = 998_244_353
large_witness = 3
known_root = 123_456_789
constructed_square = known_root * known_root % large_prime

largest_uint64_prime = 18_446_744_073_709_551_557
largest_witness = 2
boundary_root = largest_uint64_prime - 123
boundary_square = boundary_root * boundary_root % largest_uint64_prime

assert (
    exercise_small_primes() == 279
    and canonical_square_root_mod_prime(constructed_square, large_prime, large_witness)
    == known_root
    and canonical_square_root_mod_prime(large_witness, large_prime, large_witness) is None
    and canonical_square_root_mod_prime(
        boundary_square,
        largest_uint64_prime,
        largest_witness,
    )
    == 123
    and canonical_square_root_mod_prime(0, largest_uint64_prime, largest_witness) == 0
    and raises(TypeError, lambda: canonical_square_root_mod_prime(True, 7, 3))
    and raises(ValueError, lambda: canonical_square_root_mod_prime(1, 2, 1))
    and raises(ValueError, lambda: canonical_square_root_mod_prime(1, 9, 2))
    and raises(ValueError, lambda: canonical_square_root_mod_prime(7, 7, 3))
    and raises(ValueError, lambda: canonical_square_root_mod_prime(1, 7, 2))
)
```

## Trade-offs and Limitations

The Tonelli-Shanks phase uses `O(log**2 P)` modular operations in the worst
case and retains `O(1)` Python integers. The fixed Miller-Rabin witness set adds
a bounded number of `O(log P)` modular-exponentiation steps. Integer operation
costs still grow with the bit length of `P`.

The modulus must be an odd prime in `3..2**64 - 1`, the residue must already be
canonical, and the caller must supply a canonical non-residue. The function
does not find that witness, factor composite moduli, return both roots, or
provide constant-time behavior, proof certificates, or cryptographic safety.

## Related Snippets

<!-- catalog:related:start -->
- [Test Unsigned 64-Bit Integers for Primality with Deterministic Miller-Rabin](test-unsigned-64-bit-integers-for-primality-with-deterministic-miller-rabin.md)
- [Find the Least Bounded Discrete Logarithm for a Coprime Base with Baby-Step Giant-Step](find-the-least-bounded-discrete-logarithm-for-a-coprime-base-with-baby-step-giant-step.md)
- [Solve a Bounded Linear Congruence as a Canonical Residue Class](solve-a-bounded-linear-congruence-as-a-canonical-residue-class.md)
<!-- catalog:related:end -->
