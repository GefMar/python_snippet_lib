---
title: "Count and Witness Bounded Solutions of a Two-Variable Linear Diophantine Equation"
snippet_type: algorithm
use_cases:
  - data-transformation
  - testing
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - compute-canonical-bezout-coefficients-for-two-bounded-integers.md
  - solve-a-bounded-linear-congruence-as-a-canonical-residue-class.md
  - sum-a-bounded-signed-affine-floor-sequence-with-euclidean-reduction.md
---

# Count and Witness Bounded Solutions of a Two-Variable Linear Diophantine Equation

## Idea and Problem

Count the integer pairs in one inclusive rectangle that satisfy a two-variable linear equation, and return the lexicographically first pair without enumerating the rectangle.

When both coefficients are nonzero, a greatest-common-divisor test first
decides whether any integer solution exists. One solution then generates the
complete family
`x = x0 + (b / gcd) * t` and `y = y0 - (a / gcd) * t`.
Intersecting the two coordinate bounds produces one closed integer interval
for the parameter `t`; its width is the answer.

The explicit zero-coefficient cases cover unrestricted coordinates without
division by zero. The immutable result always carries both the exact count and
either the first pair or `None`.

## When to Use

Use this function when one exact linear equation must be intersected with
independent inclusive bounds on two integer variables. It is useful for
counting bounded allocations, checking whether a constrained integer balance
is feasible, or obtaining one deterministic witness for later validation.

Use a congruence solver when only one residue class is needed, and use integer
linear programming when there are several equations, coupled inequalities, or
an objective function. Direct enumeration is simpler only for deliberately
tiny rectangles.

## Implementation

```python
from dataclasses import dataclass
from math import gcd

_MIN_SIGNED_64 = -(1 << 63)
_MAX_SIGNED_64 = (1 << 63) - 1


@dataclass(frozen=True, slots=True)
class DiophantineBoxResult:
    count: int
    first: tuple[int, int] | None


def _require_signed_64(name: str, value: int) -> None:
    if type(value) is not int:
        raise TypeError(f"{name} must be an exact integer")
    if not _MIN_SIGNED_64 <= value <= _MAX_SIGNED_64:
        raise ValueError(f"{name} is outside the signed 64-bit range")


def _ceil_div(numerator: int, denominator: int) -> int:
    return -((-numerator) // denominator)


def _bounded_parameter_interval(
    base: int,
    step: int,
    minimum: int,
    maximum: int,
) -> tuple[int, int]:
    if step > 0:
        return (
            _ceil_div(minimum - base, step),
            (maximum - base) // step,
        )
    return (
        _ceil_div(maximum - base, step),
        (minimum - base) // step,
    )


def solve_bounded_diophantine(
    a: int,
    b: int,
    c: int,
    x_min: int,
    x_max: int,
    y_min: int,
    y_max: int,
) -> DiophantineBoxResult:
    """Count bounded solutions and return their lexicographically first pair."""
    for name, value in (
        ("a", a),
        ("b", b),
        ("c", c),
        ("x_min", x_min),
        ("x_max", x_max),
        ("y_min", y_min),
        ("y_max", y_max),
    ):
        _require_signed_64(name, value)

    if x_min > x_max:
        raise ValueError("x_min must not exceed x_max")
    if y_min > y_max:
        raise ValueError("y_min must not exceed y_max")

    x_count = x_max - x_min + 1
    y_count = y_max - y_min + 1

    if a == 0 and b == 0:
        if c != 0:
            return DiophantineBoxResult(0, None)
        return DiophantineBoxResult(x_count * y_count, (x_min, y_min))

    if a == 0:
        y, remainder = divmod(c, b)
        if remainder or not y_min <= y <= y_max:
            return DiophantineBoxResult(0, None)
        return DiophantineBoxResult(x_count, (x_min, y))

    if b == 0:
        x, remainder = divmod(c, a)
        if remainder or not x_min <= x <= x_max:
            return DiophantineBoxResult(0, None)
        return DiophantineBoxResult(y_count, (x, y_min))

    common_divisor = gcd(a, b)
    if c % common_divisor:
        return DiophantineBoxResult(0, None)

    x_modulus = abs(b) // common_divisor
    if x_modulus == 1:
        x_base = 0
    else:
        reduced_a = a // common_divisor
        reduced_c = c // common_divisor
        inverse = pow(reduced_a % x_modulus, -1, x_modulus)
        x_base = (reduced_c * inverse) % x_modulus

    y_base, remainder = divmod(c - a * x_base, b)
    if remainder:
        raise AssertionError("the constructed base solution must be exact")

    x_step = b // common_divisor
    y_step = -(a // common_divisor)
    x_parameter_min, x_parameter_max = _bounded_parameter_interval(
        x_base,
        x_step,
        x_min,
        x_max,
    )
    y_parameter_min, y_parameter_max = _bounded_parameter_interval(
        y_base,
        y_step,
        y_min,
        y_max,
    )

    parameter_min = max(x_parameter_min, y_parameter_min)
    parameter_max = min(x_parameter_max, y_parameter_max)
    if parameter_min > parameter_max:
        return DiophantineBoxResult(0, None)

    first_parameter = parameter_min if x_step > 0 else parameter_max
    first = (
        x_base + x_step * first_parameter,
        y_base + y_step * first_parameter,
    )
    return DiophantineBoxResult(parameter_max - parameter_min + 1, first)
```

## Example

```python
from itertools import product


def brute_bounded_diophantine(
    a: int,
    b: int,
    c: int,
    x_min: int,
    x_max: int,
    y_min: int,
    y_max: int,
) -> DiophantineBoxResult:
    solutions = tuple(
        (x, y)
        for x in range(x_min, x_max + 1)
        for y in range(y_min, y_max + 1)
        if a * x + b * y == c
    )
    return DiophantineBoxResult(
        len(solutions),
        min(solutions) if solutions else None,
    )


small_intervals = tuple(
    (minimum, maximum) for minimum in range(-2, 3) for maximum in range(minimum, 3)
)
checked = 0
for small_a, small_b, small_c in product(range(-3, 4), repeat=3):
    for x_bounds in small_intervals:
        for y_bounds in small_intervals:
            arguments = (
                small_a,
                small_b,
                small_c,
                *x_bounds,
                *y_bounds,
            )
            assert solve_bounded_diophantine(
                *arguments,
            ) == brute_bounded_diophantine(*arguments)
            checked += 1

explicit_cases = (
    (
        (0, 0, 0, -2, 2, -3, 3),
        DiophantineBoxResult(35, (-2, -3)),
    ),
    (
        (0, 0, 1, -2, 2, -3, 3),
        DiophantineBoxResult(0, None),
    ),
    (
        (0, 3, 6, -2, 2, -3, 3),
        DiophantineBoxResult(5, (-2, 2)),
    ),
    (
        (-4, 0, 8, -3, 3, -2, 2),
        DiophantineBoxResult(5, (-2, -2)),
    ),
    (
        (6, 9, 4, -5, 5, -5, 5),
        DiophantineBoxResult(0, None),
    ),
    (
        (6, 9, 3, -5, 5, -5, 5),
        DiophantineBoxResult(4, (-4, 3)),
    ),
    (
        (6, -9, 3, -5, 5, -5, 5),
        DiophantineBoxResult(4, (-4, -3)),
    ),
    (
        (2, 3, 12, 3, 3, 2, 2),
        DiophantineBoxResult(1, (3, 2)),
    ),
)
for arguments, expected in explicit_cases:
    assert solve_bounded_diophantine(*arguments) == expected

full_signed_box = solve_bounded_diophantine(
    0,
    0,
    0,
    _MIN_SIGNED_64,
    _MAX_SIGNED_64,
    _MIN_SIGNED_64,
    _MAX_SIGNED_64,
)
one_fixed_coordinate = solve_bounded_diophantine(
    _MIN_SIGNED_64,
    0,
    _MIN_SIGNED_64,
    1,
    1,
    _MIN_SIGNED_64,
    _MAX_SIGNED_64,
)
full_signed_diagonal = solve_bounded_diophantine(
    _MAX_SIGNED_64,
    -_MAX_SIGNED_64,
    0,
    _MIN_SIGNED_64,
    _MAX_SIGNED_64,
    _MIN_SIGNED_64,
    _MAX_SIGNED_64,
)
maximum_general_point = solve_bounded_diophantine(
    _MIN_SIGNED_64,
    _MAX_SIGNED_64,
    _MIN_SIGNED_64,
    1,
    1,
    0,
    0,
)
assert full_signed_box == DiophantineBoxResult(
    1 << 128,
    (_MIN_SIGNED_64, _MIN_SIGNED_64),
)
assert one_fixed_coordinate == DiophantineBoxResult(
    1 << 64,
    (1, _MIN_SIGNED_64),
)
assert full_signed_diagonal == DiophantineBoxResult(
    1 << 64,
    (_MIN_SIGNED_64, _MIN_SIGNED_64),
)
assert maximum_general_point == DiophantineBoxResult(1, (1, 0))

valid_arguments = [1, 1, 0, -1, 1, -1, 1]
rejected = 0
for index in range(len(valid_arguments)):
    for invalid, expected_error in (
        (True, TypeError),
        (_MIN_SIGNED_64 - 1, ValueError),
        (_MAX_SIGNED_64 + 1, ValueError),
    ):
        invalid_arguments = valid_arguments.copy()
        invalid_arguments[index] = invalid
        try:
            solve_bounded_diophantine(*invalid_arguments)
        except expected_error:
            rejected += 1
        else:
            raise AssertionError("invalid input must be rejected")

for invalid_bounds in (
    (1, 1, 0, 2, 1, -1, 1),
    (1, 1, 0, -1, 1, 2, 1),
):
    try:
        solve_bounded_diophantine(*invalid_bounds)
    except ValueError:
        rejected += 1
    else:
        raise AssertionError("inverted bounds must be rejected")

assert checked == 77_175
assert rejected == 23
```

## Trade-offs and Limitations

For nonzero coefficients, the gcd calculation and modular inverse take
`O(log(max(abs(a), abs(b))))` integer-arithmetic steps. Only a constant
number of Python integers is retained, although their bit-operation cost still
depends on magnitude. The rectangle may contain up to `2**128` points, but
its dimensions never affect the number of algorithm steps.

All seven inputs must be exact non-Boolean signed-64-bit integers, and both
coordinate intervals are closed and nonempty. The count is an arbitrary-size
Python integer. The witness uses ordinary tuple order: minimize `x` first,
then `y`.

The function solves one equation in exactly two variables. It does not list
the solutions, optimize a cost over them, accept unbounded intervals, solve
systems of equations, or provide constant-time cryptographic behavior.

## Related Snippets

<!-- catalog:related:start -->
- [Compute Canonical Bézout Coefficients for Two Bounded Integers](compute-canonical-bezout-coefficients-for-two-bounded-integers.md)
- [Solve a Bounded Linear Congruence as a Canonical Residue Class](solve-a-bounded-linear-congruence-as-a-canonical-residue-class.md)
- [Sum a Bounded Signed Affine Floor Sequence with Euclidean Reduction](sum-a-bounded-signed-affine-floor-sequence-with-euclidean-reduction.md)
<!-- catalog:related:end -->
