---
title: "Apply Divisor Zeta and Möbius Transforms Modulo an Integer"
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
  - apply-bounded-subset-zeta-and-mobius-transforms-modulo-an-integer.md
  - compute-eulers-totient-function-through-a-bounded-limit-with-a-linear-sieve.md
  - convolve-bounded-subset-functions-modulo-an-integer.md
---

# Apply Divisor Zeta and Möbius Transforms Modulo an Integer

## Idea and Problem

Aggregate a dense arithmetic-function table over divisors, or invert those aggregates, without testing every possible divisor for every output.

For values attached to the integers `1` through `N`, the divisor zeta
transform at `n` is the sum of the original values at every divisor of
`n`. Scattering each original value to its multiples visits exactly the
required pairs. Möbius inversion reverses this triangular relation by
recovering integers in ascending order and subtracting each recovered value
from its proper multiples.

Both operations use only addition and subtraction modulo an explicit integer.
They therefore remain exact for composite as well as prime moduli.

## When to Use

Use these transforms when a calculation needs divisor sums for every integer
in one dense bounded prefix, or must recover exact-per-integer values from such
sums. Examples include arithmetic-function preprocessing, multiplicative
dynamic programs, and checking divisor-convolution identities.

Use direct divisor enumeration for only a few queried integers, and use a
sparse representation when most of the prefix is absent. A subset zeta
transform is a different operation: its indices are bitmasks ordered by set
containment rather than positive integers ordered by divisibility.

## Implementation

```python
_MAX_DIVISOR_TRANSFORM_SIZE = 1 << 15
_MAX_DIVISOR_TRANSFORM_MODULUS = (1 << 31) - 1


def _validate_divisor_table(
    name: str,
    values: tuple[int, ...],
    modulus: int,
) -> None:
    if type(values) is not tuple:
        raise TypeError(f"{name} must be an exact tuple")
    if type(modulus) is not int:
        raise TypeError("modulus must be an exact integer")
    if not 2 <= modulus <= _MAX_DIVISOR_TRANSFORM_MODULUS:
        raise ValueError("modulus is outside 2..2^31-1")
    if not 1 <= len(values) <= _MAX_DIVISOR_TRANSFORM_SIZE:
        raise ValueError(f"{name} must contain 1..32,768 residues")

    for index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"{name}[{index}] must be an exact integer")
        if not 0 <= value < modulus:
            raise ValueError(f"{name}[{index}] is not a canonical residue")


def divisor_zeta(
    values: tuple[int, ...],
    modulus: int,
) -> tuple[int, ...]:
    """Return each sum over positive divisors, reduced modulo modulus."""
    _validate_divisor_table("values", values, modulus)
    transformed = [0] * len(values)

    for divisor, value in enumerate(values, start=1):
        for multiple in range(divisor, len(values) + 1, divisor):
            index = multiple - 1
            transformed[index] = (transformed[index] + value) % modulus

    return tuple(transformed)


def divisor_mobius(
    aggregates: tuple[int, ...],
    modulus: int,
) -> tuple[int, ...]:
    """Invert divisor_zeta over integers modulo modulus."""
    _validate_divisor_table("aggregates", aggregates, modulus)
    transformed = list(aggregates)

    for divisor in range(1, len(transformed) + 1):
        recovered = transformed[divisor - 1]
        for multiple in range(
            2 * divisor,
            len(transformed) + 1,
            divisor,
        ):
            index = multiple - 1
            transformed[index] = (transformed[index] - recovered) % modulus

    return tuple(transformed)
```

## Example

```python
from itertools import product


def direct_divisor_zeta(
    values: tuple[int, ...],
    modulus: int,
) -> tuple[int, ...]:
    return tuple(
        sum(values[divisor - 1] for divisor in range(1, integer + 1) if integer % divisor == 0)
        % modulus
        for integer in range(1, len(values) + 1)
    )


def direct_divisor_mobius(
    aggregates: tuple[int, ...],
    modulus: int,
) -> tuple[int, ...]:
    recovered: list[int] = []
    for integer, aggregate in enumerate(aggregates, start=1):
        proper_divisor_sum = sum(
            recovered[divisor - 1] for divisor in range(1, integer) if integer % divisor == 0
        )
        recovered.append((aggregate - proper_divisor_sum) % modulus)
    return tuple(recovered)


examined = 0
for small_modulus in (2, 3, 4, 6):
    for size in range(1, 7):
        alphabet = range(small_modulus) if size <= 3 else range(min(small_modulus, 2))
        for small_values in product(alphabet, repeat=size):
            expected_zeta = direct_divisor_zeta(
                small_values,
                small_modulus,
            )
            expected_mobius = direct_divisor_mobius(
                small_values,
                small_modulus,
            )
            assert (
                divisor_zeta(
                    small_values,
                    small_modulus,
                )
                == expected_zeta
            )
            assert (
                divisor_mobius(
                    small_values,
                    small_modulus,
                )
                == expected_mobius
            )
            assert (
                divisor_mobius(
                    expected_zeta,
                    small_modulus,
                )
                == small_values
            )
            assert (
                divisor_zeta(
                    expected_mobius,
                    small_modulus,
                )
                == small_values
            )
            examined += 1

sample = (1, 2, 3, 4, 5, 6)
sample_zeta = divisor_zeta(sample, 12)
assert sample_zeta == (1, 3, 4, 7, 6, 0)
assert divisor_mobius(sample_zeta, 12) == sample

ones = (1,) * 10
assert divisor_zeta(ones, 12) == (1, 2, 2, 3, 2, 4, 2, 4, 3, 4)
assert divisor_mobius(ones, 12) == (1,) + (0,) * 9

impulse_at_three = (0, 0, 1) + (0,) * 7
assert divisor_zeta(impulse_at_three, 7) == tuple(int(integer % 3 == 0) for integer in range(1, 11))

maximum_impulse = (1,) + (0,) * (_MAX_DIVISOR_TRANSFORM_SIZE - 1)
maximum_ones = (1,) * _MAX_DIVISOR_TRANSFORM_SIZE
assert (
    divisor_zeta(
        maximum_impulse,
        _MAX_DIVISOR_TRANSFORM_MODULUS,
    )
    == maximum_ones
)
assert (
    divisor_mobius(
        maximum_ones,
        _MAX_DIVISOR_TRANSFORM_MODULUS,
    )
    == maximum_impulse
)
assert divisor_zeta(
    (_MAX_DIVISOR_TRANSFORM_MODULUS - 1,),
    _MAX_DIVISOR_TRANSFORM_MODULUS,
) == (_MAX_DIVISOR_TRANSFORM_MODULUS - 1,)

invalid_calls = (
    (lambda: divisor_zeta([], 5), TypeError),
    (lambda: divisor_zeta((), 5), ValueError),
    (
        lambda: divisor_zeta(
            (0,) * (_MAX_DIVISOR_TRANSFORM_SIZE + 1),
            5,
        ),
        ValueError,
    ),
    (lambda: divisor_zeta((0, True), 5), TypeError),
    (lambda: divisor_zeta((0, -1), 5), ValueError),
    (lambda: divisor_zeta((0, 5), 5), ValueError),
    (lambda: divisor_zeta((0,), True), TypeError),
    (lambda: divisor_zeta((0,), 1), ValueError),
    (
        lambda: divisor_zeta(
            (0,),
            _MAX_DIVISOR_TRANSFORM_MODULUS + 1,
        ),
        ValueError,
    ),
    (lambda: divisor_mobius([], 5), TypeError),
)
rejected = 0
for invalid_call, expected_error in invalid_calls:
    try:
        invalid_call()
    except expected_error:
        rejected += 1
    else:
        raise AssertionError("invalid input must be rejected")

assert examined == 843
assert rejected == 10
```

## Trade-offs and Limitations

For `N` input residues, zeta performs
`sum(floor(N / d) for d in 1..N)` modular updates. Inversion omits the
`N` diagonal updates; both therefore take `Theta(N log N)` time and retain
`O(N)` residues. The fixed `N <= 32_768` cap bounds the pure-Python work
but is not a latency promise.

Tuple index `n - 1` represents the positive integer `n`; there is no
sentinel entry for zero. Unlike subset transforms, the tuple length need not
be a power of two. Inputs must already be canonical residues, and results are
always reduced to the same canonical range.

The functions materialize a complete dense prefix. They do not compute or
return the number-theoretic Möbius function itself, infer a modulus, normalize
signed inputs, exploit sparsity, extend a previous table, or evaluate other
forms of convolution.

## Related Snippets

<!-- catalog:related:start -->
- [Apply Bounded Subset Zeta and Möbius Transforms Modulo an Integer](apply-bounded-subset-zeta-and-mobius-transforms-modulo-an-integer.md)
- [Compute Euler's Totient Function Through a Bounded Limit with a Linear Sieve](compute-eulers-totient-function-through-a-bounded-limit-with-a-linear-sieve.md)
- [Convolve Bounded Subset Functions Modulo an Integer](convolve-bounded-subset-functions-modulo-an-integer.md)
<!-- catalog:related:end -->
