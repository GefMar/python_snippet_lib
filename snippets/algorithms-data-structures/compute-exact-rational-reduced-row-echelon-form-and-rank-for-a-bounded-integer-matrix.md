---
title: "Compute Exact Rational Reduced Row-Echelon Form and Rank for a Bounded Integer Matrix"
snippet_type: algorithm
use_cases:
  - data-transformation
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-an-exact-integer-matrix-determinant-with-bareiss-elimination.md
  - solve-a-bounded-tridiagonal-integer-system-exactly-with-thomas-elimination.md
  - interpolate-a-global-polynomial-exactly-from-bounded-integer-points.md
---

# Compute Exact Rational Reduced Row-Echelon Form and Rank for a Bounded Integer Matrix

## Idea and Problem

Reduce a bounded rectangular integer matrix to its unique reduced row-echelon form over the rational numbers and report its pivot structure.

Gauss-Jordan elimination scans columns from left to right. For each pivot it
selects the first remaining row with a nonzero entry, normalizes that row, and
eliminates the pivot column from every other row. `fractions.Fraction` keeps
each operation exact, so rank decisions never depend on a floating-point
tolerance.

The normalized rows, rank, and increasing pivot-column indexes form one
immutable result. The pivot-row selection rule makes the computation
reproducible even though the mathematical RREF is already unique.

## When to Use

Use this algorithm for small exact matrices when rational row relations,
reproducible rank, or a transparent test oracle matters. It is especially
useful when all inputs are integers and converting a nearly dependent matrix to
floating point would make a tolerance part of the result.

Use a numerical linear-algebra library for large dense arrays, approximate
measurements, condition estimates, sparse storage, vectorized performance, or
carefully designed numerical-rank decisions. Use an integer normal-form
algorithm when the matrix is being studied as an integer module rather than a
vector space over the rationals.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_RREF_ROWS = 30
_MAX_RREF_COLUMNS = 30


@dataclass(frozen=True, slots=True)
class ExactRrefResult:
    rows: tuple[tuple[Fraction, ...], ...]
    rank: int
    pivot_columns: tuple[int, ...]


def exact_rational_rref(
    matrix: tuple[tuple[int, ...], ...],
) -> ExactRrefResult:
    """Return exact rational RREF, rank, and pivot columns."""
    if type(matrix) is not tuple:
        raise TypeError("matrix must be an exact tuple")
    if not 1 <= len(matrix) <= _MAX_RREF_ROWS:
        raise ValueError("matrix row count is outside the supported range")

    column_count: int | None = None
    for row_index, row in enumerate(matrix):
        if type(row) is not tuple:
            raise TypeError(f"matrix[{row_index}] must be an exact tuple")
        if column_count is None:
            column_count = len(row)
            if not 1 <= column_count <= _MAX_RREF_COLUMNS:
                raise ValueError("matrix column count is outside the supported range")
        elif len(row) != column_count:
            raise ValueError("matrix must be rectangular")

        for column_index, value in enumerate(row):
            if type(value) is not int:
                raise TypeError(f"matrix[{row_index}][{column_index}] must be an exact integer")
            if not _MIN_INT64 <= value <= _MAX_INT64:
                raise ValueError(
                    f"matrix[{row_index}][{column_index}] is outside the signed 64-bit range"
                )

    if column_count is None:
        raise AssertionError("a non-empty matrix must have columns")
    rows = [[Fraction(value) for value in row] for row in matrix]
    pivot_columns: list[int] = []
    pivot_row = 0

    for column in range(column_count):
        selected_row = next(
            (candidate for candidate in range(pivot_row, len(rows)) if rows[candidate][column]),
            None,
        )
        if selected_row is None:
            continue

        rows[pivot_row], rows[selected_row] = (
            rows[selected_row],
            rows[pivot_row],
        )
        pivot_value = rows[pivot_row][column]
        rows[pivot_row] = [value / pivot_value for value in rows[pivot_row]]

        for row_index, row in enumerate(rows):
            if row_index == pivot_row:
                continue
            factor = row[column]
            if factor:
                rows[row_index] = [
                    value - factor * pivot_entry
                    for value, pivot_entry in zip(row, rows[pivot_row], strict=True)
                ]

        pivot_columns.append(column)
        pivot_row += 1
        if pivot_row == len(rows):
            break

    return ExactRrefResult(
        rows=tuple(tuple(row) for row in rows),
        rank=len(pivot_columns),
        pivot_columns=tuple(pivot_columns),
    )
```

## Example

```python
dependent = exact_rational_rref(
    (
        (1, 2, 3),
        (2, 4, 6),
        (1, 1, 1),
    )
)
wide = exact_rational_rref(((1, 2, 3), (0, 1, 4)))
tall = exact_rational_rref(((1, 2), (2, 4), (0, 0)))
zero = exact_rational_rref(((0, 0), (0, 0)))

assert dependent == ExactRrefResult(
    rows=(
        (Fraction(1), Fraction(0), Fraction(-1)),
        (Fraction(0), Fraction(1), Fraction(2)),
        (Fraction(0), Fraction(0), Fraction(0)),
    ),
    rank=2,
    pivot_columns=(0, 1),
)
assert wide == ExactRrefResult(
    rows=(
        (Fraction(1), Fraction(0), Fraction(-5)),
        (Fraction(0), Fraction(1), Fraction(4)),
    ),
    rank=2,
    pivot_columns=(0, 1),
)
assert tall == ExactRrefResult(
    rows=(
        (Fraction(1), Fraction(2)),
        (Fraction(0), Fraction(0)),
        (Fraction(0), Fraction(0)),
    ),
    rank=1,
    pivot_columns=(0,),
)
assert zero == ExactRrefResult(
    rows=(
        (Fraction(0), Fraction(0)),
        (Fraction(0), Fraction(0)),
    ),
    rank=0,
    pivot_columns=(),
)
```

## Trade-offs and Limitations

For `r` rows, `c` columns, and at most `min(r, c)` pivots, elimination
performs `O(r * c * min(r, c))` rational arithmetic operations and stores
`O(r * c)` fractions. Those operations are not constant-time: intermediate
numerators and denominators can grow far beyond the signed 64-bit input range.
The small dimensions bound object count, not coefficient bit length.

The returned rank is exact rank over the rational field. It is not numerical
rank under a tolerance, and the result is not Smith or Hermite normal form over
the integers. Exact zero rows settle at the bottom as a consequence of the
left-to-right pivot scan.

This function does not return row-operation certificates, determinants,
inverses, nullspace bases, equation solutions, condition information, or sparse
representations. Repeated work on large or structured matrices belongs in a
specialized exact linear-algebra system.

## Related Snippets

<!-- catalog:related:start -->
- [Compute an Exact Integer-Matrix Determinant with Bareiss Elimination](compute-an-exact-integer-matrix-determinant-with-bareiss-elimination.md)
- [Solve a Bounded Tridiagonal Integer System Exactly with Thomas Elimination](solve-a-bounded-tridiagonal-integer-system-exactly-with-thomas-elimination.md)
- [Interpolate a Global Polynomial Exactly from Bounded Integer Points](interpolate-a-global-polynomial-exactly-from-bounded-integer-points.md)
<!-- catalog:related:end -->
