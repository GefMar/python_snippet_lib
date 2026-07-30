---
title: "Factor a Bounded Positive Integer by Deterministic Trial Division"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - test-unsigned-64-bit-integers-for-primality-with-deterministic-miller-rabin.md
  - enumerate-primes-in-a-bounded-half-open-range-with-a-segmented-sieve.md
---

# Factor a Bounded Positive Integer by Deterministic Trial Division

## Idea and Problem

Decompose one bounded positive integer into its prime factors and exact multiplicities without probabilistic choices.

Remove every factor of two first, then try odd divisors in ascending order.
After each successful divisor, shrink the remaining quotient and its integer
square-root limit. If the quotient is still greater than one after no divisor
through that limit remains, the quotient itself is prime.

The result is an immutable tuple of `(prime, exponent)` pairs ordered by
increasing prime. The multiplicities reconstruct the input exactly, while one
has the canonical empty factorization.

## When to Use

Use trial division when one positive integer is no larger than the explicit
limit and actual factors are required. It is simple, deterministic, and useful
for bounded validation, small arithmetic fixtures, or reducing one modest
composite into prime powers.

Use a sieve when many values from a dense interval must be processed. Use a
stronger factorization library for large machine-word values or adversarial
near-primes: a fast primality predicate can classify them, but trial division
must still inspect every possible small divisor before returning a factorization.

## Implementation

```python
from math import isqrt

_MAX_TRIAL_DIVISION_VALUE = 1_000_000_000_000


def factor_positive_integer_by_trial_division(
    value: int,
) -> tuple[tuple[int, int], ...]:
    """Return ascending prime factors and their positive exponents."""
    if type(value) is not int:
        raise TypeError("value must be an exact non-boolean integer")
    if not 1 <= value <= _MAX_TRIAL_DIVISION_VALUE:
        raise ValueError("value is outside the supported range")

    remaining = value
    factors: list[tuple[int, int]] = []

    power_of_two = 0
    while remaining % 2 == 0:
        remaining //= 2
        power_of_two += 1
    if power_of_two:
        factors.append((2, power_of_two))

    divisor = 3
    limit = isqrt(remaining)
    while divisor <= limit:
        exponent = 0
        while remaining % divisor == 0:
            remaining //= divisor
            exponent += 1
        if exponent:
            factors.append((divisor, exponent))
            limit = isqrt(remaining)
        divisor += 2

    if remaining > 1:
        factors.append((remaining, 1))

    return tuple(factors)
```

## Example

```python
def factor_positive_integer_by_direct_search(
    value: int,
) -> tuple[tuple[int, int], ...]:
    remaining = value
    factors: list[tuple[int, int]] = []
    for candidate in range(2, value + 1):
        exponent = 0
        while remaining % candidate == 0:
            remaining //= candidate
            exponent += 1
        if exponent:
            factors.append((candidate, exponent))
        if remaining == 1:
            break
    return tuple(factors)


checked_values = 0
for small_value in range(1, 513):
    assert factor_positive_integer_by_trial_division(
        small_value
    ) == factor_positive_integer_by_direct_search(small_value)
    checked_values += 1

near_limit_square = 999_983**2
near_limit_factors = factor_positive_integer_by_trial_division(near_limit_square)

rejected = 0
for invalid_value in (True, 0, -1, _MAX_TRIAL_DIVISION_VALUE + 1):
    try:
        factor_positive_integer_by_trial_division(invalid_value)
    except (TypeError, ValueError):
        rejected += 1

assert (
    checked_values,
    factor_positive_integer_by_trial_division(1),
    factor_positive_integer_by_trial_division(360),
    factor_positive_integer_by_trial_division(_MAX_TRIAL_DIVISION_VALUE),
    near_limit_factors,
    rejected,
) == (
    512,
    (),
    ((2, 3), (3, 2), (5, 1)),
    ((2, 12), (5, 12)),
    ((999_983, 2),),
    4,
)
```

## Trade-offs and Limitations

In the worst case, an input with no small factor requires `O(sqrt(N))`
divisibility checks. Repeated division adds at most `O(log N)` successful
steps. The factor list uses `O(log N)` output space in the loose worst-case
bound, while the algorithm retains `O(1)` additional Python-integer state.
The bit cost of division grows with the operands and is not hidden by the
operation-count bound.

The function accepts one exact non-Boolean integer from one through
1,000,000,000,000. Factors are strictly increasing primes, exponents are
positive, and their prime powers multiply back to the input. No factor is
returned for one.

The fixed cap keeps the worst-case divisor scan finite but does not make every
near-prime cheap. The function does not accept zero or negative values, reuse a
sieve across calls, return primality certificates, apply probabilistic methods,
or claim suitability for cryptographic factorization.

## Related Snippets

<!-- catalog:related:start -->
- [Test Unsigned 64-Bit Integers for Primality with Deterministic Miller-Rabin](test-unsigned-64-bit-integers-for-primality-with-deterministic-miller-rabin.md)
- [Enumerate Primes in a Bounded Half-Open Range with a Segmented Sieve](enumerate-primes-in-a-bounded-half-open-range-with-a-segmented-sieve.md)
<!-- catalog:related:end -->
