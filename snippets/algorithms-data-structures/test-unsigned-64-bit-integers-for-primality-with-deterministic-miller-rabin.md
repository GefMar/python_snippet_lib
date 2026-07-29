---
title: "Test Unsigned 64-Bit Integers for Primality with Deterministic Miller-Rabin"
snippet_type: algorithm
use_cases:
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - enumerate-primes-in-a-bounded-half-open-range-with-a-segmented-sieve.md
  - combine-a-bounded-system-of-possibly-non-coprime-integer-congruences.md
  - compute-a-distant-linear-recurrence-term-modulo-an-integer-by-polynomial-reduction.md
---

# Test Unsigned 64-Bit Integers for Primality with Deterministic Miller-Rabin

## Idea and Problem

Decide primality for one unsigned 64-bit integer with a fixed Miller-Rabin witness set that is complete on exactly that finite range.

For an odd candidate, write `n - 1` as `d * 2**s` with `d` odd. A strong
probable-prime round raises one base to `d` modulo `n`, then repeatedly squares
the residue. A composite is exposed unless the sequence starts at one or reaches
minus one modulo `n` at an allowed step.

Miller-Rabin is usually probabilistic, but these seven bases together detect
every composite below `2**64`. Python's three-argument `pow` performs the
modular exponentiation without materializing the enormous ordinary power.

## When to Use

Use this test for isolated non-negative machine-word values when trial division
to their square roots would be too slow. It complements a sieve, which is
usually better when every prime in a dense interval is required.

Use a number-theory or cryptographic library when inputs are wider than 64 bits,
when a proof certificate is required, or when primes must be generated under a
security policy. A primality predicate alone does not establish that a key or
modulus is safe.

## Implementation

```python
_MAX_UINT64 = (1 << 64) - 1
_SMALL_PRIMES = (2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37)
_UINT64_MILLER_RABIN_BASES = (
    2,
    325,
    9_375,
    28_178,
    450_775,
    9_780_504,
    1_795_265_022,
)


def is_prime_uint64(value: int) -> bool:
    """Return whether one exact unsigned 64-bit integer is prime."""
    if type(value) is not int:
        raise TypeError("value must be an exact integer")
    if not 0 <= value <= _MAX_UINT64:
        raise ValueError("value is outside the unsigned 64-bit range")
    if value < 2:
        return False

    for prime in _SMALL_PRIMES:
        if value == prime:
            return True
        if value % prime == 0:
            return False

    odd_part = value - 1
    power_of_two = 0
    while odd_part % 2 == 0:
        odd_part //= 2
        power_of_two += 1

    for witness in _UINT64_MILLER_RABIN_BASES:
        base = witness % value
        if base in (0, 1):
            continue

        residue = pow(base, odd_part, value)
        if residue in (1, value - 1):
            continue

        for _ in range(power_of_two - 1):
            residue = pow(residue, 2, value)
            if residue == value - 1:
                break
        else:
            return False

    return True
```

## Example

```python
largest_uint64_prime = 18_446_744_073_709_551_557
large_32_bit_prime = 4_294_967_291
strong_pseudoprimes = (
    3_215_031_751,
    341_550_071_728_321,
    3_825_123_056_546_413_051,
)

assert [is_prime_uint64(value) for value in range(12)] == [
    False,
    False,
    True,
    True,
    False,
    True,
    False,
    True,
    False,
    False,
    False,
    True,
]
assert is_prime_uint64(largest_uint64_prime)
assert is_prime_uint64(large_32_bit_prime)
assert not is_prime_uint64(_MAX_UINT64)
assert not is_prime_uint64(large_32_bit_prime**2)
assert not any(is_prime_uint64(value) for value in strong_pseudoprimes)
```

## Trade-offs and Limitations

The fixed seven rounds each perform `O(log n)` modular
multiplications/squarings and retain `O(1)` Python-integer objects. Those
operations are not constant-time, and their bit cost grows with the candidate.
Small-prime division avoids unnecessary rounds for common composites.

The deterministic claim is limited to `0 <= n < 2**64` and the exact bases
shown here, as stated by the published
[finite-range result](https://ceur-ws.org/Vol-1326/020-Forisek.pdf).
Changing the range or using a shorter witness list can admit strong
pseudoprimes. Every witness is reduced modulo `n` before its round.

The function returns no factor, proof certificate, probable-prime confidence,
or explanation. Its control flow and modular arithmetic are not constant-time.
It is not an arbitrary-precision deterministic test and is not a complete
cryptographic prime-generation procedure.

## Related Snippets

<!-- catalog:related:start -->
- [Enumerate Primes in a Bounded Half-Open Range with a Segmented Sieve](enumerate-primes-in-a-bounded-half-open-range-with-a-segmented-sieve.md)
- [Combine a Bounded System of Possibly Non-Coprime Integer Congruences](combine-a-bounded-system-of-possibly-non-coprime-integer-congruences.md)
- [Compute a Distant Linear-Recurrence Term Modulo an Integer by Polynomial Reduction](compute-a-distant-linear-recurrence-term-modulo-an-integer-by-polynomial-reduction.md)
<!-- catalog:related:end -->
