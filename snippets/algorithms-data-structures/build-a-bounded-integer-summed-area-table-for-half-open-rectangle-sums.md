---
title: "Build a Bounded Integer Summed-Area Table for Half-Open Rectangle Sums"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - maintain-bounded-point-adds-and-half-open-range-sums-with-a-fenwick-tree.md
  - map-points-between-rectangular-coordinate-spaces.md
  - ../data-processing/transpose-bounded-ragged-exact-string-rows-with-none-for-missing-cells.md
---

# Build a Bounded Integer Summed-Area Table for Half-Open Rectangle Sums

## Idea and Problem

Precompute one immutable two-dimensional prefix table so any rectangular sum needs only four exact integer lookups.

The stored table has a zero sentinel row and column. Each remaining cell holds
the sum above and to its left, including the corresponding source cell. A
half-open query then adds two corners and subtracts the other two without
revisiting the original grid.

## When to Use

Use a summed-area table when one bounded rectangular integer snapshot receives
many rectangle-sum queries but no updates. Half-open bounds compose naturally:
adjacent rectangles may share a boundary without sharing a cell, and an empty
height or width has sum zero.

Use a Fenwick tree or segment tree when values change between queries. Choose a
sparse representation when most of a large coordinate space is absent, and use
an array library when vectorized construction, compact numeric storage, or
multidimensional operations matter more than a standard-library-only object.

## Implementation

```python
from dataclasses import dataclass, field

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_ROWS = 512
_MAX_COLUMNS = 512
_MAX_CELLS = 65_536


class SummedAreaTableError(ValueError):
    """Raised when a grid or rectangle violates the bounded contract."""


@dataclass(frozen=True, slots=True, init=False)
class IntegerSummedAreaTable:
    """Own an immutable two-dimensional prefix table for exact sums."""

    _prefix: tuple[tuple[int, ...], ...] = field(repr=False)

    def __init__(self, rows: tuple[tuple[int, ...], ...]) -> None:
        if type(rows) is not tuple:
            raise TypeError("rows must be an exact tuple")
        if not 1 <= len(rows) <= _MAX_ROWS:
            raise SummedAreaTableError("row count is outside the supported range")

        column_count: int | None = None
        for row_index, row in enumerate(rows):
            if type(row) is not tuple:
                raise TypeError(f"rows[{row_index}] must be an exact tuple")
            if not row:
                raise SummedAreaTableError(f"rows[{row_index}] must not be empty")
            if column_count is None:
                column_count = len(row)
                if column_count > _MAX_COLUMNS:
                    raise SummedAreaTableError("column count exceeds the supported limit")
                if len(rows) * column_count > _MAX_CELLS:
                    raise SummedAreaTableError("cell count exceeds the supported limit")
            elif len(row) != column_count:
                raise SummedAreaTableError("rows must form a rectangular grid")

            for column_index, value in enumerate(row):
                if type(value) is not int:
                    raise TypeError(f"rows[{row_index}][{column_index}] must be an exact integer")
                if not _MIN_INT64 <= value <= _MAX_INT64:
                    raise SummedAreaTableError(
                        f"rows[{row_index}][{column_index}] is outside the signed 64-bit range"
                    )

        if column_count is None:
            raise AssertionError("a non-empty grid must have a column count")

        prefix_rows: list[tuple[int, ...]] = [tuple(0 for _ in range(column_count + 1))]
        for row in rows:
            previous_prefix = prefix_rows[-1]
            current_prefix = [0]
            running_row_sum = 0
            for column_index, value in enumerate(row, start=1):
                running_row_sum += value
                current_prefix.append(previous_prefix[column_index] + running_row_sum)
            prefix_rows.append(tuple(current_prefix))

        object.__setattr__(self, "_prefix", tuple(prefix_rows))

    @property
    def row_count(self) -> int:
        return len(self._prefix) - 1

    @property
    def column_count(self) -> int:
        return len(self._prefix[0]) - 1

    def rectangle_sum(
        self,
        top: int,
        left: int,
        bottom: int,
        right: int,
    ) -> int:
        """Return the exact sum over [top, bottom) x [left, right)."""
        bounds = (
            ("top", top, self.row_count),
            ("bottom", bottom, self.row_count),
            ("left", left, self.column_count),
            ("right", right, self.column_count),
        )
        for name, value, upper_bound in bounds:
            if type(value) is not int:
                raise TypeError(f"{name} must be an exact integer")
            if not 0 <= value <= upper_bound:
                raise IndexError(f"{name} is outside the supported range")
        if top > bottom:
            raise SummedAreaTableError("top must not be greater than bottom")
        if left > right:
            raise SummedAreaTableError("left must not be greater than right")

        prefix = self._prefix
        return prefix[bottom][right] - prefix[top][right] - prefix[bottom][left] + prefix[top][left]
```

## Example

```python
table = IntegerSummedAreaTable(
    (
        (3, -1, 4, 2),
        (0, 5, -2, 1),
        (7, 1, 3, -4),
    )
)

try:
    IntegerSummedAreaTable(((1, 2), (3,)))
except SummedAreaTableError:
    ragged_rejected = True
else:
    ragged_rejected = False

assert (
    table.row_count,
    table.column_count,
    table.rectangle_sum(0, 0, 3, 4),
    table.rectangle_sum(0, 1, 2, 4),
    table.rectangle_sum(1, 0, 3, 2),
    table.rectangle_sum(2, 1, 2, 4),
    ragged_rejected,
) == (3, 4, 19, 9, 13, 0, True)
```

## Trade-offs and Limitations

Complete validation and construction take `O(R*C)` time. The immutable prefix
table uses `O(R*C)` memory, and every rectangle query takes `O(1)` time and
working space. Source cells are signed 64-bit integers, while Python keeps
prefix values and returned sums exact even when an aggregate leaves that range.

The object retains only nested tuples containing the prefix table; it does not
retain or expose mutable source rows. Query bounds are exact integers and use
half-open semantics. Empty rectangles are valid, while reversed or
out-of-range bounds are rejected.

The grid must be non-empty, rectangular, dense, and fully materialized. This
structure does not accept floats, sparse coordinates, NumPy arrays, mutation,
range updates, persistence, synchronization, or higher-dimensional inputs.

## Related Snippets

<!-- catalog:related:start -->
- [Maintain Bounded Point Adds and Half-Open Range Sums with a Fenwick Tree](maintain-bounded-point-adds-and-half-open-range-sums-with-a-fenwick-tree.md)
- [Map Points Between Rectangular Coordinate Spaces](map-points-between-rectangular-coordinate-spaces.md)
- [Transpose Bounded Ragged Exact-String Rows with None for Missing Cells](../data-processing/transpose-bounded-ragged-exact-string-rows-with-none-for-missing-cells.md)
<!-- catalog:related:end -->
