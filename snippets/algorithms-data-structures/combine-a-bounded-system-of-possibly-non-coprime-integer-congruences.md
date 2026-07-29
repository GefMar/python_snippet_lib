---
title: "Combine a Bounded System of Possibly Non-Coprime Integer Congruences"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../networking-protocols/unwrap-one-uint32-serial-around-an-explicit-absolute-reference.md
---

# Combine a Bounded System of Possibly Non-Coprime Integer Congruences

## Idea and Problem

Combine several integer remainder constraints into one canonical congruence, including compatible moduli that share factors.

Normalize every remainder, then merge constraints incrementally. Two current
congruences are compatible exactly when their residue difference is divisible
by the greatest common divisor of their moduli. Dividing out that common factor
makes a modular inverse available and produces the next canonical residue.

## When to Use

Use this algorithm when one bounded in-memory calculation must satisfy several
periodic integer conditions and the moduli are not guaranteed to be coprime.
The canonical result is useful for deterministic scheduling calculations,
cycle alignment, or fixtures that need the first non-negative representative.

Use domain-specific solvers when constraints include inequalities or ranges.
Do not treat this arithmetic construction as cryptography, and reject a result
space larger than the caller can safely retain or search rather than relying on
Python's unbounded integers alone.

## Implementation

```python
from math import gcd

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_CONGRUENCES = 32
_MAX_MODULUS = 1_000_000_000
_MAX_COMBINED_MODULUS = 10**18


def combine_integer_congruences(
    congruences: tuple[tuple[int, int], ...],
) -> tuple[int, int] | None:
    """Return one canonical congruence, or None when the system conflicts."""
    if type(congruences) is not tuple:
        raise TypeError("congruences must be an exact tuple")
    if not 1 <= len(congruences) <= _MAX_CONGRUENCES:
        raise ValueError("congruence count is outside the supported range")

    combined_modulus = 1
    for index, congruence in enumerate(congruences):
        if type(congruence) is not tuple:
            raise TypeError(f"congruences[{index}] must be an exact tuple")
        if len(congruence) != 2:
            raise ValueError(f"congruences[{index}] must contain two items")

        remainder, modulus = congruence
        if type(remainder) is not int:
            raise TypeError(f"congruences[{index}].remainder must be an exact integer")
        if not _MIN_INT64 <= remainder <= _MAX_INT64:
            raise ValueError(f"congruences[{index}].remainder is outside the signed 64-bit range")
        if type(modulus) is not int:
            raise TypeError(f"congruences[{index}].modulus must be an exact integer")
        if not 1 <= modulus <= _MAX_MODULUS:
            raise ValueError(f"congruences[{index}].modulus is outside the supported range")

        new_factor = modulus // gcd(combined_modulus, modulus)
        if combined_modulus > _MAX_COMBINED_MODULUS // new_factor:
            raise ValueError("combined modulus exceeds the supported limit")
        combined_modulus *= new_factor

    residue = 0
    current_modulus = 1
    for remainder, modulus in congruences:
        normalized = remainder % modulus
        common_divisor = gcd(current_modulus, modulus)
        difference = normalized - residue
        if difference % common_divisor:
            return None

        reduced_modulus = modulus // common_divisor
        if reduced_modulus == 1:
            multiplier = 0
        else:
            inverse = pow(
                current_modulus // common_divisor,
                -1,
                reduced_modulus,
            )
            multiplier = (difference // common_divisor * inverse) % reduced_modulus

        next_combined_modulus = current_modulus * reduced_modulus
        residue = (residue + current_modulus * multiplier) % next_combined_modulus
        current_modulus = next_combined_modulus

    return residue, combined_modulus
```

## Example

```python
def brute_congruences(
    congruences: tuple[tuple[int, int], ...],
) -> tuple[int, int] | None:
    from math import lcm

    combined = lcm(*(modulus for _, modulus in congruences))
    for candidate in range(combined):
        if all(candidate % modulus == remainder % modulus for remainder, modulus in congruences):
            return candidate, combined
    return None


def exercise_small_systems() -> int:
    from itertools import product

    options = tuple((remainder, modulus) for modulus in range(1, 6) for remainder in range(-2, 3))
    checked = 0
    for item_count in range(1, 4):
        for system in product(options, repeat=item_count):
            expected = brute_congruences(system)
            assert combine_integer_congruences(system) == expected
            assert combine_integer_congruences(tuple(reversed(system))) == expected
            checked += 1
    return checked


def results_for_every_order(
    congruences: tuple[tuple[int, int], ...],
) -> set[tuple[int, int] | None]:
    from itertools import permutations

    return {combine_integer_congruences(order) for order in permutations(congruences)}


checked_systems = exercise_small_systems()
normalization_system = ((-1, 5), (9, 7), (0, 3))
permutation_results = results_for_every_order(normalization_system)

try:
    combine_integer_congruences(((0, 1_000_000_000), (0, 999_999_937), (0, 999_999_929)))
except ValueError:
    oversized_lcm_rejected = True
else:
    oversized_lcm_rejected = False

try:
    combine_integer_congruences(((True, 2),))
except TypeError:
    boolean_rejected = True
else:
    boolean_rejected = False

assert (
    checked_systems,
    combine_integer_congruences(((2, 6), (5, 9))),
    combine_integer_congruences(((0, 2), (1, 4))),
    combine_integer_congruences(((123, 1), (-1, 5))),
    permutation_results,
    oversized_lcm_rejected,
    boolean_rejected,
) == (
    16_275,
    (14, 18),
    None,
    (4, 5),
    {(9, 105)},
    True,
    True,
)
```

## Trade-offs and Limitations

For `K` constraints and bounded combined modulus `L`, validation and merging
use `O(K log L)` integer-arithmetic time and `O(1)` incremental state. The
first pass deliberately validates the order-independent least common multiple
before compatibility is considered, so an oversized system is rejected even
when a later pair would also conflict.

The returned modulus is the combined least common multiple. Its residue is the
unique representative from zero through one less than that modulus. A valid
but incompatible system returns `None`, while malformed or over-budget input
raises an exception. Modulus one is supported explicitly and adds no
restriction.

The function does not accept inequalities, negative or zero moduli, floating
or approximate arithmetic, unbounded result spaces, solution enumeration, or
cryptographic claims.

## Related Snippets

<!-- catalog:related:start -->
- [Unwrap One uint32 Serial Around an Explicit Absolute Reference](../networking-protocols/unwrap-one-uint32-serial-around-an-explicit-absolute-reference.md)
<!-- catalog:related:end -->
