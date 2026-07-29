---
title: "Solve a Bounded Tridiagonal Integer System Exactly with Thomas Elimination"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-an-exact-integer-matrix-determinant-with-bareiss-elimination.md
  - interpolate-a-global-polynomial-exactly-from-bounded-integer-points.md
  - ../machine-learning-statistics/fit-an-exact-ordinary-least-squares-line-to-bounded-integer-points.md
---

# Solve a Bounded Tridiagonal Integer System Exactly with Thomas Elimination

## Idea and Problem

Solve one bounded tridiagonal integer linear system exactly without materializing or eliminating a dense matrix.

A tridiagonal matrix has non-zero entries only on its main diagonal and the
two adjacent diagonals. Thomas elimination stores one modified upper
coefficient and right-hand-side value per row, then reconstructs the solution
by backward substitution. `Fraction` preserves every division exactly.

The algorithm deliberately performs no row pivoting. Each modified forward
pivot must be non-zero; otherwise the input is outside this function's
Thomas-compatible contract even when a general pivoting solver could still
solve the matrix.

## When to Use

Use this solver for one complete, small integer system whose matrix is known to
be tridiagonal and whose forward Thomas pivots are all non-zero. Exact rational
output is useful for reproducible algebraic fixtures, recurrence boundary
calculations, and symbolic checks where floating-point rounding is unwanted.

Use a pivoting dense or banded solver when zero pivots are possible, a
numerical library when conditioning and floating-point performance matter, or
a reusable factorization when many right-hand sides share one matrix. Cyclic
tridiagonal and wider band matrices require different algorithms.

## Implementation

```python
from fractions import Fraction

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_TRIDIAGONAL_ORDER = 128


def _validated_integer_diagonal(
    values: object,
    *,
    field: str,
) -> tuple[int, ...]:
    if type(values) is not tuple:
        raise TypeError(f"{field} must be an exact tuple")
    for position, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"{field}[{position}] must be an exact integer")
        if not _MIN_INT64 <= value <= _MAX_INT64:
            raise ValueError(f"{field}[{position}] is outside the signed 64-bit range")
    return values


def solve_tridiagonal_integer_system_exactly(
    lower: tuple[int, ...],
    diagonal: tuple[int, ...],
    upper: tuple[int, ...],
    right_hand_side: tuple[int, ...],
) -> tuple[Fraction, ...]:
    """Return the exact solution of one Thomas-compatible system."""
    for field, values in (
        ("lower", lower),
        ("diagonal", diagonal),
        ("upper", upper),
        ("right_hand_side", right_hand_side),
    ):
        if type(values) is not tuple:
            raise TypeError(f"{field} must be an exact tuple")

    order = len(diagonal)
    if not 1 <= order <= _MAX_TRIDIAGONAL_ORDER:
        raise ValueError("system order is outside the supported range")
    if len(lower) != order - 1:
        raise ValueError("lower must contain exactly order - 1 values")
    if len(upper) != order - 1:
        raise ValueError("upper must contain exactly order - 1 values")
    if len(right_hand_side) != order:
        raise ValueError("right_hand_side must contain exactly order values")

    checked_lower = _validated_integer_diagonal(lower, field="lower")
    checked_diagonal = _validated_integer_diagonal(diagonal, field="diagonal")
    checked_upper = _validated_integer_diagonal(upper, field="upper")
    checked_rhs = _validated_integer_diagonal(right_hand_side, field="right_hand_side")

    first_pivot = Fraction(checked_diagonal[0])
    if first_pivot == 0:
        raise ValueError("system has a zero Thomas pivot at row 0")

    modified_upper = [Fraction()] * max(order - 1, 0)
    modified_rhs = [Fraction()] * order
    if order > 1:
        modified_upper[0] = Fraction(checked_upper[0]) / first_pivot
    modified_rhs[0] = Fraction(checked_rhs[0]) / first_pivot

    for row in range(1, order):
        pivot = Fraction(checked_diagonal[row]) - checked_lower[row - 1] * modified_upper[row - 1]
        if pivot == 0:
            raise ValueError(f"system has a zero Thomas pivot at row {row}")
        if row < order - 1:
            modified_upper[row] = Fraction(checked_upper[row]) / pivot
        modified_rhs[row] = (
            Fraction(checked_rhs[row]) - checked_lower[row - 1] * modified_rhs[row - 1]
        ) / pivot

    solution = [Fraction()] * order
    solution[-1] = modified_rhs[-1]
    for row in range(order - 2, -1, -1):
        solution[row] = modified_rhs[row] - modified_upper[row] * solution[row + 1]
    return tuple(solution)
```

## Example

```python
def dense_tridiagonal_matrix(
    lower: tuple[int, ...],
    diagonal: tuple[int, ...],
    upper: tuple[int, ...],
) -> tuple[tuple[int, ...], ...]:
    order = len(diagonal)
    rows: list[tuple[int, ...]] = []
    for row in range(order):
        values = [0] * order
        values[row] = diagonal[row]
        if row:
            values[row - 1] = lower[row - 1]
        if row + 1 < order:
            values[row + 1] = upper[row]
        rows.append(tuple(values))
    return tuple(rows)


def solve_dense_system_with_pivoting(
    matrix: tuple[tuple[int, ...], ...],
    right_hand_side: tuple[int, ...],
) -> tuple[Fraction, ...] | None:
    order = len(matrix)
    work = [
        [Fraction(value) for value in row] + [Fraction(right_hand_side[row_index])]
        for row_index, row in enumerate(matrix)
    ]

    for column in range(order):
        pivot_row = next(
            (row for row in range(column, order) if work[row][column] != 0),
            None,
        )
        if pivot_row is None:
            return None
        work[column], work[pivot_row] = work[pivot_row], work[column]
        pivot = work[column][column]
        for row in range(column + 1, order):
            factor = work[row][column] / pivot
            for position in range(column, order + 1):
                work[row][position] -= factor * work[column][position]

    solution = [Fraction()] * order
    for row in range(order - 1, -1, -1):
        remainder = work[row][-1] - sum(
            work[row][column] * solution[column] for column in range(row + 1, order)
        )
        solution[row] = remainder / work[row][row]
    return tuple(solution)


def assert_exact_solution(
    matrix: tuple[tuple[int, ...], ...],
    right_hand_side: tuple[int, ...],
    solution: tuple[Fraction, ...],
) -> None:
    assert all(
        sum(coefficient * value for coefficient, value in zip(row, solution, strict=True))
        == expected
        for row, expected in zip(matrix, right_hand_side, strict=True)
    )


def exercise_small_systems() -> int:
    from itertools import product

    checked = 0
    for order in (1, 2):
        for values in product((-1, 0, 1), repeat=4 * order - 2):
            lower = tuple(values[: order - 1])
            diagonal = tuple(values[order - 1 : 2 * order - 1])
            upper = tuple(values[2 * order - 1 : 3 * order - 2])
            right_hand_side = tuple(values[3 * order - 2 :])
            matrix = dense_tridiagonal_matrix(lower, diagonal, upper)
            dense_solution = solve_dense_system_with_pivoting(matrix, right_hand_side)

            compatible = diagonal[0] != 0
            if order == 2 and compatible:
                compatible = Fraction(diagonal[1]) - Fraction(lower[0] * upper[0], diagonal[0]) != 0
            if compatible:
                actual = solve_tridiagonal_integer_system_exactly(
                    lower,
                    diagonal,
                    upper,
                    right_hand_side,
                )
                assert actual == dense_solution
                assert_exact_solution(matrix, right_hand_side, actual)
            else:
                try:
                    solve_tridiagonal_integer_system_exactly(
                        lower,
                        diagonal,
                        upper,
                        right_hand_side,
                    )
                except ValueError:
                    pass
                else:
                    raise AssertionError("zero Thomas pivot was accepted")
            checked += 1
    return checked


def exercise_random_systems() -> int:
    from random import Random

    generator = Random(20_260_729)
    checked = 0
    for order in range(3, 9):
        for _ in range(100):
            lower = tuple(generator.randint(-3, 3) for _ in range(order - 1))
            upper = tuple(generator.randint(-3, 3) for _ in range(order - 1))
            diagonal = tuple(
                generator.choice((-1, 1)) * generator.randint(10, 20) for _ in range(order)
            )
            right_hand_side = tuple(generator.randint(-20, 20) for _ in range(order))
            matrix = dense_tridiagonal_matrix(lower, diagonal, upper)
            expected = solve_dense_system_with_pivoting(matrix, right_hand_side)
            actual = solve_tridiagonal_integer_system_exactly(
                lower,
                diagonal,
                upper,
                right_hand_side,
            )
            assert actual == expected
            assert_exact_solution(matrix, right_hand_side, actual)
            checked += 1
    return checked


def raises(error_type: type[Exception], operation) -> bool:
    try:
        operation()
    except error_type:
        return True
    return False


first_pivot_matrix = ((0, 1), (1, 1))
late_pivot_matrix = ((1, 1, 0), (1, 1, 1), (0, 1, 1))
assert solve_dense_system_with_pivoting(first_pivot_matrix, (1, 2)) == (
    Fraction(1),
    Fraction(1),
)
assert solve_dense_system_with_pivoting(late_pivot_matrix, (3, 6, 5)) == (
    Fraction(1),
    Fraction(2),
    Fraction(3),
)

maximum_diagonal = tuple(
    _MAX_INT64 if row % 2 == 0 else _MIN_INT64 for row in range(_MAX_TRIDIAGONAL_ORDER)
)
maximum_rhs = tuple(reversed(maximum_diagonal))
maximum_solution = solve_tridiagonal_integer_system_exactly(
    (0,) * (_MAX_TRIDIAGONAL_ORDER - 1),
    maximum_diagonal,
    (0,) * (_MAX_TRIDIAGONAL_ORDER - 1),
    maximum_rhs,
)

assert (
    exercise_small_systems(),
    exercise_random_systems(),
    len(maximum_solution),
    maximum_solution[0],
    maximum_solution[-1],
    raises(
        ValueError,
        lambda: solve_tridiagonal_integer_system_exactly((1,), (0, 1), (1,), (1, 2)),
    ),
    raises(
        ValueError,
        lambda: solve_tridiagonal_integer_system_exactly(
            (1, 1),
            (1, 1, 1),
            (1, 1),
            (3, 6, 5),
        ),
    ),
    raises(ValueError, lambda: solve_tridiagonal_integer_system_exactly((), (), (), ())),
    raises(
        ValueError,
        lambda: solve_tridiagonal_integer_system_exactly((), (1, 2), (), (1, 2)),
    ),
    raises(
        TypeError,
        lambda: solve_tridiagonal_integer_system_exactly((), (True,), (), (1,)),
    ),
) == (
    738,
    600,
    128,
    Fraction(_MIN_INT64, _MAX_INT64),
    Fraction(_MAX_INT64, _MIN_INT64),
    True,
    True,
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

For order `n`, forward elimination and backward substitution use `O(n)`
rational operations and `O(n)` auxiliary `Fraction` values. This is not a
unit-cost `O(n)` bit-time guarantee: numerators and denominators can grow, and
each exact operation may spend additional time reducing large integers.

A zero modified pivot raises `ValueError`; it does not prove that the matrix is
singular. Row exchanges can solve some rejected systems, but adding them would
change the storage and algorithmic contract. The signed 64-bit bounds apply to
inputs, while exact intermediate and output integers can be larger.

The function solves one static square system and returns one complete rational
solution. It does not handle floating-point tolerances, numerical stability,
multiple right-hand sides, reusable factorization, cyclic corner entries,
wider bands, sparse general matrices, underdetermined systems, or approximate
least-squares solutions.

## Related Snippets

<!-- catalog:related:start -->
- [Compute an Exact Integer-Matrix Determinant with Bareiss Elimination](compute-an-exact-integer-matrix-determinant-with-bareiss-elimination.md)
- [Interpolate a Global Polynomial Exactly from Bounded Integer Points](interpolate-a-global-polynomial-exactly-from-bounded-integer-points.md)
- [Fit an Exact Ordinary Least-Squares Line to Bounded Integer Points](../machine-learning-statistics/fit-an-exact-ordinary-least-squares-line-to-bounded-integer-points.md)
<!-- catalog:related:end -->
