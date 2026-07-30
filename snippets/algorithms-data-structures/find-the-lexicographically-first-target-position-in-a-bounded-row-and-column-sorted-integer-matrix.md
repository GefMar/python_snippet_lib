---
title: "Find the Lexicographically First Target Position in a Bounded Row-and-Column-Sorted Integer Matrix"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - build-a-bounded-integer-summed-area-table-for-half-open-rectangle-sums.md
  - select-a-zero-based-order-statistic-from-bounded-integers-with-three-way-quickselect.md
  - find-a-point-in-disjoint-half-open-intervals.md
---

# Find the Lexicographically First Target Position in a Bounded Row-and-Column-Sorted Integer Matrix

## Idea and Problem

Find the earliest row-and-column position of one target in a validated two-dimensionally sorted integer matrix.

Starting at the top-right cell makes every comparison eliminate a complete
row prefix or column suffix. A value greater than the target eliminates its
column at and below the current row, while a smaller value eliminates the
remaining prefix of its row. On equality, continuing left finds the first
target column in the earliest row that can contain the target.

## When to Use

Use this staircase search when a bounded rectangular matrix is already
expected to be nondecreasing both left-to-right and top-to-bottom, duplicate
values are allowed, and one deterministic target position is enough. The
validation performed here is useful at an untrusted or assertion-heavy input
boundary.

Skip repeated full validation when the same immutable matrix has already been
validated for many queries; retain a validated wrapper or build a dedicated
index instead. Use row-wise binary search when only rows are sorted, and use a
linear scan when no two-dimensional ordering promise exists.

## Implementation

```python
_MAX_MONOTONE_MATRIX_ROWS = 512
_MAX_MONOTONE_MATRIX_COLUMNS = 512
_MAX_MONOTONE_MATRIX_CELLS = 65_536
_MIN_SIGNED_64 = -(1 << 63)
_MAX_SIGNED_64 = (1 << 63) - 1


def find_first_monotone_matrix_target(
    matrix: tuple[tuple[int, ...], ...],
    target: int,
) -> tuple[int, int] | None:
    """Return the smallest row-first target position, or None if absent."""
    if type(matrix) is not tuple:
        raise TypeError("matrix must be an exact tuple")
    if type(target) is not int:
        raise TypeError("target must be an exact integer")
    if not _MIN_SIGNED_64 <= target <= _MAX_SIGNED_64:
        raise ValueError("target is outside the signed 64-bit range")

    row_count = len(matrix)
    if row_count > _MAX_MONOTONE_MATRIX_ROWS:
        raise ValueError("matrix has more than 512 rows")

    column_count = 0
    if row_count:
        first_row = matrix[0]
        if type(first_row) is not tuple:
            raise TypeError("matrix[0] must be an exact tuple")
        column_count = len(first_row)
        if column_count > _MAX_MONOTONE_MATRIX_COLUMNS:
            raise ValueError("matrix has more than 512 columns")
        if row_count * column_count > _MAX_MONOTONE_MATRIX_CELLS:
            raise ValueError("matrix has more than 65536 cells")

    for row_index, row in enumerate(matrix):
        if type(row) is not tuple:
            raise TypeError(f"matrix[{row_index}] must be an exact tuple")
        if len(row) != column_count:
            raise ValueError("matrix must be rectangular")
        for column_index, value in enumerate(row):
            if type(value) is not int:
                raise TypeError("matrix cells must be exact integers")
            if not _MIN_SIGNED_64 <= value <= _MAX_SIGNED_64:
                raise ValueError("matrix cell is outside the signed 64-bit range")
            if column_index and row[column_index - 1] > value:
                raise ValueError("matrix rows must be nondecreasing")
            if row_index and matrix[row_index - 1][column_index] > value:
                raise ValueError("matrix columns must be nondecreasing")

    if not row_count or not column_count:
        return None

    row_index = 0
    column_index = column_count - 1
    first_position: tuple[int, int] | None = None

    while row_index < row_count and column_index >= 0:
        value = matrix[row_index][column_index]
        if value > target:
            column_index -= 1
        elif value < target:
            if first_position is not None:
                return first_position
            row_index += 1
        else:
            first_position = (row_index, column_index)
            column_index -= 1

    return first_position
```

## Example

```python
def row_major_target_oracle(
    matrix: tuple[tuple[int, ...], ...],
    target: int,
) -> tuple[int, int] | None:
    return next(
        (
            (row_index, column_index)
            for row_index, row in enumerate(matrix)
            for column_index, value in enumerate(row)
            if value == target
        ),
        None,
    )


def exercise_tiny_monotone_matrices() -> int:
    from itertools import product

    checked = 0
    for row_count in range(1, 4):
        for column_count in range(1, 4):
            for flat_values in product(range(3), repeat=row_count * column_count):
                matrix = tuple(
                    tuple(
                        flat_values[row_index * column_count + column_index]
                        for column_index in range(column_count)
                    )
                    for row_index in range(row_count)
                )
                rows_sorted = all(
                    row[column_index - 1] <= row[column_index]
                    for row in matrix
                    for column_index in range(1, column_count)
                )
                columns_sorted = all(
                    matrix[row_index - 1][column_index]
                    <= matrix[row_index][column_index]
                    for row_index in range(1, row_count)
                    for column_index in range(column_count)
                )
                if not rows_sorted or not columns_sorted:
                    continue

                for target in range(-1, 4):
                    assert find_first_monotone_matrix_target(
                        matrix, target
                    ) == row_major_target_oracle(matrix, target)
                    checked += 1
    return checked


duplicate_plateau = (
    (1, 2, 2, 2),
    (2, 2, 2, 3),
    (2, 3, 4, 5),
)
cell_boundary = tuple(
    tuple(row_index * 128 + column_index for column_index in range(128))
    for row_index in range(512)
)

rejected = 0
for invalid_matrix in (
    ((0, 1), (2,)),
    ((0, 2, 1),),
    ((0, 2), (1, 1)),
    ((0, True),),
):
    try:
        find_first_monotone_matrix_target(invalid_matrix, 1)
    except (TypeError, ValueError):
        rejected += 1

assert (
    exercise_tiny_monotone_matrices() == 1_650
    and find_first_monotone_matrix_target((), 1) is None
    and find_first_monotone_matrix_target(((), ()), 1) is None
    and find_first_monotone_matrix_target(duplicate_plateau, 2) == (0, 1)
    and find_first_monotone_matrix_target(((1,), (2,), (2,)), 2) == (1, 0)
    and find_first_monotone_matrix_target(cell_boundary, 65_535) == (511, 127)
    and rejected == 4
)
```

## Trade-offs and Limitations

Shape, cell-type, signed-range and monotonicity validation take `O(R * C)`
time before every search. The subsequent staircase visits at most `R + C`
cells and uses `O(1)` auxiliary space. Validation therefore dominates one
call; the search bound is most useful when the matrix's ordering is already a
trusted invariant or validation is retained separately.

Rows and columns are both nondecreasing, so duplicates may form plateaus. The
result orders positions by row first and then column. Continuing left after
the first equality selects the smallest column in the earliest possible row.
Empty and rectangular zero-width inputs have no target position.

The function rejects ragged, descending, partially sorted, non-integer and
out-of-range matrices. It does not return every occurrence, find a nearest
value, repair ordering, mutate the matrix, accelerate a batch of queries, or
provide NumPy-specific behavior.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Bounded Integer Summed-Area Table for Half-Open Rectangle Sums](build-a-bounded-integer-summed-area-table-for-half-open-rectangle-sums.md)
- [Select a Zero-Based Order Statistic from Bounded Integers with Three-Way Quickselect](select-a-zero-based-order-statistic-from-bounded-integers-with-three-way-quickselect.md)
- [Find a Point in Disjoint Half-Open Intervals](find-a-point-in-disjoint-half-open-intervals.md)
<!-- catalog:related:end -->
