---
title: "Plan an Additive SQLite Column Projection"
snippet_type: recipe
use_cases:
  - data-transformation
  - persistence
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - project-a-dataclass-into-a-validated-insert-row.md
  - compare-and-swap-a-versioned-sqlite-setting-with-one-conditional-update.md
  - ../configuration-serialization/migrate-one-bounded-json-record-to-a-current-version.md
---

# Plan an Additive SQLite Column Projection

## Idea and Problem

Build a parameterized SQLite row projection from one frozen column schema into a strictly additive successor schema.

An additive schema change sometimes needs an explicit row projection before a
caller writes data into a replacement table. Existing columns should be read by
name, while every new column receives a reviewed scalar default. Validating the
complete old and new shapes before rendering prevents a removal, reorder, type
change, or unsafe literal from being disguised as an additive operation.

## When to Use

Use this planner when one migration owner already has two small ordered schema
snapshots and supports only SQLite `INTEGER`, `REAL`, `TEXT`, and `BLOB` storage
classes. It fits an out-of-place migration whose transaction, destination
table, constraints, and verification are handled separately. It is useful when
defaults must be bound as values rather than embedded in SQL text.

Do not use it for renames, type conversions, generated columns, nullable
defaults, constraints, or arbitrary SQL expressions. Those changes require a
larger dialect-aware migration design.

## Implementation

```python
import math
import re
from dataclasses import dataclass


_MAX_COLUMNS = 64
_MAX_TEXT_BYTES = 512
_MAX_BLOB_BYTES = 4_096
_MAX_INTEGER = (1 << 63) - 1
_IDENTIFIER = re.compile(r"[a-z][a-z0-9_]{0,62}", re.ASCII)
_SQLITE_TYPES = frozenset({"INTEGER", "REAL", "TEXT", "BLOB"})
_MISSING = object()


@dataclass(frozen=True, slots=True)
class SqliteColumn:
    name: str
    storage_type: str
    default: object = _MISSING


@dataclass(frozen=True, slots=True)
class AdditiveProjection:
    select_sql: str
    parameters: tuple[object, ...]
    added_columns: tuple[str, ...]


def _identifier(value: object, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if _IDENTIFIER.fullmatch(value) is None:
        raise ValueError(f"{field} is not a conservative SQL identifier")
    return value


def _columns(value: object, field: str) -> tuple[SqliteColumn, ...]:
    if type(value) is not tuple:
        raise TypeError(f"{field} must be an exact tuple")
    if not 1 <= len(value) <= _MAX_COLUMNS:
        raise ValueError(f"{field} must contain 1..{_MAX_COLUMNS} columns")

    seen: set[str] = set()
    for column in value:
        if type(column) is not SqliteColumn:
            raise TypeError(f"{field} must contain exact SqliteColumn records")
        name = _identifier(column.name, f"{field} column name")
        if name in seen:
            raise ValueError(f"{field} contains a duplicate column")
        seen.add(name)
        if type(column.storage_type) is not str:
            raise TypeError("storage_type must be an exact string")
        if column.storage_type not in _SQLITE_TYPES:
            raise ValueError("unsupported SQLite storage type")
    return value


def _default_value(column: SqliteColumn) -> object:
    value = column.default
    match column.storage_type:
        case "INTEGER" if type(value) is int:
            if not -_MAX_INTEGER - 1 <= value <= _MAX_INTEGER:
                raise ValueError("INTEGER default is outside the signed 64-bit range")
        case "REAL" if type(value) is int:
            if not -_MAX_INTEGER - 1 <= value <= _MAX_INTEGER:
                raise ValueError("integral REAL default is outside the supported range")
        case "REAL" if type(value) is float:
            if not math.isfinite(value):
                raise ValueError("REAL default must be finite")
        case "TEXT" if type(value) is str:
            if len(value.encode("utf-8")) > _MAX_TEXT_BYTES or "\x00" in value:
                raise ValueError("TEXT default is too large or contains NUL")
        case "BLOB" if type(value) is bytes:
            if len(value) > _MAX_BLOB_BYTES:
                raise ValueError("BLOB default is too large")
        case _:
            raise TypeError("new-column default does not match its storage type")
    return value


def plan_additive_sqlite_projection(
    table_name: str,
    current: tuple[SqliteColumn, ...],
    desired: tuple[SqliteColumn, ...],
) -> AdditiveProjection:
    table = _identifier(table_name, "table_name")
    old_columns = _columns(current, "current")
    new_columns = _columns(desired, "desired")
    old_by_name = {column.name: column for column in old_columns}

    if any(column.default is not _MISSING for column in old_columns):
        raise ValueError("current columns must not carry projection defaults")

    old_index = 0
    expressions: list[str] = []
    parameters: list[object] = []
    added: list[str] = []

    for column in new_columns:
        quoted_name = f'"{column.name}"'
        if old_index < len(old_columns) and column.name == old_columns[old_index].name:
            previous = old_columns[old_index]
            if column.storage_type != previous.storage_type:
                raise ValueError("existing column type changed")
            if column.default is not _MISSING:
                raise ValueError("existing columns must not carry projection defaults")
            expressions.append(quoted_name)
            old_index += 1
        elif column.name in old_by_name:
            raise ValueError("existing columns were reordered")
        else:
            if column.default is _MISSING:
                raise ValueError("new columns require explicit defaults")
            parameters.append(_default_value(column))
            expressions.append(
                f'CAST(? AS {column.storage_type}) AS {quoted_name}'
            )
            added.append(column.name)

    if old_index != len(old_columns):
        raise ValueError("existing columns were removed")
    if not added:
        raise ValueError("schemas do not contain an additive change")

    return AdditiveProjection(
        select_sql=f'SELECT {", ".join(expressions)} FROM "{table}"',
        parameters=tuple(parameters),
        added_columns=tuple(added),
    )
```

## Example

```python
current = (
    SqliteColumn("record_id", "INTEGER"),
    SqliteColumn("payload", "BLOB"),
)
desired = (
    SqliteColumn("record_id", "INTEGER"),
    SqliteColumn("label", "TEXT", "unassigned"),
    SqliteColumn("payload", "BLOB"),
    SqliteColumn("weight", "REAL", 1.5),
)

plan = plan_additive_sqlite_projection("records", current, desired)

assert plan == AdditiveProjection(
    select_sql=(
        'SELECT "record_id", CAST(? AS TEXT) AS "label", "payload", '
        'CAST(? AS REAL) AS "weight" FROM "records"'
    ),
    parameters=("unassigned", 1.5),
    added_columns=("label", "weight"),
)
```

## Trade-offs and Limitations

Validation and rendering take `O(n)` time and memory for at most 64 columns.
The conservative identifier grammar is narrower than SQLite's full quoted-name
grammar, and the four-type/default policy deliberately excludes many valid
SQLite expressions. Placeholders bind default values; they cannot bind table or
column identifiers, so those identifiers are both validated and quoted.

The result is only a row-shape projection. It does not inspect a database,
create or replace a table, preserve indexes or constraints, execute SQL,
backfill rows, open a transaction, or prove that a complete migration is safe.
The caller must perform those operations atomically where appropriate and
verify the destination before switching readers or deleting old data.

## Related Snippets

<!-- catalog:related:start -->
- [Project a Dataclass into a Validated Insert Row](project-a-dataclass-into-a-validated-insert-row.md)
- [Compare and Swap a Versioned SQLite Setting with One Conditional Update](compare-and-swap-a-versioned-sqlite-setting-with-one-conditional-update.md)
- [Migrate One Bounded JSON Record to a Current Version](../configuration-serialization/migrate-one-bounded-json-record-to-a-current-version.md)
<!-- catalog:related:end -->
