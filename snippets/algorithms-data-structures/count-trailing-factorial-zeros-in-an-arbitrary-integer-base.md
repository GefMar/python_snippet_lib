---
title: "Count Trailing Factorial Zeros in an Arbitrary Integer Base"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
tested_python:
  - "3.14"
dependencies: []
related:
  - factor-a-bounded-positive-integer-by-deterministic-trial-division.md
  - compute-a-huge-binomial-coefficient-modulo-a-small-prime-with-lucass-theorem.md
---

# Count Trailing Factorial Zeros in an Arbitrary Integer Base

## Idea and Problem

Count the trailing zeros of a factorial in any integer base without computing the factorial or converting it to digits.

Each trailing zero requires one factor of the base. For a prime `p`, count
the multiples of `p`, then the multiples of `p**2` for an additional factor,
and so on. If the base contains `p` with exponent `e`, divide that total by
`e` and round down. The smallest result across the base's prime factors is
the number of complete factors of the base in the factorial.

## When to Use

Use this for exact divisibility checks or digit-count questions involving a
large factorial and a reasonably small base. The inputs are integers with
`n >= 0` and `base >= 2`. Both `0!` and `1!` are one and have no trailing
zeros in any valid base.

## Implementation

```python
from collections.abc import Iterator


def _prime_powers(value: int) -> Iterator[tuple[int, int]]:
    divisor = 2
    while divisor * divisor <= value:
        exponent = 0
        while value % divisor == 0:
            value //= divisor
            exponent += 1
        if exponent:
            yield divisor, exponent
        divisor = 3 if divisor == 2 else divisor + 2
    if value > 1:
        yield value, 1


def _prime_exponent_in_factorial(n: int, prime: int) -> int:
    total = 0
    while n >= prime:
        n //= prime
        total += n
    return total


def factorial_trailing_zeros(n: int, base: int) -> int:
    if type(n) is not int or type(base) is not int:
        raise TypeError("n and base must be integers")
    if n < 0 or base < 2:
        raise ValueError("require n >= 0 and base >= 2")
    if n < 2:
        return 0
    return min(
        _prime_exponent_in_factorial(n, prime) // exponent
        for prime, exponent in _prime_powers(base)
    )
```

## Example

```python
assert factorial_trailing_zeros(10, 8) == 2
assert factorial_trailing_zeros(20, 12) == 8
assert factorial_trailing_zeros(10, 11) == 0
assert factorial_trailing_zeros(0, 16) == 0
assert factorial_trailing_zeros(1_000_000, 10) == 249_998
```

## Trade-offs and Limitations

Each prime `p` needs at most `floor(log_p(n))` quotient steps. Trial division
may test candidates up to the square root of the base, so large primes or
products of two large primes are slow. This is not a bounded-time routine for
arbitrary user-provided bases. Factor a repeated base once and reuse its
prime powers when answering many queries.

## Related Snippets

<!-- catalog:related:start -->
- [Factor a Bounded Positive Integer by Deterministic Trial Division](factor-a-bounded-positive-integer-by-deterministic-trial-division.md)
- [Compute a Huge Binomial Coefficient Modulo a Small Prime with Lucas's Theorem](compute-a-huge-binomial-coefficient-modulo-a-small-prime-with-lucass-theorem.md)
<!-- catalog:related:end -->
