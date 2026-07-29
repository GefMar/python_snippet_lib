---
title: "Compute an Exact Integer-Matrix Determinant with Bareiss Elimination"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - build-and-evaluate-a-bounded-binary-assignment-constraint-system.md
  - combine-a-bounded-system-of-possibly-non-coprime-integer-congruences.md
  - ../machine-learning-statistics/fit-pca-with-numpy-and-report-cumulative-explained-variance.md
---

# Compute an Exact Integer-Matrix Determinant with Bareiss Elimination

## Idea and Problem

Compute the determinant of a bounded square integer matrix exactly without introducing fractions at every elimination step.

Bareiss elimination replaces each trailing entry with a two-by-two determinant
divided by the preceding pivot. For an integer matrix, that division is exact;
intermediate values remain integers while ordinary rational Gaussian
elimination would repeatedly construct and reduce fractions.

A zero diagonal entry does not by itself prove singularity. Swapping with the
first later row that has a non-zero entry in the pivot column permits
elimination to continue, while a sign tracks the determinant change. If no such
row exists, the determinant is zero.

## When to Use

Use this algorithm for a small, dense integer matrix when an exact determinant
is required and floating-point conditioning or rounding is unacceptable. It is
useful for reproducible algebraic checks, orientation tests, and bounded
integer transforms whose invertibility is determined by a non-zero
determinant.

Use a numerical library for large or floating-point matrices, especially when
conditioning, decompositions, rank, or linear-system solutions are also
required. Modular or sparse algorithms are better fits when matrix size or
coefficient growth makes dense bigint elimination expensive.

## Implementation

```python
_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_DETERMINANT_ORDER = 32


def integer_matrix_determinant(matrix: tuple[tuple[int, ...], ...]) -> int:
    """Return the exact determinant of a bounded square integer matrix."""
    if type(matrix) is not tuple:
        raise TypeError("matrix must be an exact tuple")
    if len(matrix) > _MAX_DETERMINANT_ORDER:
        raise ValueError("matrix order exceeds the supported limit")

    order = len(matrix)
    for row_index, row in enumerate(matrix):
        if type(row) is not tuple:
            raise TypeError(f"matrix[{row_index}] must be an exact tuple")
        if len(row) != order:
            raise ValueError("matrix must be square")
        for column_index, value in enumerate(row):
            if type(value) is not int:
                raise TypeError(f"matrix[{row_index}][{column_index}] must be an exact integer")
            if not _MIN_INT64 <= value <= _MAX_INT64:
                raise ValueError(
                    f"matrix[{row_index}][{column_index}] is outside the signed 64-bit range"
                )

    if order == 0:
        return 1

    work = [list(row) for row in matrix]
    sign = 1
    preceding_pivot = 1

    for pivot_index in range(order - 1):
        if work[pivot_index][pivot_index] == 0:
            swap_index = next(
                (
                    row_index
                    for row_index in range(pivot_index + 1, order)
                    if work[row_index][pivot_index] != 0
                ),
                None,
            )
            if swap_index is None:
                return 0
            work[pivot_index], work[swap_index] = work[swap_index], work[pivot_index]
            sign = -sign

        pivot = work[pivot_index][pivot_index]
        for row_index in range(pivot_index + 1, order):
            eliminated_value = work[row_index][pivot_index]
            for column_index in range(pivot_index + 1, order):
                numerator = (
                    work[row_index][column_index] * pivot
                    - eliminated_value * work[pivot_index][column_index]
                )
                quotient, remainder = divmod(numerator, preceding_pivot)
                if remainder != 0:
                    raise AssertionError("Bareiss division must remain exact")
                work[row_index][column_index] = quotient
            work[row_index][pivot_index] = 0
        preceding_pivot = pivot

    return sign * work[-1][-1]
```

## Example

```python
def integer_product(values: tuple[int, ...]) -> int:
    result = 1
    for value in values:
        result *= value
    return result


def determinant_by_permutations(matrix: tuple[tuple[int, ...], ...]) -> int:
    from itertools import permutations

    order = len(matrix)
    result = 0
    for selected_columns in permutations(range(order)):
        inversions = sum(
            selected_columns[left] > selected_columns[right]
            for left in range(order)
            for right in range(left + 1, order)
        )
        term = integer_product(tuple(matrix[row][selected_columns[row]] for row in range(order)))
        result += -term if inversions % 2 else term
    return result


def determinant_with_fractions(matrix: tuple[tuple[int, ...], ...]) -> int:
    from fractions import Fraction

    order = len(matrix)
    if order == 0:
        return 1

    work = [[Fraction(value) for value in row] for row in matrix]
    sign = 1
    for pivot_index in range(order):
        swap_index = next(
            (
                row_index
                for row_index in range(pivot_index, order)
                if work[row_index][pivot_index] != 0
            ),
            None,
        )
        if swap_index is None:
            return 0
        if swap_index != pivot_index:
            work[pivot_index], work[swap_index] = work[swap_index], work[pivot_index]
            sign = -sign

        pivot = work[pivot_index][pivot_index]
        for row_index in range(pivot_index + 1, order):
            factor = work[row_index][pivot_index] / pivot
            work[row_index][pivot_index] = Fraction()
            for column_index in range(pivot_index + 1, order):
                work[row_index][column_index] -= factor * work[pivot_index][column_index]

    determinant = Fraction(sign)
    for index in range(order):
        determinant *= work[index][index]
    assert determinant.denominator == 1
    return determinant.numerator


def exercise_small_integer_matrices() -> int:
    from itertools import product

    checked = 0
    for order in range(4):
        for flat_values in product((-1, 0, 1), repeat=order * order):
            matrix = tuple(
                tuple(flat_values[row * order : (row + 1) * order]) for row in range(order)
            )
            expected = determinant_by_permutations(matrix)
            assert integer_matrix_determinant(matrix) == expected
            assert determinant_with_fractions(matrix) == expected
            checked += 1
    return checked


def exercise_random_matrices() -> int:
    from random import Random

    generator = Random(20_260_729)
    checked = 0
    for order in range(4, 7):
        for _ in range(200):
            matrix = tuple(
                tuple(generator.randint(-8, 8) for _ in range(order)) for _ in range(order)
            )
            expected = determinant_with_fractions(matrix)
            assert integer_matrix_determinant(matrix) == expected
            assert determinant_by_permutations(matrix) == expected
            checked += 1
    return checked


def raises(error_type: type[Exception], operation: object) -> bool:
    try:
        operation()  # type: ignore[operator]
    except error_type:
        return True
    return False


late_swap_matrix = ((2, 2, 0), (2, 2, 1), (0, 1, 1))
singular_matrix = ((1, 2, 3), (2, 4, 6), (7, 8, 9))
extrema_matrix = ((_MIN_INT64, _MAX_INT64), (_MAX_INT64, _MIN_INT64))

boundary_matrix = tuple(
    tuple(
        (
            _MAX_INT64
            if row_index == column_index and row_index % 2 == 0
            else _MIN_INT64
            if row_index == column_index
            else row_index - column_index
            if column_index < row_index
            else 0
        )
        for column_index in range(_MAX_DETERMINANT_ORDER)
    )
    for row_index in range(_MAX_DETERMINANT_ORDER)
)
boundary_diagonal = tuple(boundary_matrix[index][index] for index in range(_MAX_DETERMINANT_ORDER))

assert (
    exercise_small_integer_matrices(),
    exercise_random_matrices(),
    integer_matrix_determinant(()),
    integer_matrix_determinant(((7,),)),
    integer_matrix_determinant(late_swap_matrix),
    integer_matrix_determinant((late_swap_matrix[1], late_swap_matrix[0], late_swap_matrix[2])),
    integer_matrix_determinant(singular_matrix),
    integer_matrix_determinant(extrema_matrix),
    integer_matrix_determinant(boundary_matrix) == integer_product(boundary_diagonal),
    raises(TypeError, lambda: integer_matrix_determinant(((1, True), (0, 1)))),
    raises(ValueError, lambda: integer_matrix_determinant(((1, 2),))),
    raises(
        ValueError,
        lambda: integer_matrix_determinant(
            tuple((0,) * 33 for _ in range(33)),
        ),
    ),
) == (
    19_768,
    600,
    1,
    7,
    -2,
    2,
    0,
    (1 << 64) - 1,
    True,
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

For an `N x N` matrix, elimination performs `O(N^3)` bigint
multiplications, subtractions, and exact divisions. The mutable matrix copy
uses `O(N^2)` integer slots. Row selection is deterministic, and every swap
reverses the tracked sign while leaving the original immutable rows unchanged.

Fraction-free does not mean constant-cost. Intermediate entries are signed
minors whose bit lengths can grow with both matrix order and input magnitude;
the determinant can also be much wider than 64 bits. Python preserves exactness,
but bigint multiplication and division become more expensive as those values
grow. The explicit remainder check protects the exact-division invariant rather
than silently accepting floor division after an implementation error.

The determinant of the empty matrix is defined as one. A zero pivot triggers
the first usable later-row swap, and a column with no usable pivot proves the
determinant is zero. This dense algorithm does not accept floats or symbolic
values, assess numerical conditioning, calculate rank or an inverse, solve a
linear system, exploit sparsity, use modular reconstruction, or return
intermediate elimination state.

## Related Snippets

<!-- catalog:related:start -->
- [Build and Evaluate a Bounded Binary Assignment Constraint System](build-and-evaluate-a-bounded-binary-assignment-constraint-system.md)
- [Combine a Bounded System of Possibly Non-Coprime Integer Congruences](combine-a-bounded-system-of-possibly-non-coprime-integer-congruences.md)
- [Fit PCA with NumPy and Report Cumulative Explained Variance](../machine-learning-statistics/fit-pca-with-numpy-and-report-cumulative-explained-variance.md)
<!-- catalog:related:end -->
