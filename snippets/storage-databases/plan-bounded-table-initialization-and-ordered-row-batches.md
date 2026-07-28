---
title: "Plan Bounded Table Initialization and Ordered Row Batches"
snippet_type: pattern
use_cases:
  - data-transformation
  - persistence
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - project-a-dataclass-into-a-validated-insert-row.md
  - build-and-apply-a-deterministic-mapping-delta.md
  - plan-a-bounded-named-posix-layout-under-explicit-roots.md
---

# Plan Bounded Table Initialization and Ordered Row Batches

## Idea and Problem

Turn an exact schema, an explicit observation of current state, and finite rows into a frozen creation-and-batch plan.

The planner accepts only a closed set of scalar kinds. It validates every column and cell,
proves all cardinality and logical-byte bounds, and compares a supplied present schema with
the desired schema before materializing any output. Planning is therefore deterministic and
does not depend on hidden state.

## When to Use

Use this pattern when a caller already has an authoritative, immutable schema description and
an exact tuple of rows, while a separate executor needs ordered batches with a fixed maximum
row count. An absent observation always produces a creation description, including when there
are no rows. A present observation suppresses that description only when its schema is exactly
equal to the desired schema.

This deliberately narrow contract suits initialization, not schema evolution. Any difference
in column order, name, kind, or nullability is rejected instead of being transformed.

## Implementation

```python
import math
import re
from dataclasses import dataclass
from enum import Enum

_MAX_COLUMNS = 64
_MAX_ROWS = 20_000
_MAX_CELLS = 400_000
_MAX_CELL_BYTES = 64 * 1_024
_MAX_LOGICAL_BYTES = 16 * 1_024 * 1_024
_MAX_INTEGER = (1 << 63) - 1
_COLUMN_NAME = re.compile(r"[a-z][a-z0-9_]{0,47}", re.ASCII)


class ScalarKind(Enum):
    BOOLEAN = "boolean"
    INTEGER = "integer"
    FLOAT = "float"
    TEXT = "text"
    BYTES = "bytes"


type Cell = None | bool | int | float | str | bytes
type FrozenRow = tuple[Cell, ...]


@dataclass(frozen=True, slots=True)
class FieldSpec:
    name: str
    kind: ScalarKind
    nullable: bool


@dataclass(frozen=True, slots=True)
class Absent:
    pass


@dataclass(frozen=True, slots=True)
class Present:
    schema: tuple[FieldSpec, ...]


type TableObservation = Absent | Present


@dataclass(frozen=True, slots=True)
class CreationPlan:
    schema: tuple[FieldSpec, ...]


@dataclass(frozen=True, slots=True)
class RowBatch:
    sequence: int
    rows: tuple[FrozenRow, ...]


@dataclass(frozen=True, slots=True)
class InitializationPlan:
    creation: CreationPlan | None
    batches: tuple[RowBatch, ...]


def _validate_schema(schema: object, field: str) -> None:
    if type(schema) is not tuple:
        raise TypeError(f"{field} must be an exact tuple")
    if not 1 <= len(schema) <= _MAX_COLUMNS:
        raise ValueError(f"{field} must contain 1..{_MAX_COLUMNS} fields")

    names: set[str] = set()
    for position, column in enumerate(schema):
        item = f"{field}[{position}]"
        if type(column) is not FieldSpec:
            raise TypeError(f"{item} must be an exact FieldSpec")
        if type(column.name) is not str:
            raise TypeError(f"{item}.name must be an exact string")
        if _COLUMN_NAME.fullmatch(column.name) is None:
            raise ValueError(f"{item}.name has an invalid shape")
        if column.name in names:
            raise ValueError(f"{field} field names must be unique")
        names.add(column.name)
        if type(column.kind) is not ScalarKind:
            raise TypeError(f"{item}.kind must be an exact ScalarKind")
        if type(column.nullable) is not bool:
            raise TypeError(f"{item}.nullable must be an exact bool")


def _cell_size(value: object, column: FieldSpec, field: str) -> int:
    if value is None:
        if not column.nullable:
            raise ValueError(f"{field} must not be null")
        return 1

    if column.kind is ScalarKind.BOOLEAN:
        if type(value) is not bool:
            raise TypeError(f"{field} must be an exact bool")
        size = 1
    elif column.kind is ScalarKind.INTEGER:
        if type(value) is not int:
            raise TypeError(f"{field} must be an exact int")
        if not -_MAX_INTEGER - 1 <= value <= _MAX_INTEGER:
            raise ValueError(f"{field} is outside the signed 64-bit range")
        size = 8
    elif column.kind is ScalarKind.FLOAT:
        if type(value) is not float:
            raise TypeError(f"{field} must be an exact float")
        if not math.isfinite(value):
            raise ValueError(f"{field} must be finite")
        size = 8
    elif column.kind is ScalarKind.TEXT:
        if type(value) is not str:
            raise TypeError(f"{field} must be an exact string")
        try:
            size = len(value.encode("utf-8"))
        except UnicodeEncodeError as error:
            raise ValueError(f"{field} must be valid UTF-8 text") from error
    elif column.kind is ScalarKind.BYTES:
        if type(value) is not bytes:
            raise TypeError(f"{field} must be exact bytes")
        size = len(value)
    else:
        raise AssertionError("unreachable scalar kind")

    if size > _MAX_CELL_BYTES:
        raise ValueError(f"{field} exceeds the per-cell logical-byte limit")
    return size


def _preflight_rows(
    rows: object,
    schema: tuple[FieldSpec, ...],
) -> None:
    if type(rows) is not tuple:
        raise TypeError("rows must be an exact tuple")
    if len(rows) > _MAX_ROWS:
        raise ValueError("rows exceeds the row-count limit")

    cell_count = len(rows) * len(schema)
    if cell_count > _MAX_CELLS:
        raise ValueError("rows exceeds the cell-count limit")

    logical_bytes = 0
    for row_position, row in enumerate(rows):
        if type(row) is not tuple:
            raise TypeError(f"rows[{row_position}] must be an exact tuple")
        if len(row) != len(schema):
            raise ValueError(f"rows[{row_position}] has the wrong shape")
        for column_position, (value, column) in enumerate(zip(row, schema, strict=True)):
            logical_bytes += _cell_size(
                value,
                column,
                f"rows[{row_position}][{column_position}]",
            )
            if logical_bytes > _MAX_LOGICAL_BYTES:
                raise ValueError("rows exceeds the aggregate logical-byte limit")


def plan_table_initialization(
    desired_schema: tuple[FieldSpec, ...],
    observation: TableObservation,
    rows: tuple[FrozenRow, ...],
    batch_limit: int,
) -> InitializationPlan:
    _validate_schema(desired_schema, "desired_schema")
    if type(batch_limit) is not int:
        raise TypeError("batch_limit must be an exact integer")
    if batch_limit <= 0:
        raise ValueError("batch_limit must be positive")

    if type(observation) is Absent:
        needs_creation = True
    elif type(observation) is Present:
        _validate_schema(observation.schema, "observation.schema")
        if observation.schema != desired_schema:
            raise ValueError("the present schema is not identical to desired_schema")
        needs_creation = False
    else:
        raise TypeError("observation must be an exact Absent or Present")

    _preflight_rows(rows, desired_schema)

    creation = CreationPlan(desired_schema) if needs_creation else None
    batches = tuple(
        RowBatch(sequence=start // batch_limit, rows=rows[start : start + batch_limit])
        for start in range(0, len(rows), batch_limit)
    )
    return InitializationPlan(creation=creation, batches=batches)
```

## Example

```python
schema = (
    FieldSpec("sample_key", ScalarKind.TEXT, False),
    FieldSpec("verified", ScalarKind.BOOLEAN, False),
    FieldSpec("reading_c", ScalarKind.FLOAT, True),
)
rows = (
    ("jetty-a", True, 18.75),
    ("jetty-b", False, None),
    ("jetty-c", True, 19.0),
    ("jetty-d", True, 17.625),
    ("jetty-e", False, 20.25),
)

plan = plan_table_initialization(schema, Absent(), rows, batch_limit=2)
empty_plan = plan_table_initialization(schema, Absent(), (), batch_limit=2)
present_plan = plan_table_initialization(schema, Present(schema), rows[:1], batch_limit=2)

assert plan.creation == CreationPlan(schema)
assert tuple(len(batch.rows) for batch in plan.batches) == (2, 2, 1)
assert tuple(row[0] for batch in plan.batches for row in batch.rows) == tuple(
    row[0] for row in rows
)
assert empty_plan == InitializationPlan(creation=CreationPlan(schema), batches=())
assert present_plan == InitializationPlan(
    creation=None,
    batches=(RowBatch(sequence=0, rows=(rows[0],)),),
)
```

## Trade-offs and Limitations

Preflight takes `O(columns + rows * columns)` time, and materialization takes `O(rows)`
additional references. The fixed limits and conservative field-name grammar intentionally reject
larger or more permissive inputs. Logical-byte accounting is a planning metric: text uses its
UTF-8 length, byte strings use their length, and fixed-width scalars use the sizes declared in
the implementation.

An observation is supplied evidence and can become stale. Immediately before applying a
creation description, the executor must revalidate that precondition against authoritative
current state and define how a competing creator is resolved. This planner only classifies
validated values; it neither reads authoritative state nor applies its output. An incompatible
present schema is rejected, with no conversion path.

## Related Snippets

<!-- catalog:related:start -->
- [Project a Dataclass into a Validated Insert Row](project-a-dataclass-into-a-validated-insert-row.md)
- [Build and Apply a Deterministic Mapping Delta](build-and-apply-a-deterministic-mapping-delta.md)
- [Plan a Bounded Named POSIX Layout Under Explicit Roots](plan-a-bounded-named-posix-layout-under-explicit-roots.md)
<!-- catalog:related:end -->
