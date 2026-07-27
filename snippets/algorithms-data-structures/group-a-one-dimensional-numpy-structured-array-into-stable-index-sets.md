---
title: "Group a One-Dimensional NumPy Structured Array into Stable Index Sets"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies:
  - name: numpy
    version: "2.5.1"
related:
  - ../data-processing/group-items-by-an-exact-compatibility-signature-and-report-unmatched-inputs.md
  - ../data-processing/select-one-record-per-key-with-an-explicit-ranking-rule.md
---

# Group a One-Dimensional NumPy Structured Array into Stable Index Sets

## Idea and Problem

Group a bounded one-dimensional NumPy structured array by scalar fields while returning row positions instead of copying records into separate arrays.

The first occurrence of each key fixes group order, and scanning rows from left
to right makes every group's index tuple ascending. Converting supported NumPy
scalars to exact Python `bool`, `int`, `bytes`, or `str` values produces
immutable, hashable key tuples without accepting ambiguous floating-point,
object, complex, nested, or subarray comparison semantics.

## When to Use

Use this algorithm when a structured array is already in memory, group identity
is exact, and downstream code needs stable positions for indexing, auditing, or
partitioned processing. One to eight existing fields may form the key. Non-key
fields can have other dtypes because the function neither reads nor copies
their values.

Use `numpy.unique`, pandas grouping, a database, or an external partitioner
when sorted-key order, aggregation, missing-value semantics, arbitrary Python
objects, or data larger than the fixed in-memory limits is required. Normalize
text and define any case-folding policy before this boundary rather than
changing equality inside the grouping operation.

## Implementation

```python
from dataclasses import dataclass

import numpy as np


StructuredKeyValue = bool | int | bytes | str
_MAX_ROWS = 100_000
_MAX_KEY_FIELDS = 8
_MAX_FIELD_NAME_CHARACTERS = 128
_MAX_KEY_FIELD_BYTES = 4_096
_MAX_GROUPS = 10_000
_SUPPORTED_KEY_KINDS = frozenset({"b", "i", "u", "S", "U"})


@dataclass(frozen=True, slots=True)
class StructuredIndexGroup:
    key: tuple[StructuredKeyValue, ...]
    indices: tuple[int, ...]


def _validate_key_fields(
    array: np.ndarray,
    key_fields: tuple[str, ...],
) -> tuple[np.ndarray, ...]:
    if type(key_fields) is not tuple:
        raise TypeError("key_fields must be an exact tuple")
    if not 1 <= len(key_fields) <= _MAX_KEY_FIELDS:
        raise ValueError("key field count is outside the supported range")

    declared_names = array.dtype.names
    if declared_names is None:
        raise TypeError("array must have a structured dtype")

    seen: set[str] = set()
    views: list[np.ndarray] = []
    for name in key_fields:
        if type(name) is not str:
            raise TypeError("every key field name must be an exact string")
        if not name or len(name) > _MAX_FIELD_NAME_CHARACTERS:
            raise ValueError("a key field name is outside the supported range")
        if name in seen:
            raise ValueError("key field names must be unique")
        if name not in declared_names:
            raise ValueError(f"unknown key field: {name!r}")

        field_dtype = array.dtype.fields[name][0]
        if (
            field_dtype.subdtype is not None
            or field_dtype.kind not in _SUPPORTED_KEY_KINDS
        ):
            raise TypeError(
                "key fields must be scalar bool, integer, bytes, or Unicode values"
            )
        if not 1 <= field_dtype.itemsize <= _MAX_KEY_FIELD_BYTES:
            raise ValueError("key field width is outside the supported range")

        seen.add(name)
        views.append(array[name])
    return tuple(views)


def _python_key_value(value: np.generic) -> StructuredKeyValue:
    converted = value.item()
    if type(converted) not in (bool, int, bytes, str):
        raise TypeError("a key scalar could not be converted to a supported value")
    return converted


def group_structured_indices(
    array: np.ndarray,
    *,
    key_fields: tuple[str, ...],
) -> tuple[StructuredIndexGroup, ...]:
    if type(array) is not np.ndarray:
        raise TypeError("array must be an exact NumPy ndarray")
    if array.ndim != 1:
        raise ValueError("array must be one-dimensional")
    if len(array) > _MAX_ROWS:
        raise ValueError("row count exceeds the supported limit")

    field_views = _validate_key_fields(array, key_fields)
    positions_by_key: dict[tuple[StructuredKeyValue, ...], list[int]] = {}
    ordered_keys: list[tuple[StructuredKeyValue, ...]] = []

    for row_index in range(len(array)):
        key = tuple(
            _python_key_value(field[row_index])
            for field in field_views
        )
        positions = positions_by_key.get(key)
        if positions is None:
            if len(ordered_keys) == _MAX_GROUPS:
                raise ValueError("group count exceeds the supported limit")
            positions = []
            positions_by_key[key] = positions
            ordered_keys.append(key)
        positions.append(row_index)

    return tuple(
        StructuredIndexGroup(key, tuple(positions_by_key[key]))
        for key in ordered_keys
    )
```

## Example

```python
records = np.array(
    [
        ("east", 2, True, 8.5),
        ("west", 1, False, 3.0),
        ("east", 2, True, 9.25),
        ("east", 3, False, 4.5),
    ],
    dtype=[
        ("region", "U8"),
        ("tier", "i2"),
        ("enabled", "?"),
        ("measurement", "f8"),
    ],
)
before = records.copy()
groups = group_structured_indices(
    records,
    key_fields=("region", "tier", "enabled"),
)

try:
    group_structured_indices(
        np.array([("east",)], dtype=[("region", object)]),
        key_fields=("region",),
    )
except TypeError:
    object_key_rejected = True
else:
    object_key_rejected = False

assert (
    groups,
    np.array_equal(records, before),
    object_key_rejected,
) == (
    (
        StructuredIndexGroup(("east", 2, True), (0, 2)),
        StructuredIndexGroup(("west", 1, False), (1,)),
        StructuredIndexGroup(("east", 3, False), (3,)),
    ),
    True,
    True,
)
```

## Trade-offs and Limitations

The function performs `O(r * k)` scalar reads for `r` rows and `k` key fields.
It stores one Python integer per row plus one dictionary entry and immutable
key per group. Field selection creates views, but no group array or record copy
is materialized. Converting fixed-width byte and Unicode scalars with `.item()`
uses NumPy's logical scalar value; preserve raw fixed-width storage separately
if padding bytes themselves carry meaning.

The bounds cap rows, groups, fields, field-name lengths, and selected field
widths. Exact `ndarray` validation intentionally rejects subclasses and masked
arrays. The input is only read and remains untouched, but concurrent mutation
by another thread would make the scan inconsistent. This function does not
aggregate values, represent missing keys, sort groups, or promise portable
byte order for later serialization.

## Related Snippets

<!-- catalog:related:start -->
- [Group Items by an Exact Compatibility Signature and Report Unmatched Inputs](../data-processing/group-items-by-an-exact-compatibility-signature-and-report-unmatched-inputs.md)
- [Select One Record per Key with an Explicit Ranking Rule](../data-processing/select-one-record-per-key-with-an-explicit-ranking-rule.md)
<!-- catalog:related:end -->
