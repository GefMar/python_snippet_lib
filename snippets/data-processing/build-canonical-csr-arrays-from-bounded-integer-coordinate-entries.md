---
title: "Build Canonical CSR Arrays from Bounded Integer Coordinate Entries"
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
  - pivot-bounded-coordinate-records-into-a-closed-rectangular-table-with-collision-evidence.md
  - ../algorithms-data-structures/build-a-bounded-integer-summed-area-table-for-half-open-rectangle-sums.md
  - ../algorithms-data-structures/compute-exact-rational-reduced-row-echelon-form-and-rank-for-a-bounded-integer-matrix.md
---

# Build Canonical CSR Arrays from Bounded Integer Coordinate Entries

## Idea and Problem

Convert bounded integer coordinate entries into one canonical compressed sparse row representation without allocating a dense matrix.

Coordinate entries can arrive unordered, repeat the same cell, contain explicit
zeros, or cancel after duplicate values are added. Sort entries by row and
column, sum each equal-coordinate run with exact Python integers, and omit a
run when its total is zero. The remaining coordinates define canonical CSR
arrays independent of input order.

The row-offset array delimits a contiguous slice for every declared row. Within
each slice, column indices strictly increase and align positionally with the
stored values. Empty rows therefore remain visible as adjacent equal offsets
without occupying a dense row-by-column product.

## When to Use

Use this conversion when bounded coordinate triples need one immutable sparse
matrix representation for deterministic comparison, interchange, fixtures, or
a later sparse algorithm. It fits integer data whose duplicate coordinates
mean addition and whose zero totals should not consume stored entries.

Use a maintained sparse-array library when matrices are large, operations must
be vectorized, index dtypes or numeric promotion matter, or the result must
interoperate directly with a scientific-computing API. Preserve coordinate
records instead when duplicate occurrences, explicit zeros, or arrival order
carry information that aggregation would destroy.

## Implementation

```python
from dataclasses import dataclass

_MAX_DIMENSION = 4_096
_MAX_COORDINATE_ENTRY_COUNT = 65_536
_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1


@dataclass(frozen=True, slots=True)
class IntegerCoordinateEntry:
    row: int
    column: int
    value: int


@dataclass(frozen=True, slots=True)
class CanonicalCSR:
    row_count: int
    column_count: int
    row_offsets: tuple[int, ...]
    column_indices: tuple[int, ...]
    values: tuple[int, ...]


def build_canonical_csr(
    entries: tuple[IntegerCoordinateEntry, ...],
    *,
    row_count: int,
    column_count: int,
) -> CanonicalCSR:
    """Aggregate bounded coordinate entries into canonical CSR arrays."""
    if type(row_count) is not int:
        raise TypeError("row_count must be an exact integer")
    if not 1 <= row_count <= _MAX_DIMENSION:
        raise ValueError("row_count is outside the supported range")
    if type(column_count) is not int:
        raise TypeError("column_count must be an exact integer")
    if not 1 <= column_count <= _MAX_DIMENSION:
        raise ValueError("column_count is outside the supported range")
    if type(entries) is not tuple:
        raise TypeError("entries must be an exact tuple")
    if len(entries) > _MAX_COORDINATE_ENTRY_COUNT:
        raise ValueError("entry count exceeds the supported limit")

    ordered: list[tuple[int, int, int]] = []
    for index, entry in enumerate(entries):
        if type(entry) is not IntegerCoordinateEntry:
            raise TypeError(f"entries[{index}] must be an exact coordinate entry")
        if type(entry.row) is not int:
            raise TypeError(f"entries[{index}].row must be an exact integer")
        if not 0 <= entry.row < row_count:
            raise ValueError(f"entries[{index}].row is outside the matrix")
        if type(entry.column) is not int:
            raise TypeError(f"entries[{index}].column must be an exact integer")
        if not 0 <= entry.column < column_count:
            raise ValueError(f"entries[{index}].column is outside the matrix")
        if type(entry.value) is not int:
            raise TypeError(f"entries[{index}].value must be an exact integer")
        if not _MIN_INT64 <= entry.value <= _MAX_INT64:
            raise ValueError(f"entries[{index}].value is outside the signed 64-bit range")
        ordered.append((entry.row, entry.column, entry.value))

    ordered.sort(key=lambda item: (item[0], item[1]))
    aggregates: list[tuple[int, int, int]] = []
    position = 0
    while position < len(ordered):
        row, column, value = ordered[position]
        total = value
        position += 1
        while (
            position < len(ordered)
            and ordered[position][0] == row
            and ordered[position][1] == column
        ):
            total += ordered[position][2]
            position += 1
        if total != 0:
            aggregates.append((row, column, total))

    row_offsets = [0]
    column_indices: list[int] = []
    values: list[int] = []
    aggregate_index = 0
    for row in range(row_count):
        while aggregate_index < len(aggregates) and aggregates[aggregate_index][0] == row:
            _, column, value = aggregates[aggregate_index]
            column_indices.append(column)
            values.append(value)
            aggregate_index += 1
        row_offsets.append(len(values))

    if aggregate_index != len(aggregates):
        raise AssertionError("every aggregate must belong to one declared row")
    return CanonicalCSR(
        row_count=row_count,
        column_count=column_count,
        row_offsets=tuple(row_offsets),
        column_indices=tuple(column_indices),
        values=tuple(values),
    )
```

## Example

```python
def dense_csr_oracle(
    entries: tuple[IntegerCoordinateEntry, ...],
    *,
    row_count: int,
    column_count: int,
) -> CanonicalCSR:
    dense = [[0] * column_count for _ in range(row_count)]
    for entry in entries:
        dense[entry.row][entry.column] += entry.value

    row_offsets = [0]
    column_indices: list[int] = []
    values: list[int] = []
    for row in dense:
        for column, value in enumerate(row):
            if value != 0:
                column_indices.append(column)
                values.append(value)
        row_offsets.append(len(values))
    return CanonicalCSR(
        row_count,
        column_count,
        tuple(row_offsets),
        tuple(column_indices),
        tuple(values),
    )


def coordinate_triples(csr: CanonicalCSR) -> tuple[tuple[int, int, int], ...]:
    from itertools import pairwise

    triples: list[tuple[int, int, int]] = []
    for row, (start, stop) in enumerate(pairwise(csr.row_offsets)):
        triples.extend(
            (row, csr.column_indices[index], csr.values[index]) for index in range(start, stop)
        )
    return tuple(triples)


def has_strict_csr_invariants(csr: CanonicalCSR) -> bool:
    from itertools import pairwise

    if len(csr.row_offsets) != csr.row_count + 1 or csr.row_offsets[0] != 0:
        return False
    if csr.row_offsets[-1] != len(csr.values):
        return False
    if len(csr.column_indices) != len(csr.values):
        return False
    if any(left > right for left, right in pairwise(csr.row_offsets)):
        return False
    for start, stop in pairwise(csr.row_offsets):
        columns = csr.column_indices[start:stop]
        if any(left >= right for left, right in pairwise(columns)):
            return False
    return all(0 <= column < csr.column_count for column in csr.column_indices) and all(
        value != 0 for value in csr.values
    )


entries = (
    IntegerCoordinateEntry(2, 1, 5),
    IntegerCoordinateEntry(0, 2, 4),
    IntegerCoordinateEntry(2, 1, -5),
    IntegerCoordinateEntry(3, 0, -1),
    IntegerCoordinateEntry(0, 0, 7),
    IntegerCoordinateEntry(3, 0, 3),
)
expected = dense_csr_oracle(entries, row_count=4, column_count=4)


def verify_permutations() -> int:
    from itertools import permutations

    checked = 0
    for permutation in permutations(entries):
        assert (
            build_canonical_csr(
                permutation,
                row_count=4,
                column_count=4,
            )
            == expected
        )
        checked += 1
    return checked


checked_permutations = verify_permutations()

empty = build_canonical_csr((), row_count=3, column_count=2)
wide_aggregate = build_canonical_csr(
    (
        IntegerCoordinateEntry(0, 0, _MAX_INT64),
        IntegerCoordinateEntry(0, 0, _MAX_INT64),
    ),
    row_count=1,
    column_count=1,
)


def rejected(
    candidate_entries: object,
    candidate_row_count: object,
    candidate_column_count: object,
) -> bool:
    try:
        build_canonical_csr(
            candidate_entries,
            row_count=candidate_row_count,
            column_count=candidate_column_count,
        )
    except (TypeError, ValueError):
        return True
    return False


invalid_calls = (
    ([], 1, 1),
    ((IntegerCoordinateEntry(True, 0, 1),), 1, 1),
    ((IntegerCoordinateEntry(1, 0, 1),), 1, 1),
    ((IntegerCoordinateEntry(0, 1, 1),), 1, 1),
    ((IntegerCoordinateEntry(0, 0, _MAX_INT64 + 1),), 1, 1),
    ((), True, 1),
    ((), 1, 0),
)

assert (
    expected,
    coordinate_triples(expected),
    has_strict_csr_invariants(expected),
    checked_permutations,
    empty,
    wide_aggregate.values,
    sum(rejected(*call) for call in invalid_calls),
) == (
    CanonicalCSR(4, 4, (0, 2, 2, 2, 3), (0, 2, 0), (7, 4, 2)),
    ((0, 0, 7), (0, 2, 4), (3, 0, 2)),
    True,
    720,
    CanonicalCSR(3, 2, (0, 0, 0, 0), (), ()),
    (2 * _MAX_INT64,),
    len(invalid_calls),
)
```

## Trade-offs and Limitations

For `R` input records, `D` nonzero aggregated coordinates, and `M` declared
rows, validation and sorting take `O(R log R)` time. Run aggregation takes
`O(R)` time, and CSR construction takes `O(M + D)` time. The sorted copy,
aggregates, and result use `O(R + M + D)` additional references and integers;
the implementation never allocates the `row_count * column_count` dense
product.

Every input value is a signed-64 integer, but duplicate addition uses arbitrary
Python integers and an emitted aggregate may exceed that range. A coordinate
whose exact total is zero is absent from the result. Sorting and exact addition
make the output independent of input order, while the declared dimensions
preserve trailing empty rows and columns.

The output is a frozen Python representation, not a SciPy sparse matrix or a
wire format. It does not retain input positions, duplicate multiplicity, or
explicit zeros; multiply matrices; transpose them; choose an index dtype;
convert values to floating point; or optimize repeated incremental updates.

## Related Snippets

<!-- catalog:related:start -->
- [Pivot Bounded Coordinate Records into a Closed Rectangular Table with Collision Evidence](pivot-bounded-coordinate-records-into-a-closed-rectangular-table-with-collision-evidence.md)
- [Build a Bounded Integer Summed-Area Table for Half-Open Rectangle Sums](../algorithms-data-structures/build-a-bounded-integer-summed-area-table-for-half-open-rectangle-sums.md)
- [Compute Exact Rational Reduced Row-Echelon Form and Rank for a Bounded Integer Matrix](../algorithms-data-structures/compute-exact-rational-reduced-row-echelon-form-and-rank-for-a-bounded-integer-matrix.md)
<!-- catalog:related:end -->
