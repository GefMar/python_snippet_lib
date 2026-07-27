---
title: "Project Bounded Records into Multiple Closed Output Schemas"
snippet_type: recipe
use_cases:
  - data-transformation
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - project-nested-records-with-explicit-field-paths.md
  - merge-bounded-row-batches-with-a-first-seen-schema-union.md
  - ../python-language/make-a-defensive-copy-at-a-mutable-input-boundary.md
---

# Project Bounded Records into Multiple Closed Output Schemas

## Idea and Problem

Validate a bounded record batch once, then project every record into several ordered schemas without exposing a partial result.

Each target declares a closed list of required fields. The function checks all
records and the complete output-cell budget before it starts copying values, so
one missing field rejects the call rather than leaving downstream consumers
with differently shaped rows.

## When to Use

Use this recipe when one finite in-memory batch must feed several consumers
that need different subsets of the same flat records. It fits trusted values
whose field names and required shape are already known. Normalize and validate
value types before this boundary, and use streaming or transactional sink logic
when the materialized output would be too large.

## Implementation

```python
import copy
import re
from collections.abc import Mapping
from dataclasses import dataclass


_NAME = re.compile(r"[a-z][a-z0-9_.-]{0,63}").fullmatch
_MAX_RECORDS = 128
_MAX_TARGETS = 8
_MAX_FIELDS = 32
_MAX_OUTPUT_CELLS = 4_096


@dataclass(frozen=True, slots=True)
class OutputSchema:
    name: str
    fields: tuple[str, ...]


class MissingOutputField(ValueError):
    pass


def _validate_name(value: object, *, kind: str) -> None:
    if type(value) is not str or _NAME(value) is None:
        raise ValueError(f"{kind} must use the supported syntax")


def project_records(
    records: tuple[Mapping[str, object], ...],
    schemas: tuple[OutputSchema, ...],
) -> dict[str, tuple[dict[str, object], ...]]:
    if type(records) is not tuple:
        raise TypeError("records must be a tuple")
    if not 1 <= len(records) <= _MAX_RECORDS:
        raise ValueError("record count is outside the supported range")
    if type(schemas) is not tuple:
        raise TypeError("schemas must be a tuple")
    if not 1 <= len(schemas) <= _MAX_TARGETS:
        raise ValueError("schema count is outside the supported range")

    seen_schema_names: set[str] = set()
    field_total = 0
    for schema in schemas:
        if not isinstance(schema, OutputSchema):
            raise TypeError("schemas must contain OutputSchema values")
        _validate_name(schema.name, kind="schema name")
        if schema.name in seen_schema_names:
            raise ValueError("schema names must be unique")
        seen_schema_names.add(schema.name)
        if type(schema.fields) is not tuple:
            raise TypeError("schema fields must be a tuple")
        if not 1 <= len(schema.fields) <= _MAX_FIELDS:
            raise ValueError("field count is outside the supported range")
        for field in schema.fields:
            _validate_name(field, kind="field name")
        if len(set(schema.fields)) != len(schema.fields):
            raise ValueError("schema fields must be unique")
        field_total += len(schema.fields)

    if len(records) * field_total > _MAX_OUTPUT_CELLS:
        raise ValueError("output cell count exceeds the supported limit")

    for record_index, record in enumerate(records):
        if not isinstance(record, Mapping):
            raise TypeError(f"record {record_index} must be a mapping")
        if any(type(key) is not str for key in record):
            raise TypeError(f"record {record_index} keys must be text")
        for schema in schemas:
            for field in schema.fields:
                if field not in record:
                    raise MissingOutputField(
                        f"record {record_index} is missing a required field"
                    )

    return {
        schema.name: tuple(
            {
                field: copy.deepcopy(record[field])
                for field in schema.fields
            }
            for record in records
        )
        for schema in schemas
    }
```

## Example

```python
records = (
    {"item": "alpha", "score": 3, "labels": ["new"], "extra": True},
    {"item": "beta", "score": None, "labels": []},
)
schemas = (
    OutputSchema("summary", ("item", "score")),
    OutputSchema("labels", ("item", "labels")),
)

outputs = project_records(records, schemas)
records[0]["labels"].append("changed")

assert outputs == {
    "summary": (
        {"item": "alpha", "score": 3},
        {"item": "beta", "score": None},
    ),
    "labels": (
        {"item": "alpha", "labels": ["new"]},
        {"item": "beta", "labels": []},
    ),
}
```

## Trade-offs and Limitations

The result is fully materialized and deep-copies every selected cell, including
a value selected by more than one target. That prevents aliasing but can be
expensive, and `copy.deepcopy` may invoke methods on user-defined objects; use
this helper only with trusted in-memory values or narrow the accepted value
model. The cell limit does not bound byte size. Extra input fields are silently
ignored, value types are not checked, and concurrent mutation of custom
mappings is unsupported. The function performs no normalization, enrichment,
streaming, sink writes, or transaction management.

## Related Snippets

<!-- catalog:related:start -->
- [Project Nested Records with Explicit Field Paths](project-nested-records-with-explicit-field-paths.md)
- [Merge Bounded Row Batches with a First-Seen Schema Union](merge-bounded-row-batches-with-a-first-seen-schema-union.md)
- [Make a Defensive Copy at a Mutable Input Boundary](../python-language/make-a-defensive-copy-at-a-mutable-input-boundary.md)
<!-- catalog:related:end -->
