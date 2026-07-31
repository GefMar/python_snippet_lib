---
title: "Sum a Bounded Signed Affine Floor Sequence with Euclidean Reduction"
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
  - compute-canonical-bezout-coefficients-for-two-bounded-integers.md
  - solve-a-bounded-linear-congruence-as-a-canonical-residue-class.md
  - apportion-a-bounded-integer-total-by-exact-weights-with-largest-remainders.md
---

# Sum a Bounded Signed Affine Floor Sequence with Euclidean Reduction

## Idea and Problem

Compute an exact signed affine floor sum without iterating over every term.

The target is
`sum((slope * index + intercept) // modulus for index in range(count))`.
The count can be far too large for direct iteration, but successive quotient
and remainder reductions turn the remaining problem into a smaller one, much
like Euclid's algorithm.

Python floor division rounds negative quotients toward negative infinity.
Using `divmod()` with a positive modulus separates each signed coefficient
into an integer quotient and a non-negative remainder. The quotient parts have
closed-form sums; the Euclidean loop only needs to process the two remainders.

## When to Use

Use this function when a fixed positive modulus divides one signed affine
integer expression over a very large consecutive index range. It can aggregate
exact bucket indexes, count grid positions beneath a rational boundary when
the domain makes the terms counts, or support bounded number-theoretic
calculations without floating-point rounding.

Use a direct generator expression when the count is small and clarity matters
more than asymptotic work. Choose a domain-specific recurrence or counting
algorithm when the numerator is nonlinear, the modulus changes by term, or
the caller needs every individual quotient instead of their sum.

## Implementation

```python
_MAX_AFFINE_FLOOR_INPUT = 10**18


def _sum_nonnegative_affine_floors(
    count: int,
    modulus: int,
    slope: int,
    intercept: int,
) -> int:
    """Sum normalized floors with non-negative slope and intercept."""
    total = 0
    while True:
        slope_quotient, slope = divmod(slope, modulus)
        total += slope_quotient * count * (count - 1) // 2

        intercept_quotient, intercept = divmod(intercept, modulus)
        total += intercept_quotient * count

        transformed_height = slope * count + intercept
        if transformed_height < modulus:
            return total

        count, intercept = divmod(transformed_height, modulus)
        slope, modulus = modulus, slope


def floor_sum_signed(
    count: int,
    modulus: int,
    slope: int,
    intercept: int,
) -> int:
    """Return the exact sum of the bounded signed affine floor sequence."""
    if type(count) is not int:
        raise TypeError("count must be an exact non-boolean integer")
    if not 0 <= count <= _MAX_AFFINE_FLOOR_INPUT:
        raise ValueError("count is outside the supported range")

    if type(modulus) is not int:
        raise TypeError("modulus must be an exact non-boolean integer")
    if not 1 <= modulus <= _MAX_AFFINE_FLOOR_INPUT:
        raise ValueError("modulus is outside the supported range")

    if type(slope) is not int:
        raise TypeError("slope must be an exact non-boolean integer")
    if not -_MAX_AFFINE_FLOOR_INPUT <= slope <= _MAX_AFFINE_FLOOR_INPUT:
        raise ValueError("slope is outside the supported range")

    if type(intercept) is not int:
        raise TypeError("intercept must be an exact non-boolean integer")
    if not -_MAX_AFFINE_FLOOR_INPUT <= intercept <= _MAX_AFFINE_FLOOR_INPUT:
        raise ValueError("intercept is outside the supported range")

    slope_quotient, normalized_slope = divmod(slope, modulus)
    intercept_quotient, normalized_intercept = divmod(intercept, modulus)
    index_sum = count * (count - 1) // 2

    return (
        slope_quotient * index_sum
        + intercept_quotient * count
        + _sum_nonnegative_affine_floors(
            count,
            modulus,
            normalized_slope,
            normalized_intercept,
        )
    )
```

## Example

```python
def sum_signed_affine_floors_directly(
    count: int,
    modulus: int,
    slope: int,
    intercept: int,
) -> int:
    return sum(
        (slope * index + intercept) // modulus
        for index in range(count)
    )


checked = 0
for small_count in range(8):
    for small_modulus in range(1, 9):
        for small_slope in range(-9, 10):
            for small_intercept in range(-9, 10):
                assert floor_sum_signed(
                    small_count,
                    small_modulus,
                    small_slope,
                    small_intercept,
                ) == sum_signed_affine_floors_directly(
                    small_count,
                    small_modulus,
                    small_slope,
                    small_intercept,
                )
                checked += 1

from random import Random


generator = Random(0xF100_5A7)
for _ in range(2_000):
    random_count = generator.randrange(31)
    random_modulus = generator.randrange(1, 32)
    random_slope = generator.randrange(-100, 101)
    random_intercept = generator.randrange(-100, 101)
    assert floor_sum_signed(
        random_count,
        random_modulus,
        random_slope,
        random_intercept,
    ) == sum_signed_affine_floors_directly(
        random_count,
        random_modulus,
        random_slope,
        random_intercept,
    )
    checked += 1

metamorphic_count = 127
metamorphic_modulus = 37
metamorphic_slope = -29
metamorphic_intercept = 41
base = floor_sum_signed(
    metamorphic_count,
    metamorphic_modulus,
    metamorphic_slope,
    metamorphic_intercept,
)
index_sum = metamorphic_count * (metamorphic_count - 1) // 2
for multiple in (-2, -1, 1, 3):
    assert floor_sum_signed(
        metamorphic_count,
        metamorphic_modulus,
        metamorphic_slope + multiple * metamorphic_modulus,
        metamorphic_intercept,
    ) == base + multiple * index_sum
    assert floor_sum_signed(
        metamorphic_count,
        metamorphic_modulus,
        metamorphic_slope,
        metamorphic_intercept + multiple * metamorphic_modulus,
    ) == base + multiple * metamorphic_count

maximum = _MAX_AFFINE_FLOOR_INPUT
maximum_index_sum = maximum * (maximum - 1) // 2
assert floor_sum_signed(maximum, 1, maximum, -maximum) == (
    maximum * maximum_index_sum - maximum * maximum
)
assert floor_sum_signed(0, maximum, -maximum, maximum) == 0
assert -1 // 2 == -1
assert floor_sum_signed(3, 2, -1, -1) == -4

huge_modulus = maximum - 1
huge_slope = -maximum
huge_intercept = maximum
assert floor_sum_signed(
    maximum,
    huge_modulus,
    huge_slope,
    huge_intercept,
) - floor_sum_signed(
    maximum - 1,
    huge_modulus,
    huge_slope,
    huge_intercept,
) == (huge_slope * (maximum - 1) + huge_intercept) // huge_modulus

invalid_calls = (
    lambda: floor_sum_signed(True, 1, 0, 0),
    lambda: floor_sum_signed(0, True, 0, 0),
    lambda: floor_sum_signed(0, 1, False, 0),
    lambda: floor_sum_signed(0, 1, 0, False),
    lambda: floor_sum_signed(-1, 1, 0, 0),
    lambda: floor_sum_signed(maximum + 1, 1, 0, 0),
    lambda: floor_sum_signed(0, 0, 0, 0),
    lambda: floor_sum_signed(0, maximum + 1, 0, 0),
    lambda: floor_sum_signed(0, 1, maximum + 1, 0),
    lambda: floor_sum_signed(0, 1, 0, -maximum - 1),
)
rejected = 0
for invalid_call in invalid_calls:
    try:
        invalid_call()
    except (TypeError, ValueError):
        rejected += 1

assert checked == 25_104
assert rejected == len(invalid_calls)
```

## Trade-offs and Limitations

The Euclidean reduction uses `O(log(modulus + 1))` loop iterations and retains
only a constant number of Python integers. Integer multiplication and division
are not constant-cost operations, so their bit complexity grows with the
accepted inputs and with an exact result that can be much larger than any one
argument.

All four arguments are exact non-Boolean integers. The count and positive
modulus are at most `10^18`; the signed slope and intercept have absolute value
at most `10^18`. A zero count still validates the other arguments before
returning zero.

The result follows Python's `//` rule for negative numerators, which rounds
toward negative infinity rather than truncating toward zero. The function does
not use floats, wrap the result modulo another integer, return individual
terms, accept a non-positive or varying modulus, or handle nonlinear
numerators.

## Related Snippets

<!-- catalog:related:start -->
- [Compute Canonical Bézout Coefficients for Two Bounded Integers](compute-canonical-bezout-coefficients-for-two-bounded-integers.md)
- [Solve a Bounded Linear Congruence as a Canonical Residue Class](solve-a-bounded-linear-congruence-as-a-canonical-residue-class.md)
- [Apportion a Bounded Integer Total by Exact Weights with Largest Remainders](apportion-a-bounded-integer-total-by-exact-weights-with-largest-remainders.md)
<!-- catalog:related:end -->
