---
title: "Project Nested Records with Explicit Field Paths"
snippet_type: recipe
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../configuration-serialization/get-nested-values-with-a-validated-dot-path.md
  - normalize-optional-csv-columns-in-a-single-pass.md
  - ../python-language/apply-partial-dataclass-updates-with-an-omitted-value-sentinel.md
---

# Project Nested Records with Explicit Field Paths

## Idea and Problem

Project heterogeneous nested mappings through one declared field list while auditing required paths that are absent.

Each field uses a tuple of text keys rather than an implicit path grammar. A
private sentinel distinguishes a missing path from a present value of `None`;
missing columns are omitted from that output row, and required omissions are
reported with stable record indexes.

## When to Use

Use this recipe before tabular conversion when input records have similar but
not identical nested mapping shapes and the desired columns are known. It fits
bounded JSON-like batches where lists should remain leaf values. Use a
schema validator when value types and cross-field rules must be enforced, or a
dedicated normalization step when a path traverses arrays of objects.

## Implementation

```python
from collections.abc import Iterable, Mapping
from dataclasses import dataclass


_MISSING = object()


class DuplicateOutputName(ValueError):
    pass


@dataclass(frozen=True, slots=True)
class ProjectionField:
    output_name: str
    path: tuple[str, ...]
    required: bool = False


@dataclass(frozen=True, slots=True)
class MissingRequiredField:
    record_index: int
    output_name: str
    path: tuple[str, ...]


@dataclass(frozen=True, slots=True)
class ProjectionResult:
    rows: tuple[dict[str, object], ...]
    missing_required: tuple[MissingRequiredField, ...]


def _validated_fields(
    fields: Iterable[ProjectionField],
) -> tuple[ProjectionField, ...]:
    provided = tuple(fields)
    if not provided:
        raise ValueError("at least one projection field is required")
    seen_names: set[str] = set()
    for field in provided:
        if not isinstance(field, ProjectionField):
            raise TypeError("fields must contain ProjectionField values")
        if not isinstance(field.output_name, str) or not field.output_name:
            raise ValueError("output_name must be non-empty text")
        if field.output_name in seen_names:
            raise DuplicateOutputName(field.output_name)
        seen_names.add(field.output_name)
        if (
            not isinstance(field.path, tuple)
            or not field.path
            or any(not isinstance(part, str) or not part for part in field.path)
        ):
            raise ValueError("each path must be a non-empty tuple of text keys")
        if not isinstance(field.required, bool):
            raise TypeError("required must be a boolean")
    return provided


def _read_mapping_path(
    record: Mapping[str, object],
    path: tuple[str, ...],
) -> object:
    current: object = record
    for part in path:
        if not isinstance(current, Mapping) or part not in current:
            return _MISSING
        current = current[part]
    return current


def project_nested_records(
    records: Iterable[Mapping[str, object]],
    fields: Iterable[ProjectionField],
) -> ProjectionResult:
    selected_fields = _validated_fields(fields)
    rows: list[dict[str, object]] = []
    missing_required: list[MissingRequiredField] = []

    for record_index, record in enumerate(records):
        if not isinstance(record, Mapping):
            raise TypeError(f"record {record_index} must be a mapping")
        row: dict[str, object] = {}
        for field in selected_fields:
            value = _read_mapping_path(record, field.path)
            if value is _MISSING:
                if field.required:
                    missing_required.append(
                        MissingRequiredField(
                            record_index=record_index,
                            output_name=field.output_name,
                            path=field.path,
                        ),
                    )
                continue
            row[field.output_name] = value
        rows.append(row)

    return ProjectionResult(
        rows=tuple(rows),
        missing_required=tuple(missing_required),
    )
```

## Example

```python
records = [
    {
        "record_id": "a",
        "profile": {"display_name": "Ada", "nickname": None},
        "labels": ["new"],
    },
    {"record_id": "b", "profile": {}, "labels": []},
    {"record_id": "c", "profile": "unavailable"},
]
fields = [
    ProjectionField("id", ("record_id",), required=True),
    ProjectionField("name", ("profile", "display_name"), required=True),
    ProjectionField("nickname", ("profile", "nickname")),
    ProjectionField("labels", ("labels",), required=True),
]

result = project_nested_records(records, fields)

try:
    project_nested_records(
        [],
        [ProjectionField("id", ("a",)), ProjectionField("id", ("b",))],
    )
except DuplicateOutputName:
    duplicate_rejected = True
else:
    duplicate_rejected = False

assert (
    result.rows,
    result.missing_required,
    result.rows[0]["nickname"] is None,
    "nickname" in result.rows[1],
    records[0]["profile"]["display_name"],
    duplicate_rejected,
) == (
    (
        {"id": "a", "name": "Ada", "nickname": None, "labels": ["new"]},
        {"id": "b", "labels": []},
        {"id": "c"},
    ),
    (
        MissingRequiredField(1, "name", ("profile", "display_name")),
        MissingRequiredField(2, "name", ("profile", "display_name")),
        MissingRequiredField(2, "labels", ("labels",)),
    ),
    True,
    False,
    "Ada",
    True,
)
```

## Trade-offs and Limitations

The result is materialized and rows can have different keys because missing
columns are omitted. Required omissions are audited rather than raised; the
caller decides whether the batch may proceed. A list, scalar, or `None`
encountered before the end of a path counts as missing, while a list at the end
is retained as one leaf. Mapping containers are read without mutation, but
selected leaves are shared by reference and may remain mutable. The recipe
does not validate leaf types, parse string paths, traverse arrays, flatten
arbitrary structures, preserve mapping subclasses, or detect changes to a
custom mapping during iteration.

## Related Snippets

<!-- catalog:related:start -->
- [Get Nested Values with a Validated Dot Path](../configuration-serialization/get-nested-values-with-a-validated-dot-path.md)
- [Normalize Optional CSV Columns in a Single Pass](normalize-optional-csv-columns-in-a-single-pass.md)
- [Apply Partial Dataclass Updates with an Omitted-Value Sentinel](../python-language/apply-partial-dataclass-updates-with-an-omitted-value-sentinel.md)
<!-- catalog:related:end -->
