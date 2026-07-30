---
title: "Pivot Bounded Coordinate Records into a Closed Rectangular Table with Collision Evidence"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - transpose-bounded-ragged-exact-string-rows-with-none-for-missing-cells.md
  - select-one-record-per-key-with-an-explicit-ranking-rule.md
  - reshape-repeated-pandas-column-families-with-wide-to-long.md
---

# Pivot Bounded Coordinate Records into a Closed Rectangular Table with Collision Evidence

## Idea and Problem

Place coordinate records into an explicitly declared rectangular domain without hiding duplicate cells or confusing missing data with None.

The row and column tuples define both the closed coordinate space and output
order. After validating every input record, the function reports all repeated
coordinates in domain order. Only collision-free input becomes a dense
immutable table, where a private singleton marks missing coordinates and
ordinary `None` remains a real stored value.

## When to Use

Use this transformation when a bounded batch of coordinate/value records must
fill a known review matrix, feature grid, or interchange table and duplicate
coordinates indicate an upstream contract violation. Explicit axes avoid
silently inventing order from input arrival and make an empty cell auditable.

Use a sparse representation when the declared Cartesian product is large, or
a dataframe pivot with an explicit aggregation when duplicate coordinates are
expected. This function performs no aggregation, type promotion, inferred-axis
discovery, numeric filling, or dataframe index behavior.

## Implementation

```python
from dataclasses import dataclass
from enum import Enum

_MAX_INT64 = 2**63 - 1
_MAX_AXIS_LABELS = 64
_MAX_AXIS_LABEL_BYTES = 128
_MAX_AXIS_TEXT_BYTES = 16_384
_MAX_CELLS = 4_096
_MAX_RECORDS = 4_096
_MAX_STRING_VALUE_BYTES = 1_024
_MAX_RECORD_TEXT_BYTES = 1_048_576


class _MissingCell(Enum):
    MISSING = object()


MISSING = _MissingCell.MISSING


@dataclass(frozen=True, slots=True)
class CoordinateRecord:
    row: str
    column: str
    value: object


@dataclass(frozen=True, slots=True)
class DuplicateCoordinate:
    row: str
    column: str
    positions: tuple[int, ...]


@dataclass(frozen=True, slots=True)
class PivotCollisions:
    duplicates: tuple[DuplicateCoordinate, ...]


@dataclass(frozen=True, slots=True)
class RectangularTable:
    rows: tuple[str, ...]
    columns: tuple[str, ...]
    cells: tuple[tuple[object, ...], ...]


def _validated_axis(value: object, *, field: str) -> tuple[tuple[str, ...], int]:
    if type(value) is not tuple:
        raise TypeError(f"{field} must be an exact tuple")
    if not 1 <= len(value) <= _MAX_AXIS_LABELS:
        raise ValueError(f"{field} count is outside 1..64")

    seen: set[str] = set()
    text_bytes = 0
    for index, label in enumerate(value):
        if type(label) is not str:
            raise TypeError(f"{field}[{index}] must be an exact string")
        try:
            size = len(label.encode("utf-8"))
        except UnicodeEncodeError:
            raise ValueError(f"{field}[{index}] must be UTF-8 encodable") from None
        if not 1 <= size <= _MAX_AXIS_LABEL_BYTES:
            raise ValueError(f"{field}[{index}] is outside its UTF-8 byte limit")
        if label in seen:
            raise ValueError(f"{field} must contain unique labels")
        seen.add(label)
        text_bytes += size
    return value, text_bytes


def _validated_value(value: object, *, field: str) -> int:
    if value is MISSING:
        raise TypeError(f"{field} cannot contain the private MISSING sentinel")
    if value is None or type(value) is bool:
        return 0
    if type(value) is int:
        if not -(2**63) <= value <= _MAX_INT64:
            raise ValueError(f"{field} integer is outside the signed 64-bit range")
        return 0
    if type(value) is str:
        try:
            size = len(value.encode("utf-8"))
        except UnicodeEncodeError:
            raise ValueError(f"{field} must be UTF-8 encodable") from None
        if size > _MAX_STRING_VALUE_BYTES:
            raise ValueError(f"{field} string exceeds its UTF-8 byte limit")
        return size
    raise TypeError(f"{field} must be exactly None, bool, int, or str")


def pivot_coordinate_records(
    rows: tuple[str, ...],
    columns: tuple[str, ...],
    records: tuple[CoordinateRecord, ...],
) -> RectangularTable | PivotCollisions:
    checked_rows, row_bytes = _validated_axis(rows, field="rows")
    checked_columns, column_bytes = _validated_axis(columns, field="columns")
    if row_bytes + column_bytes > _MAX_AXIS_TEXT_BYTES:
        raise ValueError("axis labels exceed the aggregate UTF-8 byte limit")
    if len(checked_rows) * len(checked_columns) > _MAX_CELLS:
        raise ValueError("declared rectangular domain exceeds the cell limit")

    if type(records) is not tuple:
        raise TypeError("records must be an exact tuple")
    if len(records) > _MAX_RECORDS:
        raise ValueError("record count exceeds the supported limit")

    row_indexes = {label: index for index, label in enumerate(checked_rows)}
    column_indexes = {
        label: index for index, label in enumerate(checked_columns)
    }
    positions: dict[tuple[str, str], list[int]] = {}
    record_text_bytes = 0
    for index, record in enumerate(records):
        field = f"records[{index}]"
        if type(record) is not CoordinateRecord:
            raise TypeError(f"{field} must be an exact CoordinateRecord")
        if type(record.row) is not str or type(record.column) is not str:
            raise TypeError(f"{field} coordinates must be exact strings")
        if record.row not in row_indexes or record.column not in column_indexes:
            raise ValueError(f"{field} coordinate is outside the declared domains")
        record_text_bytes += len(record.row.encode("utf-8"))
        record_text_bytes += len(record.column.encode("utf-8"))
        record_text_bytes += _validated_value(record.value, field=f"{field}.value")
        if record_text_bytes > _MAX_RECORD_TEXT_BYTES:
            raise ValueError("record text exceeds the aggregate UTF-8 byte limit")
        positions.setdefault((record.row, record.column), []).append(index)

    duplicates = tuple(
        DuplicateCoordinate(row, column, tuple(positions[(row, column)]))
        for row in checked_rows
        for column in checked_columns
        if len(positions.get((row, column), ())) > 1
    )
    if duplicates:
        return PivotCollisions(duplicates)

    mutable_cells = [
        [MISSING for _ in checked_columns]
        for _ in checked_rows
    ]
    for record in records:
        mutable_cells[row_indexes[record.row]][column_indexes[record.column]] = (
            record.value
        )
    return RectangularTable(
        checked_rows,
        checked_columns,
        tuple(tuple(row) for row in mutable_cells),
    )
```

## Example

```python
def dense_reference(
    rows: tuple[str, ...],
    columns: tuple[str, ...],
    records: tuple[CoordinateRecord, ...],
) -> tuple[tuple[object, ...], ...]:
    by_coordinate = {(record.row, record.column): record.value for record in records}
    return tuple(
        tuple(by_coordinate.get((row, column), MISSING) for column in columns)
        for row in rows
    )


def check_dense_states() -> int:
    from itertools import product

    rows = ("r0", "r1")
    columns = ("c0", "c1", "c2")
    coordinates = tuple(product(rows, columns))
    states = (MISSING, None, False, 0, "x")
    checked = 0
    for cell_states in product(states, repeat=len(coordinates)):
        records = tuple(
            CoordinateRecord(row, column, value)
            for (row, column), value in zip(coordinates, cell_states, strict=True)
            if value is not MISSING
        )
        expected_cells = dense_reference(rows, columns, records)
        for declaration in (records, tuple(reversed(records))):
            observed = pivot_coordinate_records(rows, columns, declaration)
            assert observed == RectangularTable(rows, columns, expected_cells)
        checked += 1
    return checked


checked = check_dense_states()

rows = ("r0", "r1")
columns = ("c0", "c1", "c2")
permuted_records = (
    CoordinateRecord("r0", "c0", None),
    CoordinateRecord("r0", "c2", 3),
    CoordinateRecord("r1", "c1", "value"),
)
expected = pivot_coordinate_records(rows, columns, permuted_records)


def every_permutation_matches() -> bool:
    from itertools import permutations

    return all(
        pivot_coordinate_records(rows, columns, order) == expected
        for order in permutations(permuted_records)
    )

collisions = pivot_coordinate_records(
    rows,
    columns,
    (
        CoordinateRecord("r1", "c2", "a"),
        CoordinateRecord("r0", "c1", 7),
        CoordinateRecord("r1", "c2", "a"),
        CoordinateRecord("r0", "c1", 8),
        CoordinateRecord("r1", "c2", "b"),
    ),
)

assert (
    checked == 15_625
    and every_permutation_matches()
    and collisions
    == PivotCollisions(
        (
            DuplicateCoordinate("r0", "c1", (1, 3)),
            DuplicateCoordinate("r1", "c2", (0, 2, 4)),
        )
    )
)
```

## Trade-offs and Limitations

For `R` records and `C` declared cells, validation and materialization take
`O(R + C + B)` time for `B` inspected UTF-8 bytes and `O(R + C)` memory. The
dense output cost is paid even when most coordinates are missing, so the cell
product is rejected before allocation.

Successful output is independent of record order. Collision evidence instead
retains original positions while ordering coordinates by the declared axes.
The function validates every record before returning that evidence, preventing
an earlier duplicate from hiding a later malformed value or unknown
coordinate.

## Related Snippets

<!-- catalog:related:start -->
- [Transpose Bounded Ragged Exact-String Rows with None for Missing Cells](transpose-bounded-ragged-exact-string-rows-with-none-for-missing-cells.md)
- [Select One Record per Key with an Explicit Ranking Rule](select-one-record-per-key-with-an-explicit-ranking-rule.md)
- [Reshape Repeated pandas Column Families with Wide-to-Long](reshape-repeated-pandas-column-families-with-wide-to-long.md)
<!-- catalog:related:end -->
