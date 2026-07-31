---
title: "Invert Bounded Nonzero Prime-Field Residues with One Modular Inversion"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - solve-a-bounded-linear-congruence-as-a-canonical-residue-class.md
  - combine-a-bounded-system-of-possibly-non-coprime-integer-congruences.md
  - compute-a-monic-gcd-of-two-bounded-prime-field-polynomials.md
---

# Invert Bounded Nonzero Prime-Field Residues with One Modular Inversion

## Idea and Problem

Invert a bounded batch of nonzero residues modulo a small verified prime while performing only one modular inversion.

Forward prefix products capture the product before each value. After inverting
the total product once, a reverse sweep combines that suffix inverse with the
saved prefix to recover each individual inverse. Multiplying the suffix
inverse by the current value then prepares it for the preceding position.

## When to Use

Use this technique when one algorithmic step needs inverses of many known
nonzero elements from the same prime field. It is useful when modular
inversion is relatively expensive and the surrounding work already operates
on canonical field residues.

Use individual `pow(value, -1, prime)` calls for isolated values or when their
simplicity benchmarks faster for a small Python batch. Use a broader
finite-field implementation when values carry a reusable field type, moduli
are large, or zero-handling policy is part of the application.

## Implementation

```python
from math import isqrt

_MAX_BATCH_INVERSION_COUNT = 65_536
_MAX_BATCH_INVERSION_PRIME = 65_521


def _validate_batch_inversion_prime(prime: int) -> None:
    if type(prime) is not int:
        raise TypeError("prime must be an exact integer")
    if not 2 <= prime <= _MAX_BATCH_INVERSION_PRIME:
        raise ValueError("prime is outside 2..65,521")
    if prime % 2 == 0:
        if prime != 2:
            raise ValueError("prime must be prime")
        return
    for divisor in range(3, isqrt(prime) + 1, 2):
        if prime % divisor == 0:
            raise ValueError("prime must be prime")


def invert_prime_field_batch(
    values: tuple[int, ...],
    prime: int,
) -> tuple[int, ...]:
    """Return each canonical nonzero residue's inverse modulo prime."""
    _validate_batch_inversion_prime(prime)
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    if len(values) > _MAX_BATCH_INVERSION_COUNT:
        raise ValueError("value count exceeds 65,536")

    prefix_products: list[int] = []
    total_product = 1
    for index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"values[{index}] must be an exact integer")
        if not 1 <= value < prime:
            raise ValueError(f"values[{index}] is not a canonical nonzero residue")
        prefix_products.append(total_product)
        total_product = total_product * value % prime

    if not values:
        return ()

    inverse_suffix = pow(total_product, -1, prime)
    inverses = [0] * len(values)
    for index in range(len(values) - 1, -1, -1):
        value = values[index]
        inverses[index] = inverse_suffix * prefix_products[index] % prime
        inverse_suffix = inverse_suffix * value % prime
    return tuple(inverses)
```

## Example

```python
def exercise_reference_cases() -> int:
    from itertools import product
    from random import Random

    checked = 0
    for small_prime in (2, 3, 5, 7):
        residues = range(1, small_prime)
        for length in range(5):
            for sample in product(residues, repeat=length):
                expected = tuple(pow(value, -1, small_prime) for value in sample)
                actual = invert_prime_field_batch(sample, small_prime)
                assert actual == expected
                assert all(
                    value * inverse % small_prime == 1
                    for value, inverse in zip(sample, actual, strict=True)
                )
                checked += 1

    rng = Random(0)
    for random_prime in (11, 101, 257, 65_521):
        sample = tuple(rng.randrange(1, random_prime) for _ in range(257))
        assert invert_prime_field_batch(sample, random_prime) == tuple(
            pow(value, -1, random_prime) for value in sample
        )
    return checked


checked = exercise_reference_cases()

maximum_values = tuple(
    index % (_MAX_BATCH_INVERSION_PRIME - 1) + 1 for index in range(_MAX_BATCH_INVERSION_COUNT)
)
maximum_inverses = invert_prime_field_batch(
    maximum_values,
    _MAX_BATCH_INVERSION_PRIME,
)


class TupleSubclass(tuple):
    pass


rejected = 0
invalid_actions = (
    lambda: invert_prime_field_batch(TupleSubclass((1,)), 2),
    lambda: invert_prime_field_batch((True,), 2),
    lambda: invert_prime_field_batch((0,), 3),
    lambda: invert_prime_field_batch((3,), 3),
    lambda: invert_prime_field_batch((1,), True),
    lambda: invert_prime_field_batch((1,), 1),
    lambda: invert_prime_field_batch((1,), 15),
    lambda: invert_prime_field_batch(
        (1,) * (_MAX_BATCH_INVERSION_COUNT + 1),
        2,
    ),
)
for invalid_action in invalid_actions:
    try:
        invalid_action()
    except (TypeError, ValueError):
        rejected += 1

assert (
    checked > 0
    and invert_prime_field_batch((), 2) == ()
    and len(maximum_inverses) == _MAX_BATCH_INVERSION_COUNT
    and all(
        value * inverse % _MAX_BATCH_INVERSION_PRIME == 1
        for value, inverse in zip(maximum_values, maximum_inverses, strict=True)
    )
    and rejected == len(invalid_actions)
)
```

## Trade-offs and Limitations

Prime validation takes `O(sqrt(prime))` bounded trial divisions. For `N`
values, the batch then performs `O(N)` modular multiplications, one modular
inversion, and stores `O(N)` prefix products and results. Arithmetic cost still
depends on the modulus bit width.

All inputs must be canonical nonzero residues of the same verified prime
field. The empty tuple needs no inversion. The implementation deliberately
rejects zero, composite moduli, noncanonical representatives, Booleans and
tuple subclasses instead of silently reducing or filtering them.

Python's modular `pow` is implemented below Python level, so repeated direct
calls can outperform this Python loop for small batches despite doing more
inversions. This code is not constant-time and is unsuitable for secret values
when timing or memory-access leakage matters. It does not support composite
moduli, partial invertibility, mutable cached products, or incremental updates.

## Related Snippets

<!-- catalog:related:start -->
- [Solve a Bounded Linear Congruence as a Canonical Residue Class](solve-a-bounded-linear-congruence-as-a-canonical-residue-class.md)
- [Combine a Bounded System of Possibly Non-Coprime Integer Congruences](combine-a-bounded-system-of-possibly-non-coprime-integer-congruences.md)
- [Compute a Monic GCD of Two Bounded Prime-Field Polynomials](compute-a-monic-gcd-of-two-bounded-prime-field-polynomials.md)
<!-- catalog:related:end -->
