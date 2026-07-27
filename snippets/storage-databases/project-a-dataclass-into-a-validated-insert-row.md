---
title: "Project a Dataclass into a Validated Insert Row"
snippet_type: recipe
use_cases:
  - data-transformation
  - persistence
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../python-language/apply-partial-dataclass-updates-with-an-omitted-value-sentinel.md
  - ../data-processing/map-exact-pandas-dtypes-to-a-neutral-storage-schema.md
  - build-and-apply-a-deterministic-mapping-delta.md
---

# Project a Dataclass into a Validated Insert Row

## Idea and Problem

Project only explicitly marked dataclass fields into a frozen, declaration-ordered insert row after validating the complete metadata contract.

Dataclass metadata can describe a narrow persistence boundary without turning
every field into an implicit column. A closed directive names the destination
column and states whether `None` is allowed; empty metadata omits a field, and
every other metadata shape fails before any instance value is read.

## When to Use

Use this recipe when a dataclass already owns an in-memory record and a database
adapter needs a small ordered column/value tuple. Treat the result as parameters
for a separately reviewed statement builder or driver call. Keep table names,
SQL syntax, generated values, transactions, and constraint handling outside
this projection boundary.

The contract intentionally accepts only empty field metadata or one exact
`insert_column` directive. Choose another mapper when unrelated metadata must
coexist on the same fields or when nested objects need conversion.

## Implementation

```python
import re
from dataclasses import dataclass, fields, is_dataclass


_MAX_FIELDS = 64
_METADATA_KEY = "insert_column"
_COLUMN = re.compile(r"[A-Za-z_][A-Za-z0-9_]{0,63}", re.ASCII)


@dataclass(frozen=True, slots=True)
class InsertColumn:
    column: str
    allow_none: bool


@dataclass(frozen=True, slots=True)
class InsertRow:
    columns: tuple[str, ...]
    values: tuple[object, ...]


def project_insert_row(instance: object) -> InsertRow:
    """Validate a closed field-metadata contract, then read mapped values."""
    if isinstance(instance, type) or not is_dataclass(instance):
        raise TypeError("instance must be a dataclass instance")

    declared_fields = fields(instance)
    if not 1 <= len(declared_fields) <= _MAX_FIELDS:
        raise ValueError(
            f"the dataclass must declare between 1 and {_MAX_FIELDS} fields"
        )

    metadata_snapshot = tuple(dict(item.metadata) for item in declared_fields)
    projection: list[tuple[str, InsertColumn]] = []
    destination_columns: set[str] = set()

    for index, (item, metadata) in enumerate(
        zip(declared_fields, metadata_snapshot, strict=True)
    ):
        if not metadata:
            continue
        if set(metadata) != {_METADATA_KEY}:
            raise ValueError(
                f"field {index} metadata must be empty or contain only "
                f"{_METADATA_KEY!r}"
            )

        directive = metadata[_METADATA_KEY]
        if type(directive) is not InsertColumn:
            raise TypeError(
                f"field {index} {_METADATA_KEY!r} must be an InsertColumn"
            )
        if type(directive.column) is not str:
            raise TypeError(f"field {index} destination column must be text")
        if _COLUMN.fullmatch(directive.column) is None:
            raise ValueError(
                f"field {index} destination column must be a conservative ASCII identifier"
            )
        if type(directive.allow_none) is not bool:
            raise TypeError(f"field {index} allow_none must be an exact boolean")
        if directive.column in destination_columns:
            raise ValueError("destination columns must be unique")

        destination_columns.add(directive.column)
        projection.append((item.name, directive))

    if not projection:
        raise ValueError("at least one dataclass field must be mapped")

    columns: list[str] = []
    values: list[object] = []
    for attribute_name, directive in projection:
        value = getattr(instance, attribute_name)
        if value is None and not directive.allow_none:
            raise ValueError(
                f"None is not allowed for destination column {directive.column!r}"
            )
        columns.append(directive.column)
        values.append(value)

    return InsertRow(tuple(columns), tuple(values))
```

## Example

```python
from dataclasses import dataclass, field


@dataclass(slots=True)
class SampleRecord:
    sample_key: str = field(
        metadata={"insert_column": InsertColumn("sample_key", False)}
    )
    magnitude: float = field(
        metadata={"insert_column": InsertColumn("measured_value", False)}
    )
    annotation: str | None = field(
        default=None,
        metadata={"insert_column": InsertColumn("annotation", True)},
    )
    transient_label: str = "not-persisted"


record = SampleRecord("sample-7", 12.5)
before = (
    record.sample_key,
    record.magnitude,
    record.annotation,
    record.transient_label,
)
row = project_insert_row(record)


@dataclass
class InvalidRecord:
    value: None = field(
        default=None,
        metadata={"insert_column": InsertColumn("value", False)},
    )


rejected_none = False
try:
    project_insert_row(InvalidRecord())
except ValueError:
    rejected_none = True

assert (
    row,
    (
        record.sample_key,
        record.magnitude,
        record.annotation,
        record.transient_label,
    ),
    rejected_none,
) == (
    InsertRow(
        columns=("sample_key", "measured_value", "annotation"),
        values=("sample-7", 12.5, None),
    ),
    before,
    True,
)
```

## Trade-offs and Limitations

Metadata validation is linear in at most 64 declared fields, and projection is
linear in the mapped subset. The metadata snapshot and projection plan are
built before the first `getattr`, so malformed later metadata cannot leave a
partially read value sequence. Attribute access can still execute a custom
descriptor and raise; this function performs no rollback because it owns no
external effect.

`InsertRow` and its tuples are frozen, but retained field values are ordinary
references. A mutable value can still change after projection, and the recipe
does not recursively copy, serialize, or validate driver-specific parameter
types. Apply an ownership policy appropriate to the eventual adapter.

The function does not generate SQL, quote identifiers, select a dialect, open a
connection, inspect an ORM schema, start a transaction, generate identifiers or
timestamps, translate constraints, or execute an insert. The conservative
column grammar is validation for this local contract, not a claim of SQL safety.

## Related Snippets

<!-- catalog:related:start -->
- [Apply Partial Dataclass Updates with an Omitted-Value Sentinel](../python-language/apply-partial-dataclass-updates-with-an-omitted-value-sentinel.md)
- [Map Exact pandas Dtypes to a Neutral Storage Schema](../data-processing/map-exact-pandas-dtypes-to-a-neutral-storage-schema.md)
- [Build and Apply a Deterministic Mapping Delta](build-and-apply-a-deterministic-mapping-delta.md)
<!-- catalog:related:end -->
