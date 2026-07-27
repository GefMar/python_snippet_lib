---
title: "Merge Bounded Row Batches with a First-Seen Schema Union"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - validate-parsed-csv-rows-with-bounded-structured-problems.md
  - normalize-optional-csv-columns-in-a-single-pass.md
  - ../python-language/make-a-defensive-copy-at-a-mutable-input-boundary.md
---

# Merge Bounded Row Batches with a First-Seen Schema Union

## Idea and Problem

Merge immutable in-memory row batches into one typed table while preserving each payload column's first appearance as the union order.

Every output row receives fixed batch and one-based row lineage. A payload
column keeps one exact scalar kind across batches; if any batch omits it, the
union marks it nullable and inserts `None` for that batch's rows. Batch order
and row order are retained, so neither dictionary ordering nor schema sorting
quietly changes the result.

## When to Use

Use this algorithm after independent bounded readers have already produced
immutable `RowBatch` values. It fits small history snapshots or partition
results that may add columns over time but still share strict scalar semantics.
Supported cells are exact `bool`, signed 64-bit-range `int`, finite `float`,
bounded `str`, bounded `bytes`, and `None` where the input schema explicitly
allows nulls.

Each batch supplies an exact tuple schema and exact tuple rows. Column names are
bounded ASCII identifiers, batch IDs are bounded printable text, and the
reserved `source_batch_id` and `source_row_number` fields belong to the
merger. Use a schema registry or explicit migration when a shared name changes
meaning or type. Persistence, file discovery, retention, and deletion policy
remain outside this in-memory boundary.

## Implementation

```python
import math
import re
from dataclasses import dataclass
from typing import Literal


ScalarKind = Literal["bool", "bytes", "float", "int", "str"]
ScalarValue = bool | bytes | float | int | str | None

_MAX_BATCHES = 64
_MAX_ROWS_PER_BATCH = 10_000
_MAX_TOTAL_ROWS = 50_000
_MAX_COLUMNS = 128
_MAX_INPUT_CELLS = 500_000
_MAX_OUTPUT_CELLS = 1_000_000
_MAX_BATCH_ID_CHARACTERS = 128
_MAX_TEXT_CHARACTERS = 4_096
_MAX_BYTES_LENGTH = 4_096
_MIN_INTEGER = -(1 << 63)
_MAX_INTEGER = (1 << 63) - 1
_COLUMN_NAME = re.compile(r"[A-Za-z][A-Za-z0-9_]{0,63}", re.ASCII)
_SCALAR_KINDS = frozenset({"bool", "bytes", "float", "int", "str"})
_BATCH_ID_COLUMN = "source_batch_id"
_ROW_NUMBER_COLUMN = "source_row_number"
_RESERVED_COLUMNS = frozenset({_BATCH_ID_COLUMN, _ROW_NUMBER_COLUMN})


@dataclass(frozen=True, slots=True)
class ColumnSpec:
    name: str
    kind: ScalarKind
    nullable: bool = False


@dataclass(frozen=True, slots=True)
class RowBatch:
    batch_id: str
    columns: tuple[ColumnSpec, ...]
    rows: tuple[tuple[ScalarValue, ...], ...]


@dataclass(frozen=True, slots=True)
class MergedRows:
    columns: tuple[ColumnSpec, ...]
    rows: tuple[tuple[ScalarValue, ...], ...]


def _preflight_batches(value: object) -> tuple[tuple[RowBatch, ...], int]:
    if type(value) is not tuple:
        raise TypeError("batches must be an exact tuple")
    if not 1 <= len(value) <= _MAX_BATCHES:
        raise ValueError("batch count is outside the supported range")

    total_rows = 0
    total_cells = 0
    batches = []
    for batch in value:
        if type(batch) is not RowBatch:
            raise TypeError("batches must contain exact RowBatch values")
        if type(batch.columns) is not tuple:
            raise TypeError("batch columns must be an exact tuple")
        if not 1 <= len(batch.columns) <= _MAX_COLUMNS:
            raise ValueError("batch column count is outside the supported range")
        if type(batch.rows) is not tuple:
            raise TypeError("batch rows must be an exact tuple")
        if len(batch.rows) > _MAX_ROWS_PER_BATCH:
            raise ValueError("a batch exceeds the supported row limit")

        total_rows += len(batch.rows)
        if total_rows > _MAX_TOTAL_ROWS:
            raise ValueError("batches exceed the supported total row limit")
        for row in batch.rows:
            if type(row) is not tuple:
                raise TypeError("batch rows must contain exact tuples")
            if len(row) > _MAX_COLUMNS:
                raise ValueError("a row exceeds the supported column limit")
            total_cells += len(row)
            if total_cells > _MAX_INPUT_CELLS:
                raise ValueError("batches exceed the supported input cell limit")
            if len(row) != len(batch.columns):
                raise ValueError("a row width does not match its batch schema")
        batches.append(batch)
    return tuple(batches), total_rows


def _batch_id(value: object) -> str:
    if type(value) is not str:
        raise TypeError("batch IDs must be exact strings")
    if not value or len(value) > _MAX_BATCH_ID_CHARACTERS:
        raise ValueError("batch ID length is outside the supported range")
    if value != value.strip() or not value.isprintable():
        raise ValueError("batch IDs must be stripped printable text")
    return value


def _column_spec(value: object) -> ColumnSpec:
    if type(value) is not ColumnSpec:
        raise TypeError("schemas must contain exact ColumnSpec values")
    if type(value.name) is not str:
        raise TypeError("column names must be exact strings")
    if _COLUMN_NAME.fullmatch(value.name) is None:
        raise ValueError("a column name is outside the supported format")
    if value.name in _RESERVED_COLUMNS:
        raise ValueError("a batch schema collides with a reserved lineage field")
    if type(value.kind) is not str:
        raise TypeError("scalar kinds must be exact strings")
    if value.kind not in _SCALAR_KINDS:
        raise ValueError("a scalar kind is not supported")
    if type(value.nullable) is not bool:
        raise TypeError("column nullability must be an exact bool")
    return ColumnSpec(value.name, value.kind, value.nullable)


def _validate_schemas(
    batches: tuple[RowBatch, ...],
) -> tuple[
    tuple[str, ...],
    tuple[tuple[ColumnSpec, ...], ...],
    tuple[ColumnSpec, ...],
]:
    batch_ids = []
    known_batch_ids: set[str] = set()
    schemas = []
    union_order: list[str] = []
    union_kinds: dict[str, ScalarKind] = {}
    union_nullable: dict[str, bool] = {}
    schema_presence: dict[str, int] = {}

    for batch in batches:
        identifier = _batch_id(batch.batch_id)
        if identifier in known_batch_ids:
            raise ValueError("batch IDs must be unique")
        known_batch_ids.add(identifier)
        batch_ids.append(identifier)

        schema = []
        names_in_batch: set[str] = set()
        for raw_spec in batch.columns:
            spec = _column_spec(raw_spec)
            if spec.name in names_in_batch:
                raise ValueError("column names within a batch must be unique")
            names_in_batch.add(spec.name)
            schema.append(spec)

            known_kind = union_kinds.get(spec.name)
            if known_kind is None:
                if len(union_order) >= _MAX_COLUMNS:
                    raise ValueError("schema union exceeds the supported column limit")
                union_order.append(spec.name)
                union_kinds[spec.name] = spec.kind
                union_nullable[spec.name] = spec.nullable
                schema_presence[spec.name] = 1
            else:
                if known_kind != spec.kind:
                    raise ValueError("a column has conflicting scalar kinds")
                union_nullable[spec.name] |= spec.nullable
                schema_presence[spec.name] += 1
        schemas.append(tuple(schema))

    union = tuple(
        ColumnSpec(
            name=name,
            kind=union_kinds[name],
            nullable=(
                union_nullable[name]
                or schema_presence[name] < len(batches)
            ),
        )
        for name in union_order
    )
    return tuple(batch_ids), tuple(schemas), union


def _validate_cell(value: object, *, spec: ColumnSpec) -> ScalarValue:
    if value is None:
        if not spec.nullable:
            raise ValueError(f"column {spec.name!r} does not allow nulls")
        return None

    expected_type = {
        "bool": bool,
        "bytes": bytes,
        "float": float,
        "int": int,
        "str": str,
    }[spec.kind]
    if type(value) is not expected_type:
        raise TypeError(f"column {spec.name!r} contains the wrong scalar type")

    if spec.kind == "int" and not _MIN_INTEGER <= value <= _MAX_INTEGER:
        raise ValueError(f"column {spec.name!r} contains an out-of-range integer")
    if spec.kind == "float" and not math.isfinite(value):
        raise ValueError(f"column {spec.name!r} contains a non-finite float")
    if spec.kind == "str" and len(value) > _MAX_TEXT_CHARACTERS:
        raise ValueError(f"column {spec.name!r} contains oversized text")
    if spec.kind == "bytes" and len(value) > _MAX_BYTES_LENGTH:
        raise ValueError(f"column {spec.name!r} contains oversized bytes")
    return value


def merge_bounded_row_batches(
    batches: tuple[RowBatch, ...],
) -> MergedRows:
    prepared, total_rows = _preflight_batches(batches)
    batch_ids, schemas, union = _validate_schemas(prepared)
    output_width = len(_RESERVED_COLUMNS) + len(union)
    if total_rows * output_width > _MAX_OUTPUT_CELLS:
        raise ValueError("merged rows exceed the supported output cell limit")

    union_names = tuple(spec.name for spec in union)
    merged_rows = []
    for batch_id, batch, schema in zip(
        batch_ids,
        prepared,
        schemas,
        strict=True,
    ):
        positions = {spec.name: position for position, spec in enumerate(schema)}
        for row_number, row in enumerate(batch.rows, start=1):
            validated = tuple(
                _validate_cell(cell, spec=spec)
                for cell, spec in zip(row, schema, strict=True)
            )
            payload = tuple(
                validated[positions[name]] if name in positions else None
                for name in union_names
            )
            merged_rows.append((batch_id, row_number, *payload))

    columns = (
        ColumnSpec(_BATCH_ID_COLUMN, "str"),
        ColumnSpec(_ROW_NUMBER_COLUMN, "int"),
        *union,
    )
    return MergedRows(columns=columns, rows=tuple(merged_rows))
```

## Example

```python
first = RowBatch(
    batch_id="batch-a",
    columns=(
        ColumnSpec("record_id", "int"),
        ColumnSpec("amount", "float"),
    ),
    rows=((101, 2.5), (102, 4.0)),
)
second = RowBatch(
    batch_id="batch-b",
    columns=(
        ColumnSpec("state", "str"),
        ColumnSpec("record_id", "int"),
    ),
    rows=(("ready", 103),),
)

merged = merge_bounded_row_batches((first, second))

try:
    merge_bounded_row_batches(
        (
            first,
            RowBatch(
                "batch-c",
                (ColumnSpec("record_id", "str"),),
                (("104",),),
            ),
        )
    )
except ValueError:
    type_conflict_rejected = True
else:
    type_conflict_rejected = False


class ActiveScalar:
    equality_calls = 0

    def __eq__(self, other: object) -> bool:
        type(self).equality_calls += 1
        return False


ActiveScalar.equality_calls = 0
try:
    merge_bounded_row_batches(
        (
            RowBatch(
                "batch-active",
                (ColumnSpec("payload", "str"),),
                ((ActiveScalar(),),),
            ),
        )
    )
except TypeError:
    active_scalar_rejected = True
else:
    active_scalar_rejected = False

assert (
    merged,
    first.rows,
    second.rows,
    type_conflict_rejected,
    active_scalar_rejected,
    ActiveScalar.equality_calls,
) == (
    MergedRows(
        columns=(
            ColumnSpec("source_batch_id", "str"),
            ColumnSpec("source_row_number", "int"),
            ColumnSpec("record_id", "int"),
            ColumnSpec("amount", "float", nullable=True),
            ColumnSpec("state", "str", nullable=True),
        ),
        rows=(
            ("batch-a", 1, 101, 2.5, None),
            ("batch-a", 2, 102, 4.0, None),
            ("batch-b", 1, 103, None, "ready"),
        ),
    ),
    ((101, 2.5), (102, 4.0)),
    (("ready", 103),),
    True,
    True,
    0,
)
```

## Trade-offs and Limitations

Preflight checks the complete immutable container shape and input-cell budget
before validating schemas or scalar values. The function then stores the whole
union and output in memory, using time proportional to the input cells plus the
nullable padding cells. The output-cell cap includes both lineage columns, but
Python tuple, integer, and string overhead can still exceed a caller's tighter
memory budget.

The union is structural rather than semantic: the first occurrence chooses a
column's position, but no batch has precedence over another's values. Missing
columns become nullable even when the affected batch has no rows. Exact Python
types deliberately reject subclasses, decimals, datetimes, containers,
non-finite floats, and active objects. Errors abort without a partial return;
the algorithm does not deduplicate records, coerce types, update its inputs,
write files, persist results, or remove source data.

## Related Snippets

<!-- catalog:related:start -->
- [Validate Parsed CSV Rows with Bounded Structured Problems](validate-parsed-csv-rows-with-bounded-structured-problems.md)
- [Normalize Optional CSV Columns in a Single Pass](normalize-optional-csv-columns-in-a-single-pass.md)
- [Make a Defensive Copy at a Mutable Input Boundary](../python-language/make-a-defensive-copy-at-a-mutable-input-boundary.md)
<!-- catalog:related:end -->
