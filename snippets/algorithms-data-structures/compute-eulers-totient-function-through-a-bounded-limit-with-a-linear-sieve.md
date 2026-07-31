---
title: "Compute Euler's Totient Function Through a Bounded Limit with a Linear Sieve"
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
  - enumerate-primes-in-a-bounded-half-open-range-with-a-segmented-sieve.md
  - factor-a-bounded-positive-integer-by-deterministic-trial-division.md
  - test-unsigned-64-bit-integers-for-primality-with-deterministic-miller-rabin.md
---

# Compute Euler's Totient Function Through a Bounded Limit with a Linear Sieve

## Idea and Problem

Compute Euler's totient value for every integer through one inclusive bound in linear time. The totient phi(n) counts the integers from 1 through n that are coprime to n.

An Euler sieve discovers primes while scanning the integers in ascending order.
For each current value, it visits products with known primes only until the
current value's smallest prime factor is reached. Consequently, every composite
is produced exactly once, with enough information to apply one of two rules:
`phi(n * p) = phi(n) * p` when `p` divides `n`, and
`phi(n * p) = phi(n) * (p - 1)` otherwise.

The returned tuple is indexed directly by the integer whose totient is needed.
Index zero is the explicit sentinel `0`; `phi(1)` is `1` whenever that index is
present.

## When to Use

Use this when many totient values in one dense bounded interval are required,
such as in divisor-sum calculations, exact number-theory tables, or reference
checks. One pass shares prime discovery and factor information across all
values, avoiding separate factorization for each integer.

Use direct factorization when only a few unrelated totients are needed. A
smaller prime-only sieve is preferable when the totient table itself is not
needed, and a segmented or specialized approach is required when the full
dense table does not fit in memory.

## Implementation

```python
_MAX_TOTIENT_LIMIT = 1_000_000


def linear_totients(limit: int) -> tuple[int, ...]:
    """Return phi(0), phi(1), ..., phi(limit) using an Euler sieve."""
    if type(limit) is not int:
        raise TypeError("limit must be an exact integer")
    if not 0 <= limit <= _MAX_TOTIENT_LIMIT:
        raise ValueError("limit is outside 0..1,000,000")

    totients = [0] * (limit + 1)
    if limit == 0:
        return (0,)

    totients[1] = 1
    primes: list[int] = []
    for value in range(2, limit + 1):
        if totients[value] == 0:
            primes.append(value)
            totients[value] = value - 1

        for prime in primes:
            product = value * prime
            if product > limit:
                break
            if value % prime == 0:
                totients[product] = totients[value] * prime
                break
            totients[product] = totients[value] * (prime - 1)

    return tuple(totients)
```

## Example

```python
from math import gcd


def direct_totient(value: int) -> int:
    if value == 0:
        return 0
    return sum(gcd(candidate, value) == 1 for candidate in range(1, value + 1))


small_table = linear_totients(256)
assert small_table == tuple(direct_totient(value) for value in range(257))

identity_table = linear_totients(1_024)
for value in range(1, 1_025):
    assert (
        sum(identity_table[divisor] for divisor in range(1, value + 1) if value % divisor == 0)
        == value
    )

for prime in (2, 3, 5, 97, 251, 1_021):
    assert identity_table[prime] == prime - 1

maximum_table = linear_totients(_MAX_TOTIENT_LIMIT)
assert maximum_table[1_000_000] == 400_000

type_errors = 0
for invalid_limit in (True, 1.0, "10", None):
    try:
        linear_totients(invalid_limit)  # type: ignore[arg-type]
    except TypeError:
        type_errors += 1

value_errors = 0
for invalid_limit in (-1, _MAX_TOTIENT_LIMIT + 1):
    try:
        linear_totients(invalid_limit)
    except ValueError:
        value_errors += 1

assert (
    linear_totients(0),
    linear_totients(1),
    linear_totients(10),
    len(maximum_table),
    type_errors,
    value_errors,
) == (
    (0,),
    (0, 1),
    (0, 1, 1, 2, 2, 4, 2, 6, 4, 6, 4),
    1_000_001,
    4,
    2,
)
```

## Trade-offs and Limitations

The Euler sieve performs `O(limit)` integer-arithmetic steps and uses
`O(limit)` memory for the mutable table, discovered primes, and immutable
result. Those bounds count Python-level operations and entries; Python integer,
list, and tuple object overhead can be much larger than a packed numeric array.

The function validates and rebuilds the complete table on every call. It does
not cache or extend earlier results, stream values, return the prime list or
smallest prime factors, factor individual integers, or support bounds above
one million. The value at index zero is a convenient sentinel rather than a
mathematical definition of `phi(0)`.

## Related Snippets

<!-- catalog:related:start -->
- [Enumerate Primes in a Bounded Half-Open Range with a Segmented Sieve](enumerate-primes-in-a-bounded-half-open-range-with-a-segmented-sieve.md)
- [Factor a Bounded Positive Integer by Deterministic Trial Division](factor-a-bounded-positive-integer-by-deterministic-trial-division.md)
- [Test Unsigned 64-Bit Integers for Primality with Deterministic Miller-Rabin](test-unsigned-64-bit-integers-for-primality-with-deterministic-miller-rabin.md)
<!-- catalog:related:end -->
