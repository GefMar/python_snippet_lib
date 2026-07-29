---
title: "Enumerate Primes in a Bounded Half-Open Range with a Segmented Sieve"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - combine-a-bounded-system-of-possibly-non-coprime-integer-congruences.md
  - compute-a-distant-linear-recurrence-term-modulo-an-integer-by-polynomial-reduction.md
---

# Enumerate Primes in a Bounded Half-Open Range with a Segmented Sieve

## Idea and Problem

Enumerate every prime in a bounded non-negative half-open integer range without allocating one flag for every value below its upper endpoint.

An ordinary sieve first finds the primes through the integer square root of the
largest candidate. Each base prime then marks its multiples only inside the
requested segment. Starting at the greater of its square and its first multiple
in the segment avoids incorrectly marking the base prime itself.

The half-open contract makes an empty range unambiguous and composes naturally
with adjacent segments. Independent limits on the upper endpoint and range
width bound both the base sieve and the segment allocation.

## When to Use

Use a segmented sieve for one bounded range whose upper endpoint is much larger
than its width, when every prime in that range is required in ascending order.
It is also useful for deterministic fixtures because it uses only exact integer
arithmetic and `bytearray` flags.

Use a monolithic sieve when all primes from zero through a moderate limit will
be reused. Use a primality test when only a few unrelated values must be
classified, and use a specialized number-theory library for substantially
larger endpoints, parallel segments, probabilistic tests, or prime counting
without enumeration.

## Implementation

```python
from math import isqrt

_MAX_PRIME_RANGE_STOP = 10**12
_MAX_PRIME_RANGE_WIDTH = 1_000_000


def enumerate_primes_in_range(start: int, stop: int) -> tuple[int, ...]:
    """Return every prime in the non-negative half-open range [start, stop)."""
    if type(start) is not int or type(stop) is not int:
        raise TypeError("range endpoints must be exact integers")
    if not 0 <= start <= stop:
        raise ValueError("range endpoints must satisfy 0 <= start <= stop")
    if stop > _MAX_PRIME_RANGE_STOP:
        raise ValueError("stop exceeds the supported limit")
    if stop - start > _MAX_PRIME_RANGE_WIDTH:
        raise ValueError("range width exceeds the supported limit")

    if stop <= 2:
        return ()
    segment_start = max(start, 2)
    segment_width = stop - segment_start
    if segment_width == 0:
        return ()

    base_limit = isqrt(stop - 1)
    base_flags = bytearray(b"\x01") * (base_limit + 1)
    base_flags[0:2] = b"\x00\x00"
    for candidate in range(2, isqrt(base_limit) + 1):
        if not base_flags[candidate]:
            continue
        first = candidate * candidate
        mark_count = (base_limit - first) // candidate + 1
        base_flags[first : base_limit + 1 : candidate] = b"\x00" * mark_count

    segment_flags = bytearray(b"\x01") * segment_width
    for prime in range(2, base_limit + 1):
        if not base_flags[prime]:
            continue
        first = max(
            prime * prime,
            ((segment_start + prime - 1) // prime) * prime,
        )
        if first >= stop:
            continue
        offset = first - segment_start
        mark_count = (segment_width - 1 - offset) // prime + 1
        segment_flags[offset:segment_width:prime] = b"\x00" * mark_count

    return tuple(
        segment_start + offset for offset, is_prime in enumerate(segment_flags) if is_prime
    )
```

## Example

```python
def trial_primes(start: int, stop: int) -> tuple[int, ...]:
    from math import isqrt

    return tuple(
        value
        for value in range(max(start, 2), stop)
        if all(value % divisor for divisor in range(2, isqrt(value) + 1))
    )


def monolithic_primes(stop: int) -> tuple[int, ...]:
    flags = [True] * stop
    if stop > 0:
        flags[0] = False
    if stop > 1:
        flags[1] = False
    for candidate in range(2, stop):
        if not flags[candidate]:
            continue
        for composite in range(candidate * candidate, stop, candidate):
            flags[composite] = False
    return tuple(value for value, is_prime in enumerate(flags) if is_prime)


def exercise_small_ranges() -> int:
    expected = monolithic_primes(80)
    checked = 0
    for start in range(81):
        for stop in range(start, 81):
            assert enumerate_primes_in_range(start, stop) == tuple(
                prime for prime in expected if start <= prime < stop
            )
            checked += 1
    return checked


million_primes = enumerate_primes_in_range(0, 1_000_000)
high_start = _MAX_PRIME_RANGE_STOP - 40
high_primes = enumerate_primes_in_range(high_start, _MAX_PRIME_RANGE_STOP)

type_errors = 0
try:
    enumerate_primes_in_range(False, 10)
except TypeError:
    type_errors += 1

value_errors = 0
for invalid_start, invalid_stop in (
    (-1, 10),
    (5, 4),
    (_MAX_PRIME_RANGE_STOP, _MAX_PRIME_RANGE_STOP + 1),
    (0, _MAX_PRIME_RANGE_WIDTH + 1),
):
    try:
        enumerate_primes_in_range(invalid_start, invalid_stop)
    except ValueError:
        value_errors += 1

assert (
    exercise_small_ranges(),
    enumerate_primes_in_range(0, 2),
    enumerate_primes_in_range(2, 3),
    enumerate_primes_in_range(48, 50),
    len(million_primes),
    million_primes[-1],
    high_primes == trial_primes(high_start, _MAX_PRIME_RANGE_STOP),
    type_errors,
    value_errors,
) == (
    3_321,
    (),
    (2,),
    (),
    78_498,
    999_983,
    True,
    1,
    4,
)
```

## Trade-offs and Limitations

Let `L = isqrt(stop - 1)` and `W = stop - start`. After the constant-time
`stop <= 2` return, base sieving and segment marking take
`O((L + W) * (1 + log log(L + 2)))` integer-index operations. This form keeps
the bound meaningful for small `L` and, unlike a width-only claim, includes the
work needed to build and visit the base-prime table for a narrow range near the
maximum endpoint. Base flags, segment flags, and the returned `r` primes use
`O(L + W + r)` memory.

The function treats zero and one as non-prime, includes `start` when it is
prime, and always excludes `stop`. The ceiling multiple calculation works for
the declared non-negative endpoints, while `prime * prime` prevents a base
prime inside the segment from marking itself. Python integers make those
products exact under the fixed endpoint limit.

The implementation rebuilds base primes for every call and returns all matches
as one tuple. It does not cache base flags, stream a wider range in successive
segments, accept negative endpoints, count without enumerating, factor values,
prove primality beyond the declared bound, parallelize marking, or support an
upper endpoint or width larger than the explicit limits.

## Related Snippets

<!-- catalog:related:start -->
- [Combine a Bounded System of Possibly Non-Coprime Integer Congruences](combine-a-bounded-system-of-possibly-non-coprime-integer-congruences.md)
- [Compute a Distant Linear-Recurrence Term Modulo an Integer by Polynomial Reduction](compute-a-distant-linear-recurrence-term-modulo-an-integer-by-polynomial-reduction.md)
<!-- catalog:related:end -->
