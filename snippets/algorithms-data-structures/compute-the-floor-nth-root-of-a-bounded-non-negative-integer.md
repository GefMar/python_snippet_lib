---
title: "Compute the Floor Nth Root of a Bounded Non-Negative Integer"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - factor-a-bounded-positive-integer-by-deterministic-trial-division.md
  - test-unsigned-64-bit-integers-for-primality-with-deterministic-miller-rabin.md
  - approximate-a-bounded-fraction-under-a-denominator-limit-with-exact-error.md
---

# Compute the Floor Nth Root of a Bounded Non-Negative Integer

## Idea and Problem

Compute the greatest integer whose exact positive-degree power does not exceed one bounded non-negative integer.

A value's bit length gives an exclusive power-of-two upper bound for its root.
Binary search can then keep a known feasible lower bound and an infeasible
upper bound while comparing exact integer powers. This avoids floating-point
rounding near very large perfect powers.

## When to Use

Use this function when a non-negative Python integer needs an exact floor root
for validation, integer-only scaling, or perfect-power boundary checks. The
explicit bit cap makes the cost of every exponentiation and intermediate
integer predictable enough for bounded in-memory work.

Use `math.isqrt()` for square roots because the standard-library specialization
is shorter and faster. Use decimal or numerical methods when an approximation
is required, and choose a different contract when negative odd roots or a
fractional result must be represented.

## Implementation

```python
_MAX_ROOT_VALUE_BITS = 4_096
_MAX_ROOT_DEGREE = 4_096


def floor_nth_root(value: int, degree: int) -> int:
    """Return the greatest root whose exact degree-th power is at most value."""
    if type(value) is not int:
        raise TypeError("value must be an exact integer")
    if type(degree) is not int:
        raise TypeError("degree must be an exact integer")
    if value < 0:
        raise ValueError("value must be non-negative")
    if value.bit_length() > _MAX_ROOT_VALUE_BITS:
        raise ValueError("value exceeds the supported bit length")
    if not 1 <= degree <= _MAX_ROOT_DEGREE:
        raise ValueError("degree is outside 1..4096")

    if value < 2 or degree == 1:
        return value

    root_bit_bound = (value.bit_length() + degree - 1) // degree
    lower = 1
    upper = 1 << root_bit_bound

    while lower + 1 < upper:
        candidate = (lower + upper) // 2
        if pow(candidate, degree) <= value:
            lower = candidate
        else:
            upper = candidate
    return lower
```

## Example

```python
def floor_nth_root_by_search(value: int, degree: int) -> int:
    candidate = 0
    while pow(candidate + 1, degree) <= value:
        candidate += 1
    return candidate


def exercise_small_values() -> int:
    checked = 0
    for value in range(513):
        for degree in range(1, 13):
            assert floor_nth_root(value, degree) == floor_nth_root_by_search(
                value,
                degree,
            )
            checked += 1
    return checked


def exercise_power_boundaries() -> int:
    checked = 0
    for degree in (2, 3, 5, 17, 64):
        for root in (1, 2, 3, 17, 257):
            exact_power = pow(root, degree)
            assert floor_nth_root(exact_power - 1, degree) == root - 1
            assert floor_nth_root(exact_power, degree) == root
            assert floor_nth_root(exact_power + 1, degree) == root
            checked += 1
    return checked


maximum_value = (1 << _MAX_ROOT_VALUE_BITS) - 1

rejected = 0
for invalid_value, invalid_degree in (
    (True, 2),
    (-1, 3),
    (1 << _MAX_ROOT_VALUE_BITS, 2),
    (9, True),
    (9, 0),
    (9, _MAX_ROOT_DEGREE + 1),
):
    try:
        floor_nth_root(invalid_value, invalid_degree)
    except (TypeError, ValueError):
        rejected += 1

assert (
    exercise_small_values(),
    exercise_power_boundaries(),
    floor_nth_root(maximum_value, 1),
    floor_nth_root(maximum_value, 2),
    floor_nth_root(maximum_value, _MAX_ROOT_DEGREE),
    rejected,
) == (
    6_156,
    25,
    maximum_value,
    (1 << 2_048) - 1,
    1,
    6,
)
```

## Trade-offs and Limitations

Let `B` be the value's bit length and `D` the degree. The search performs
`O(ceil(B / D))` exact power probes, which is logarithmic in the possible root
range. Each probe has operand-dependent integer-exponentiation cost; Python's
arbitrary-precision multiplication and the temporary power are not
constant-cost. The admitted bit and degree caps also bound those intermediate
values.

The result is a floor, so a non-perfect power has an unreported remainder.
The function accepts neither negative values nor zero or negative degrees. It
does not return rational or approximate roots, test whether the value is a
perfect power, decompose powers, expose intermediate bounds, or specialize the
square-root case.

## Related Snippets

<!-- catalog:related:start -->
- [Factor a Bounded Positive Integer by Deterministic Trial Division](factor-a-bounded-positive-integer-by-deterministic-trial-division.md)
- [Test Unsigned 64-Bit Integers for Primality with Deterministic Miller-Rabin](test-unsigned-64-bit-integers-for-primality-with-deterministic-miller-rabin.md)
- [Approximate a Bounded Fraction under a Denominator Limit with Exact Error](approximate-a-bounded-fraction-under-a-denominator-limit-with-exact-error.md)
<!-- catalog:related:end -->
