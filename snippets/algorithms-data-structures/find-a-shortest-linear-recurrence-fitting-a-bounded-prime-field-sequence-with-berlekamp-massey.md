---
title: "Find a Shortest Linear Recurrence Fitting a Bounded Prime-Field Sequence with Berlekamp-Massey"
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
  - compute-a-distant-linear-recurrence-term-modulo-an-integer-by-polynomial-reduction.md
  - interpolate-a-global-polynomial-exactly-from-bounded-integer-points.md
  - convolve-bounded-integer-polynomials-modulo-998244353-with-an-iterative-ntt.md
---

# Find a Shortest Linear Recurrence Fitting a Bounded Prime-Field Sequence with Berlekamp-Massey

## Idea and Problem

Infer one shortest constant-coefficient recurrence that reproduces an observed finite sequence over a small prime field.

Berlekamp-Massey scans the residues from left to right while maintaining a
connection polynomial. A nonzero discrepancy identifies the first value that
the current relation does not explain. A shifted, scaled copy of the last
useful connection polynomial removes that discrepancy, and the remembered
polynomial changes only when the required order must grow.

This implementation returns coefficients in newest-term-first order:
`sample[n] = sum(coefficients[j] * sample[n - 1 - j] for j in range(L))`
modulo the declared prime. That orientation matches direct recurrence
evaluation and avoids exposing the connection polynomial's opposite signs.

## When to Use

Use this algorithm when a bounded sequence is known exactly modulo one prime
and a compact recurrence can replace the stored prefix, support a later-term
calculation, or provide a reproducible structural check. Every input value must
already be a canonical residue, and only agreement with the observed prefix is
promised.

Use regression or an error-correcting reconstruction method when samples are
noisy. Use ordinary exact linear algebra when the coefficient positions are
fixed in advance, and use the existing distant-term algorithm after the
recurrence is already known and a far-away value is the actual goal.

## Implementation

```python
from itertools import product
from math import isqrt
from random import Random

_MAX_SEQUENCE_LENGTH = 512
_MAX_FIELD_PRIME = 65_521


def _is_prime(value: int) -> bool:
    if value < 2:
        return False
    if value % 2 == 0:
        return value == 2
    limit = isqrt(value)
    divisor = 3
    while divisor <= limit:
        if value % divisor == 0:
            return False
        divisor += 2
    return True


def shortest_prime_field_recurrence(
    sample: tuple[int, ...],
    prime: int,
) -> tuple[int, ...]:
    """Return one shortest newest-term-first recurrence for sample."""
    if type(prime) is not int:
        raise TypeError("prime must be an exact integer")
    if not 2 <= prime <= _MAX_FIELD_PRIME or not _is_prime(prime):
        raise ValueError("prime must be prime and inside 2..65,521")
    if type(sample) is not tuple:
        raise TypeError("sample must be an exact tuple")
    if not 1 <= len(sample) <= _MAX_SEQUENCE_LENGTH:
        raise ValueError("sample length is outside 1..512")
    for index, value in enumerate(sample):
        if type(value) is not int:
            raise TypeError(f"sample[{index}] must be an exact integer")
        if not 0 <= value < prime:
            raise ValueError(f"sample[{index}] is not a canonical field residue")

    connection = [1]
    previous_connection = [1]
    order = 0
    shift = 1
    previous_discrepancy = 1

    for index, value in enumerate(sample):
        discrepancy = value
        for offset in range(1, order + 1):
            discrepancy += connection[offset] * sample[index - offset]
        discrepancy %= prime

        if discrepancy == 0:
            shift += 1
            continue

        old_connection = connection.copy()
        scale = discrepancy * pow(previous_discrepancy, -1, prime) % prime
        required_length = len(previous_connection) + shift
        if len(connection) < required_length:
            connection.extend([0] * (required_length - len(connection)))
        for offset, coefficient in enumerate(previous_connection):
            target = offset + shift
            connection[target] = (connection[target] - scale * coefficient) % prime

        if 2 * order <= index:
            order = index + 1 - order
            previous_connection = old_connection
            previous_discrepancy = discrepancy
            shift = 1
        else:
            shift += 1

    return tuple((-connection[offset]) % prime for offset in range(1, order + 1))
```

## Example

```python
def fits_observed_prefix(
    sample: tuple[int, ...],
    coefficients: tuple[int, ...],
    prime: int,
) -> bool:
    order = len(coefficients)
    return all(
        sample[index]
        == sum(
            coefficient * sample[index - 1 - offset]
            for offset, coefficient in enumerate(coefficients)
        )
        % prime
        for index in range(order, len(sample))
    )


def brute_minimum_order(sample: tuple[int, ...], prime: int) -> int:
    for order in range(len(sample) + 1):
        for coefficients in product(range(prime), repeat=order):
            if fits_observed_prefix(sample, coefficients, prime):
                return order
    raise AssertionError("a recurrence of order len(sample) is always available")


exhaustive_sequences = 0
for prime, maximum_length in ((2, 9), (3, 6)):
    for length in range(1, maximum_length + 1):
        for sample in product(range(prime), repeat=length):
            inferred = shortest_prime_field_recurrence(sample, prime)
            assert fits_observed_prefix(sample, inferred, prime)
            assert len(inferred) == brute_minimum_order(sample, prime)
            exhaustive_sequences += 1

rng = Random(17)
generated_sequences = 0
for prime in (2, 3, 5, 257):
    for declared_order in range(1, 9):
        declared = tuple(rng.randrange(prime) for _ in range(declared_order))
        values = [rng.randrange(prime) for _ in range(declared_order)]
        while len(values) < 64:
            values.append(
                sum(
                    coefficient * values[-1 - offset]
                    for offset, coefficient in enumerate(declared)
                )
                % prime
            )
        sample = tuple(values)
        inferred = shortest_prime_field_recurrence(sample, prime)
        assert fits_observed_prefix(sample, inferred, prime)
        assert len(inferred) <= declared_order
        generated_sequences += 1

fibonacci = shortest_prime_field_recurrence((0, 1, 1, 2, 3, 5, 8, 13), 65_521)
all_zero = shortest_prime_field_recurrence((0,) * _MAX_SEQUENCE_LENGTH, 2)
maximum_sample = tuple(index % 65_521 for index in range(_MAX_SEQUENCE_LENGTH))
maximum_result = shortest_prime_field_recurrence(maximum_sample, 65_521)

for invalid_prime in (1, 4, 65_523):
    try:
        shortest_prime_field_recurrence((0,), invalid_prime)
    except ValueError:
        pass
    else:
        raise AssertionError("accepted a non-prime or out-of-range modulus")

assert (
    fibonacci,
    all_zero,
    fits_observed_prefix(maximum_sample, maximum_result, 65_521),
    exhaustive_sequences,
    generated_sequences,
) == ((1, 1), (), True, 2_114, 32)
```

## Trade-offs and Limitations

The scan uses `O(N²)` field operations in the worst case and `O(N)` stored
field elements. Trial-division validation adds `O(sqrt(p))` small-integer
divisibility checks. Python modular arithmetic and inversion costs still grow
with operand bit length, although both the prime and sequence are bounded here.

A finite prefix can admit several different recurrences of the same minimum
order. The update rules return one deterministic Berlekamp-Massey result, but
it is not advertised as the lexicographically smallest or otherwise canonical
member of that set. A short prefix can also support a relation that fails on
the next unseen term, so this is recurrence fitting, not proof of an unknown
generator or a forecasting guarantee.

The function supports prime fields only. It does not handle composite rings,
extension fields, missing samples, corrupt residues, multivariate sequences,
confidence measures, or automatic distant-term evaluation.

## Related Snippets

<!-- catalog:related:start -->
- [Compute a Distant Linear-Recurrence Term Modulo an Integer by Polynomial Reduction](compute-a-distant-linear-recurrence-term-modulo-an-integer-by-polynomial-reduction.md)
- [Interpolate a Global Polynomial Exactly from Bounded Integer Points](interpolate-a-global-polynomial-exactly-from-bounded-integer-points.md)
- [Convolve Bounded Integer Polynomials Modulo 998244353 with an Iterative NTT](convolve-bounded-integer-polynomials-modulo-998244353-with-an-iterative-ntt.md)
<!-- catalog:related:end -->
