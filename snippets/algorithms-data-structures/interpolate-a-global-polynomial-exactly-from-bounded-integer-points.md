---
title: "Interpolate a Global Polynomial Exactly from Bounded Integer Points"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../data-processing/interpolate-increasing-integer-time-series-points-exactly.md
  - compute-an-exact-integer-matrix-determinant-with-bareiss-elimination.md
  - ../machine-learning-statistics/fit-an-exact-ordinary-least-squares-line-to-bounded-integer-points.md
---

# Interpolate a Global Polynomial Exactly from Bounded Integer Points

## Idea and Problem

Recover the unique global polynomial through a bounded set of distinct integer points while keeping every monomial coefficient exact.

Newton divided differences construct one coefficient for each successively
larger prefix of points. Sorting the points by their x-coordinate makes that
construction deterministic, and expanding each Newton basis product converts
the result into ordinary monomial coefficients. `Fraction` preserves exact
division even when the polynomial has non-integer coefficients.

The returned tuple lists the constant coefficient first. Trailing zero
coefficients are removed, so the same polynomial has one canonical
representation regardless of how many source points lie on it.

## When to Use

Use this algorithm for a small, complete set of exact integer points when the
single polynomial of degree below the point count is itself the required
artifact. Canonical exact coefficients are useful for deterministic symbolic
fixtures, algebraic identities, and reproducible evaluation without
floating-point rounding.

Use piecewise interpolation when values between neighboring samples should not
be affected by distant points. Choose a symbolic algebra system for larger
expressions, simplification, derivatives, multiple variables, or richer exact
domains. Regression is the appropriate model when points contain noise and an
exact pass through every observation is not intended.

## Implementation

```python
from fractions import Fraction

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_INTERPOLATION_POINTS = 32


def interpolate_integer_points_exactly(
    points: tuple[tuple[int, int], ...],
) -> tuple[Fraction, ...]:
    """Return canonical ascending monomial coefficients through all points."""
    if type(points) is not tuple:
        raise TypeError("points must be an exact tuple")
    if not 1 <= len(points) <= _MAX_INTERPOLATION_POINTS:
        raise ValueError("point count is outside the supported range")

    validated: list[tuple[int, int]] = []
    seen_x_coordinates: set[int] = set()
    for point_index, point in enumerate(points):
        if type(point) is not tuple:
            raise TypeError(f"points[{point_index}] must be an exact tuple")
        if len(point) != 2:
            raise ValueError(f"points[{point_index}] must contain exactly two values")

        x_coordinate, y_coordinate = point
        if type(x_coordinate) is not int or type(y_coordinate) is not int:
            raise TypeError(f"points[{point_index}] coordinates must be exact integers")
        if not _MIN_INT64 <= x_coordinate <= _MAX_INT64:
            raise ValueError(f"points[{point_index}].x is outside the signed 64-bit range")
        if not _MIN_INT64 <= y_coordinate <= _MAX_INT64:
            raise ValueError(f"points[{point_index}].y is outside the signed 64-bit range")
        if x_coordinate in seen_x_coordinates:
            raise ValueError("point x-coordinates must be distinct")

        seen_x_coordinates.add(x_coordinate)
        validated.append((x_coordinate, y_coordinate))

    validated.sort(key=lambda point: point[0])
    x_coordinates = [point[0] for point in validated]
    divided_differences = [Fraction(point[1]) for point in validated]

    for width in range(1, len(validated)):
        for right_index in range(len(validated) - 1, width - 1, -1):
            divided_differences[right_index] = (
                divided_differences[right_index] - divided_differences[right_index - 1]
            ) / (x_coordinates[right_index] - x_coordinates[right_index - width])

    coefficients = [Fraction()] * len(validated)
    basis = [Fraction(1)]
    for order, newton_coefficient in enumerate(divided_differences):
        for degree, basis_coefficient in enumerate(basis):
            coefficients[degree] += newton_coefficient * basis_coefficient

        if order + 1 == len(divided_differences):
            continue

        root = x_coordinates[order]
        expanded_basis = [Fraction()] * (len(basis) + 1)
        for degree, basis_coefficient in enumerate(basis):
            expanded_basis[degree] -= root * basis_coefficient
            expanded_basis[degree + 1] += basis_coefficient
        basis = expanded_basis

    while len(coefficients) > 1 and coefficients[-1] == 0:
        coefficients.pop()
    return tuple(coefficients)
```

## Example

```python
def multiply_polynomials(
    left: tuple[Fraction, ...],
    right: tuple[Fraction, ...],
) -> tuple[Fraction, ...]:
    result = [Fraction()] * (len(left) + len(right) - 1)
    for left_degree, left_coefficient in enumerate(left):
        for right_degree, right_coefficient in enumerate(right):
            result[left_degree + right_degree] += left_coefficient * right_coefficient
    return tuple(result)


def lagrange_coefficients(
    points: tuple[tuple[int, int], ...],
) -> tuple[Fraction, ...]:
    result = [Fraction()] * len(points)
    for selected_index, (selected_x, selected_y) in enumerate(points):
        numerator = (Fraction(1),)
        denominator = 1
        for other_index, (other_x, _) in enumerate(points):
            if other_index == selected_index:
                continue
            numerator = multiply_polynomials(
                numerator,
                (Fraction(-other_x), Fraction(1)),
            )
            denominator *= selected_x - other_x

        scale = Fraction(selected_y, denominator)
        for degree, coefficient in enumerate(numerator):
            result[degree] += scale * coefficient

    while len(result) > 1 and result[-1] == 0:
        result.pop()
    return tuple(result)


def evaluate_polynomial(coefficients: tuple[Fraction, ...], x_coordinate: int) -> Fraction:
    result = Fraction()
    for coefficient in reversed(coefficients):
        result = result * x_coordinate + coefficient
    return result


def assert_interpolation(points: tuple[tuple[int, int], ...]) -> None:
    actual = interpolate_integer_points_exactly(points)
    assert actual == lagrange_coefficients(points)
    assert all(
        evaluate_polynomial(actual, x_coordinate) == y_coordinate
        for x_coordinate, y_coordinate in points
    )


def exercise_small_interpolations() -> int:
    from itertools import combinations, product

    checked = 0
    x_pool = (-2, -1, 0, 1)
    for point_count in range(1, len(x_pool) + 1):
        for x_coordinates in combinations(x_pool, point_count):
            for y_coordinates in product((-1, 0, 1), repeat=point_count):
                points = tuple(zip(x_coordinates, y_coordinates, strict=True))
                assert_interpolation(points)
                assert_interpolation(tuple(reversed(points)))
                checked += 1
    return checked


def exercise_random_interpolations() -> int:
    from random import Random

    generator = Random(20_260_729)
    checked = 0
    for _ in range(400):
        point_count = generator.randint(1, 8)
        x_coordinates = generator.sample(range(-20, 21), point_count)
        points = [(x_coordinate, generator.randint(-50, 50)) for x_coordinate in x_coordinates]
        generator.shuffle(points)
        assert_interpolation(tuple(points))
        checked += 1
    return checked


def raises(error_type: type[Exception], operation) -> bool:
    try:
        operation()
    except error_type:
        return True
    return False


sparse_coefficients = (
    Fraction(1),
    Fraction(0),
    Fraction(0),
    Fraction(0),
    Fraction(1),
)
sparse_points = tuple(
    (x_coordinate, evaluate_polynomial(sparse_coefficients, x_coordinate).numerator)
    for x_coordinate in range(-2, 3)
)

known_coefficients = (
    Fraction(3),
    Fraction(-2),
    Fraction(0),
    Fraction(5),
)
boundary_points = tuple(
    (x_coordinate, evaluate_polynomial(known_coefficients, x_coordinate).numerator)
    for x_coordinate in range(-16, 16)
)

assert (
    exercise_small_interpolations(),
    exercise_random_interpolations(),
    interpolate_integer_points_exactly(((7, -3),)),
    interpolate_integer_points_exactly(((-2, 9), (0, 9), (3, 9), (8, 9))),
    interpolate_integer_points_exactly(sparse_points),
    interpolate_integer_points_exactly(((_MIN_INT64, _MIN_INT64), (_MAX_INT64, _MAX_INT64))),
    interpolate_integer_points_exactly(boundary_points),
    interpolate_integer_points_exactly(tuple(reversed(boundary_points))),
    raises(TypeError, lambda: interpolate_integer_points_exactly(((0, True),))),
    raises(ValueError, lambda: interpolate_integer_points_exactly(((0, 1), (0, 2)))),
    raises(ValueError, lambda: interpolate_integer_points_exactly(())),
    raises(
        ValueError,
        lambda: interpolate_integer_points_exactly(tuple((value, value) for value in range(33))),
    ),
) == (
    255,
    400,
    (Fraction(-3),),
    (Fraction(9),),
    sparse_coefficients,
    (Fraction(0), Fraction(1)),
    known_coefficients,
    known_coefficients,
    True,
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

Validation takes `O(N)` expected-time set operations, sorting takes
`O(N log N)` comparisons, and divided differences plus basis expansion perform
`O(N^2)` `Fraction` operations. The validated points, divided differences,
coefficients, and current basis use `O(N)` working and output slots.

Fraction arithmetic is not constant-cost. Numerators and denominators can grow
rapidly with point count, coordinate magnitude, and closely spaced
x-coordinates; multiplication, division, and GCD reduction therefore cost more
as their operands widen. Signed 64-bit input bounds do not bound the exact
coefficient widths to 64 bits.

One point produces a constant polynomial. Internal zero coefficients remain in
their degree positions, while high trailing zeroes are removed and the zero
polynomial is represented as `(Fraction(0),)`. Input order does not affect the
result. The function does not evaluate separate query points, accept duplicate
x-coordinates or floating values, construct piecewise curves or splines, fit
noisy data, or make high-degree interpolation and extrapolation a sound model.

## Related Snippets

<!-- catalog:related:start -->
- [Interpolate Increasing Integer Time-Series Points Exactly](../data-processing/interpolate-increasing-integer-time-series-points-exactly.md)
- [Compute an Exact Integer-Matrix Determinant with Bareiss Elimination](compute-an-exact-integer-matrix-determinant-with-bareiss-elimination.md)
- [Fit an Exact Ordinary Least-Squares Line to Bounded Integer Points](../machine-learning-statistics/fit-an-exact-ordinary-least-squares-line-to-bounded-integer-points.md)
<!-- catalog:related:end -->
