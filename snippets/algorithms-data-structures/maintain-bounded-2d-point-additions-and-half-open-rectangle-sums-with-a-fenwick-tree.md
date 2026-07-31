---
title: "Maintain Bounded 2D Point Additions and Half-Open Rectangle Sums with a Fenwick Tree"
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
  - maintain-bounded-point-adds-and-half-open-range-sums-with-a-fenwick-tree.md
  - build-a-bounded-integer-summed-area-table-for-half-open-rectangle-sums.md
---

# Maintain Bounded 2D Point Additions and Half-Open Rectangle Sums with a Fenwick Tree

## Idea and Problem

Maintain exact sums in a bounded, zero-initialized integer grid while individual cells receive additions and rectangular queries remain logarithmic in both dimensions.

A two-dimensional Fenwick tree extends the one-dimensional structure in both
axes. Each stored entry represents a power-of-two-aligned rectangle. A point
addition visits every covering entry, and four two-dimensional prefix sums
produce the sum over `[row_start, row_stop) x [column_start, column_stop)`.

## When to Use

Use this structure for a fixed-shape, dense in-memory grid with many interleaved
point additions and rectangle-sum queries. It is useful for mutable frequency
tables, bounded spatial counters, and matrix-like accumulators when exact Python
integer sums matter.

The one-dimensional Fenwick variant is smaller and faster when there is only one
index axis. A static summed-area table answers rectangle sums in constant time
and is preferable when cells never change. Choose another representation for
sparse coordinates, point assignments, rectangle updates, immutable snapshots,
or concurrent access.

## Implementation

```python
_MIN_FENWICK_GRID_INT64 = -(1 << 63)
_MAX_FENWICK_GRID_INT64 = (1 << 63) - 1
_MAX_FENWICK_GRID_DIMENSION = 65_536
_MAX_FENWICK_GRID_CELLS = 262_144


class FenwickRectangleSums:
    """Maintain point additions and exact half-open sums in an owned grid."""

    __slots__ = ("_columns", "_rows", "_tree", "_values")

    def __init__(self, rows: int, columns: int) -> None:
        if type(rows) is not int or type(columns) is not int:
            raise TypeError("rows and columns must be exact integers")
        if not 1 <= rows <= _MAX_FENWICK_GRID_DIMENSION:
            raise ValueError("rows is outside the supported range")
        if not 1 <= columns <= _MAX_FENWICK_GRID_DIMENSION:
            raise ValueError("columns is outside the supported range")
        if rows * columns > _MAX_FENWICK_GRID_CELLS:
            raise ValueError("the grid contains more than 262,144 cells")

        self._rows = rows
        self._columns = columns
        self._values = [[0] * columns for _ in range(rows)]
        self._tree = [[0] * (columns + 1) for _ in range(rows + 1)]

    @staticmethod
    def _validated_position(value: object, *, name: str, size: int) -> int:
        if type(value) is not int:
            raise TypeError(f"{name} must be an exact integer")
        if not 0 <= value < size:
            raise IndexError(f"{name} is outside the supported range")
        return value

    @staticmethod
    def _validated_bound(value: object, *, name: str, size: int) -> int:
        if type(value) is not int:
            raise TypeError(f"{name} must be an exact integer")
        if not 0 <= value <= size:
            raise IndexError(f"{name} is outside the supported range")
        return value

    def add(self, row: int, column: int, delta: int) -> None:
        """Add delta to one cell while keeping that cell in signed int64."""
        checked_row = self._validated_position(row, name="row", size=self._rows)
        checked_column = self._validated_position(
            column,
            name="column",
            size=self._columns,
        )
        if type(delta) is not int:
            raise TypeError("delta must be an exact integer")
        if not _MIN_FENWICK_GRID_INT64 <= delta <= _MAX_FENWICK_GRID_INT64:
            raise ValueError("delta is outside the signed 64-bit range")

        updated_value = self._values[checked_row][checked_column] + delta
        if not _MIN_FENWICK_GRID_INT64 <= updated_value <= _MAX_FENWICK_GRID_INT64:
            raise ValueError("the updated point would leave the signed 64-bit range")

        self._values[checked_row][checked_column] = updated_value
        tree_row = checked_row + 1
        while tree_row <= self._rows:
            tree_column = checked_column + 1
            while tree_column <= self._columns:
                self._tree[tree_row][tree_column] += delta
                tree_column += tree_column & -tree_column
            tree_row += tree_row & -tree_row

    def _prefix_sum(self, row_stop: int, column_stop: int) -> int:
        total = 0
        tree_row = row_stop
        while tree_row > 0:
            tree_column = column_stop
            while tree_column > 0:
                total += self._tree[tree_row][tree_column]
                tree_column -= tree_column & -tree_column
            tree_row -= tree_row & -tree_row
        return total

    def rectangle_sum(
        self,
        row_start: int,
        column_start: int,
        row_stop: int,
        column_stop: int,
    ) -> int:
        """Return the exact sum over the requested half-open rectangle."""
        checked_row_start = self._validated_bound(
            row_start,
            name="row_start",
            size=self._rows,
        )
        checked_row_stop = self._validated_bound(
            row_stop,
            name="row_stop",
            size=self._rows,
        )
        checked_column_start = self._validated_bound(
            column_start,
            name="column_start",
            size=self._columns,
        )
        checked_column_stop = self._validated_bound(
            column_stop,
            name="column_stop",
            size=self._columns,
        )
        if checked_row_start > checked_row_stop:
            raise ValueError("row_start must not be greater than row_stop")
        if checked_column_start > checked_column_stop:
            raise ValueError("column_start must not be greater than column_stop")

        return (
            self._prefix_sum(checked_row_stop, checked_column_stop)
            - self._prefix_sum(checked_row_start, checked_column_stop)
            - self._prefix_sum(checked_row_stop, checked_column_start)
            + self._prefix_sum(checked_row_start, checked_column_start)
        )
```

## Example

```python
from itertools import product
from random import Random


def direct_rectangle_sum(
    grid: list[list[int]],
    row_start: int,
    column_start: int,
    row_stop: int,
    column_stop: int,
) -> int:
    return sum(
        grid[row][column]
        for row in range(row_start, row_stop)
        for column in range(column_start, column_stop)
    )


def assert_all_rectangles(
    indexed: FenwickRectangleSums,
    grid: list[list[int]],
) -> int:
    rows = len(grid)
    columns = len(grid[0])
    checked = 0
    for row_start in range(rows + 1):
        for row_stop in range(row_start, rows + 1):
            for column_start in range(columns + 1):
                for column_stop in range(column_start, columns + 1):
                    actual = indexed.rectangle_sum(
                        row_start,
                        column_start,
                        row_stop,
                        column_stop,
                    )
                    expected = direct_rectangle_sum(
                        grid,
                        row_start,
                        column_start,
                        row_stop,
                        column_stop,
                    )
                    assert actual == expected
                    checked += 1
    return checked


# Exhaust every update trace of up to three operations on a 2-by-2 grid,
# checking every valid rectangle after each operation.
update_options = tuple(
    (row, column, delta) for row in range(2) for column in range(2) for delta in (-2, 0, 3)
)
exhaustive_checks = 0
for trace_length in range(4):
    for updates in product(update_options, repeat=trace_length):
        indexed = FenwickRectangleSums(2, 2)
        dense = [[0, 0], [0, 0]]
        exhaustive_checks += assert_all_rectangles(indexed, dense)
        for row, column, delta in updates:
            indexed.add(row, column, delta)
            dense[row][column] += delta
            exhaustive_checks += assert_all_rectangles(indexed, dense)


# Compare seeded mixed traces with a separately maintained dense oracle.
rng = Random(0x2DFE_71C)
random_checks = 0
for _ in range(40):
    rows = rng.randint(1, 6)
    columns = rng.randint(1, 6)
    indexed = FenwickRectangleSums(rows, columns)
    dense = [[0] * columns for _ in range(rows)]
    for _ in range(100):
        if rng.randrange(3):
            row = rng.randrange(rows)
            column = rng.randrange(columns)
            delta = rng.randint(-1_000, 1_000)
            indexed.add(row, column, delta)
            dense[row][column] += delta
        else:
            row_start, row_stop = sorted((rng.randrange(rows + 1), rng.randrange(rows + 1)))
            column_start, column_stop = sorted(
                (rng.randrange(columns + 1), rng.randrange(columns + 1))
            )
            assert indexed.rectangle_sum(
                row_start,
                column_start,
                row_stop,
                column_stop,
            ) == direct_rectangle_sum(
                dense,
                row_start,
                column_start,
                row_stop,
                column_stop,
            )
            random_checks += 1


# Overflow rejection is atomic at both signed boundaries.
upper = FenwickRectangleSums(1, 1)
upper.add(0, 0, _MAX_FENWICK_GRID_INT64)
try:
    upper.add(0, 0, 1)
except ValueError:
    upper_overflow_rejected = True
else:
    upper_overflow_rejected = False

lower = FenwickRectangleSums(1, 1)
lower.add(0, 0, _MIN_FENWICK_GRID_INT64)
try:
    lower.add(0, 0, -1)
except ValueError:
    lower_overflow_rejected = True
else:
    lower_overflow_rejected = False


# Exercise the maximum column bound, the maximum cell product, and an edge cell.
skinny = FenwickRectangleSums(1, _MAX_FENWICK_GRID_DIMENSION)
maximum_grid = FenwickRectangleSums(4, _MAX_FENWICK_GRID_DIMENSION)
maximum_grid.add(3, _MAX_FENWICK_GRID_DIMENSION - 1, 17)

assert (
    exhaustive_checks,
    random_checks > 1_000,
    upper_overflow_rejected,
    upper.rectangle_sum(0, 0, 1, 1),
    lower_overflow_rejected,
    lower.rectangle_sum(0, 0, 1, 1),
    skinny.rectangle_sum(0, 0, 1, _MAX_FENWICK_GRID_DIMENSION),
    maximum_grid.rectangle_sum(
        0,
        0,
        4,
        _MAX_FENWICK_GRID_DIMENSION,
    ),
) == (
    265_284,
    True,
    True,
    _MAX_FENWICK_GRID_INT64,
    True,
    _MIN_FENWICK_GRID_INT64,
    0,
    17,
)
```

## Trade-offs and Limitations

Initialization takes `O(rows * columns)` time and memory. A point addition and a
rectangle sum each take `O(log rows * log columns)` time; a rectangle query
performs four prefix traversals. The separately owned cell grid doubles the
asymptotic storage, but makes signed-64-bit point validation possible.

Dimensions are positive exact integers no greater than 65,536, and their product
cannot exceed 262,144. Deltas and stored cells remain signed 64-bit integers;
Fenwick aggregates and returned sums use exact Python integers. All validation,
including the resulting-cell overflow check, completes before an update mutates
either owned representation.

Queries use half-open bounds. A valid rectangle with zero height or width
returns zero, while reversed or out-of-bounds rectangles are rejected. Updates
are additions, not assignments. The dense object provides no rectangle updates,
sparse storage, resizing, persistence, serialization, or synchronization.

## Related Snippets

<!-- catalog:related:start -->
- [Maintain Bounded Point Adds and Half-Open Range Sums with a Fenwick Tree](maintain-bounded-point-adds-and-half-open-range-sums-with-a-fenwick-tree.md)
- [Build a Bounded Integer Summed-Area Table for Half-Open Rectangle Sums](build-a-bounded-integer-summed-area-table-for-half-open-rectangle-sums.md)
<!-- catalog:related:end -->
