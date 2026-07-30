---
title: "Find the Canonical Maximum-Sum Non-Empty Subrectangle of a Bounded Integer Matrix"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - find-a-maximum-sum-non-empty-contiguous-integer-subarray-with-explicit-ties.md
  - build-a-bounded-integer-summed-area-table-for-half-open-rectangle-sums.md
  - find-the-canonical-largest-rectangle-under-a-bounded-integer-histogram.md
---

# Find the Canonical Maximum-Sum Non-Empty Subrectangle of a Bounded Integer Matrix

## Idea and Problem

Find one non-empty axis-aligned matrix rectangle with the greatest exact sum and deterministic coordinate ties.

The input is an exact, non-empty rectangular tuple of tuples with at most 64 rows, 64 columns, and 4,096 cells. Every cell must be an exact `int` in the signed 64-bit range; booleans, ragged rows, and integer subclasses are rejected.

For every pair of row boundaries, compress the rows between them into one array of column sums. A linear maximum-subarray scan then finds the best non-empty column interval for that row band. Python integers preserve the exact sum even when adding valid cells exceeds the signed 64-bit range.

The result maximizes `total` first, then minimizes `(top, left, bottom, right)` lexicographically. Coordinates are half-open: selected row indices satisfy `top <= row < bottom`, and selected column indices satisfy `left <= column < right`. Keeping the earlier start when a one-dimensional scan ties makes a zero-only matrix choose `(0, 0, 1, 1)`; the global coordinate comparison similarly makes an all-negative matrix choose the earliest cell containing its largest value.

## When to Use

Use this for a small or medium dense integer grid when one exact, reproducible best region is needed, such as choosing a profitable block of time-and-location measurements or finding the strongest contiguous signal patch.

A summed-area table is a better fit when many already-known rectangles need to be queried. For large sparse matrices, floating-point data, frequent updates, or array-library integration, choose a representation and algorithm designed for those requirements.

## Implementation

```python
from dataclasses import dataclass

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_ROWS = 64
_MAX_COLUMNS = 64
_MAX_CELLS = 4_096


@dataclass(frozen=True, slots=True)
class MaximumSumSubrectangle:
    total: int
    top: int
    left: int
    bottom: int
    right: int


def maximum_sum_subrectangle(
    matrix: tuple[tuple[int, ...], ...],
) -> MaximumSumSubrectangle:
    """Return the canonical maximum-sum non-empty rectangle in ``matrix``."""
    if type(matrix) is not tuple:
        raise TypeError("matrix must be an exact tuple")

    row_count = len(matrix)
    if not 1 <= row_count <= _MAX_ROWS:
        raise ValueError("matrix must contain between 1 and 64 rows")

    column_count: int | None = None
    for row in matrix:
        if type(row) is not tuple:
            raise TypeError("each row must be an exact tuple")
        if column_count is None:
            column_count = len(row)
            if not 1 <= column_count <= _MAX_COLUMNS:
                raise ValueError("matrix must contain between 1 and 64 columns")
        elif len(row) != column_count:
            raise ValueError("matrix must be rectangular")

        for value in row:
            if type(value) is not int:
                raise TypeError("each cell must be an exact int")
            if not _MIN_INT64 <= value <= _MAX_INT64:
                raise ValueError("each cell must fit in a signed 64-bit integer")

    assert column_count is not None
    if row_count * column_count > _MAX_CELLS:
        raise ValueError("matrix must contain at most 4,096 cells")

    best = MaximumSumSubrectangle(
        total=matrix[0][0],
        top=0,
        left=0,
        bottom=1,
        right=1,
    )

    for top in range(row_count):
        column_sums = [0] * column_count
        for bottom in range(top + 1, row_count + 1):
            row = matrix[bottom - 1]
            for column, value in enumerate(row):
                column_sums[column] += value

            current_total = column_sums[0]
            current_left = 0
            for right in range(1, column_count + 1):
                if right > 1:
                    value = column_sums[right - 1]
                    extended_total = current_total + value
                    if value > extended_total:
                        current_total = value
                        current_left = right - 1
                    else:
                        current_total = extended_total

                coordinates = (top, current_left, bottom, right)
                best_coordinates = (best.top, best.left, best.bottom, best.right)
                if current_total > best.total or (
                    current_total == best.total and coordinates < best_coordinates
                ):
                    best = MaximumSumSubrectangle(
                        total=current_total,
                        top=top,
                        left=current_left,
                        bottom=bottom,
                        right=right,
                    )

    return best
```

## Example

```python


scores = (
    (-4, 2, -1, -3),
    (3, 5, -2, 1),
    (-6, 2, 4, 2),
)

best = maximum_sum_subrectangle(scores)
assert best == MaximumSumSubrectangle(
    total=12,
    top=1,
    left=1,
    bottom=3,
    right=4,
)

all_negative = maximum_sum_subrectangle(((-5, -2), (-2, -7)))
assert all_negative == MaximumSumSubrectangle(-2, 0, 1, 1, 2)

all_zero = maximum_sum_subrectangle(((0, 0), (0, 0)))
assert all_zero == MaximumSumSubrectangle(0, 0, 0, 1, 1)

try:
    maximum_sum_subrectangle(((1, 2), (3,)))
except ValueError:
    ragged_rejected = True
else:
    ragged_rejected = False

assert ragged_rejected
```

## Trade-offs and Limitations

The algorithm takes `O(rows² * columns)` time and `O(columns)` auxiliary space. It deliberately keeps rows as the quadratic dimension so the stated complexity and tie order remain stable; it does not transpose the matrix automatically.

This contract does not return an empty rectangle or every optimal rectangle. It also excludes ragged or sparse inputs, floating-point values, wrapping rectangles, higher-dimensional boxes, streaming or mutable matrices, dynamic updates, NumPy-specific behavior, and dimensions beyond the documented limits.

## Related Snippets

<!-- catalog:related:start -->
- [Find a Maximum-Sum Non-Empty Contiguous Integer Subarray with Explicit Ties](find-a-maximum-sum-non-empty-contiguous-integer-subarray-with-explicit-ties.md)
- [Build a Bounded Integer Summed-Area Table for Half-Open Rectangle Sums](build-a-bounded-integer-summed-area-table-for-half-open-rectangle-sums.md)
- [Find the Canonical Largest Rectangle Under a Bounded Integer Histogram](find-the-canonical-largest-rectangle-under-a-bounded-integer-histogram.md)
<!-- catalog:related:end -->
