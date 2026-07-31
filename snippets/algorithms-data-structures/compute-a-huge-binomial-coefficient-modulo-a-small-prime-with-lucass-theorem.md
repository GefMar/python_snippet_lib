---
title: "Compute a Huge Binomial Coefficient Modulo a Small Prime with Lucas's Theorem"
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
  - rank-and-unrank-index-combinations-in-itertools-combinations-order.md
  - test-unsigned-64-bit-integers-for-primality-with-deterministic-miller-rabin.md
  - compute-a-monic-gcd-of-two-bounded-prime-field-polynomials.md
---

# Compute a Huge Binomial Coefficient Modulo a Small Prime with Lucas's Theorem

## Idea and Problem

Evaluate one binomial coefficient modulo a small prime even when the full coefficient and its row number are enormous.

Lucas's theorem writes `n` and `k` in base `p`. The desired residue is the
product of the small digit-level coefficients `C(n_i, k_i) mod p`. If any
digit of `k` exceeds the matching digit of `n`, that factor and therefore the
whole result is zero.

For one verified prime, factorial and inverse-factorial tables cover every
possible base-`p` digit. The implementation builds those tables inside the
call, so no mutable cache or modulus-specific global state can leak between
uses.

## When to Use

Use this when `n` and `k` may be far too large for factorial construction up
to `n`, but the modulus is a known small prime. It is useful in modular
combinatorics, counting dynamic programs, and reference checks where an exact
residue is enough.

The modulus must be prime and small enough to materialize `O(p)` tables. Use a
different method for composite moduli, prime powers, many calls that should
share precomputed tables, or when the full integer coefficient is required.
For ordinary small values, `math.comb(n, k) % p` is simpler.

## Implementation

```python
_MAX_LUCAS_ARGUMENT = 10**18
_MAX_LUCAS_PRIME = 65_521


def _is_lucas_prime(candidate: int) -> bool:
    if candidate < 2:
        return False
    if candidate % 2 == 0:
        return candidate == 2
    divisor = 3
    while divisor * divisor <= candidate:
        if candidate % divisor == 0:
            return False
        divisor += 2
    return True


def binomial_mod_prime(n: int, k: int, prime: int) -> int:
    """Return C(n, k) modulo one verified small prime."""
    if type(n) is not int:
        raise TypeError("n must be an exact integer")
    if type(k) is not int:
        raise TypeError("k must be an exact integer")
    if type(prime) is not int:
        raise TypeError("prime must be an exact integer")
    if not 0 <= n <= _MAX_LUCAS_ARGUMENT:
        raise ValueError("n is outside 0..10**18")
    if not 0 <= k <= _MAX_LUCAS_ARGUMENT:
        raise ValueError("k is outside 0..10**18")
    if not 2 <= prime <= _MAX_LUCAS_PRIME:
        raise ValueError("prime is outside 2..65,521")
    if not _is_lucas_prime(prime):
        raise ValueError("prime must be prime")
    if k > n:
        return 0

    factorial = [1] * prime
    for value in range(1, prime):
        factorial[value] = factorial[value - 1] * value % prime

    inverse_factorial = [1] * prime
    inverse_factorial[-1] = pow(factorial[-1], prime - 2, prime)
    for value in range(prime - 1, 0, -1):
        inverse_factorial[value - 1] = inverse_factorial[value] * value % prime

    result = 1
    remaining_n = n
    remaining_k = k
    while remaining_n or remaining_k:
        n_digit = remaining_n % prime
        k_digit = remaining_k % prime
        if k_digit > n_digit:
            return 0
        result = (
            result
            * factorial[n_digit]
            * inverse_factorial[k_digit]
            * inverse_factorial[n_digit - k_digit]
            % prime
        )
        remaining_n //= prime
        remaining_k //= prime
    return result
```

## Example

```python
from math import comb
from random import Random

small_checked = 0
for small_prime in (2, 3, 5, 7, 11, 13):
    for small_n in range(81):
        for small_k in range(small_n + 4):
            expected = comb(small_n, small_k) % small_prime if small_k <= small_n else 0
            assert binomial_mod_prime(small_n, small_k, small_prime) == expected
            small_checked += 1

generator = Random(4_133)
identity_checked = 0
for _ in range(300):
    prime = generator.choice((17, 19, 23, 29, 31, 97))
    n = generator.randrange(1, 2_000)
    k = generator.randrange(n + 1)
    value = binomial_mod_prime(n, k, prime)
    assert value == binomial_mod_prime(n, n - k, prime)
    if 0 < k < n:
        assert (
            value
            == (binomial_mod_prime(n - 1, k - 1, prime) + binomial_mod_prime(n - 1, k, prime))
            % prime
        )
    identity_checked += 1

vandermonde_checked = 0
for left in range(9):
    for right in range(9):
        for chosen in range(left + right + 1):
            expected = sum(
                comb(left, from_left) * comb(right, chosen - from_left)
                for from_left in range(left + 1)
                if 0 <= chosen - from_left <= right
            )
            assert binomial_mod_prime(left + right, chosen, 101) == expected % 101
            vandermonde_checked += 1


def rejects(n: object, k: object, prime: object) -> bool:
    try:
        binomial_mod_prime(n, k, prime)  # type: ignore[arg-type]
    except (TypeError, ValueError):
        return True
    return False


invalid_calls = (
    (True, 0, 2),
    (0, True, 2),
    (0, 0, True),
    (-1, 0, 2),
    (0, -1, 2),
    (_MAX_LUCAS_ARGUMENT + 1, 0, 2),
    (0, _MAX_LUCAS_ARGUMENT + 1, 2),
    (0, 0, 1),
    (0, 0, _MAX_LUCAS_PRIME + 1),
    (10, 3, 4),
    (10, 3, 65_520),
)

assert (
    small_checked,
    identity_checked,
    vandermonde_checked,
    binomial_mod_prime(10**18, 1, _MAX_LUCAS_PRIME),
    binomial_mod_prime(10**18, 10**18, _MAX_LUCAS_PRIME),
    binomial_mod_prime(_MAX_LUCAS_PRIME, 1, _MAX_LUCAS_PRIME),
    binomial_mod_prime(2, 3, 2),
    sum(rejects(*call) for call in invalid_calls),
) == (
    21_384,
    300,
    729,
    10**18 % _MAX_LUCAS_PRIME,
    1,
    0,
    0,
    len(invalid_calls),
)
```

## Trade-offs and Limitations

Trial-division validation costs `O(sqrt(p))`. Building the factorial tables
takes `O(p)` modular operations and memory, and processing the base-`p` digits
takes `O(log_p(n + 1))` time. These bounds count modular operations; Python
integer and list-object overhead still matters.

The tables are rebuilt on every call. That keeps the function isolated and
predictable but is wasteful for repeated queries under one prime. A caller
with that workload should own and reuse a separately reviewed table object.

The function returns zero for `k > n`, following the usual extended binomial
convention. It supports only prime moduli, not composite moduli or prime
powers, and it does not construct the full coefficient or expose individual
Lucas digits.

## Related Snippets

<!-- catalog:related:start -->
- [Rank and Unrank Index Combinations in itertools.combinations Order](rank-and-unrank-index-combinations-in-itertools-combinations-order.md)
- [Test Unsigned 64-Bit Integers for Primality with Deterministic Miller-Rabin](test-unsigned-64-bit-integers-for-primality-with-deterministic-miller-rabin.md)
- [Compute a Monic GCD of Two Bounded Prime-Field Polynomials](compute-a-monic-gcd-of-two-bounded-prime-field-polynomials.md)
<!-- catalog:related:end -->
