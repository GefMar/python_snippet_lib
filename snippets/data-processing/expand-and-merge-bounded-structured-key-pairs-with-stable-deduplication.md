---
title: "Expand and Merge Bounded Structured Key Pairs with Stable Deduplication"
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
  - ../algorithms-data-structures/group-a-one-dimensional-numpy-structured-array-into-stable-index-sets.md
  - merge-bounded-row-batches-with-a-first-seen-schema-union.md
  - group-items-by-an-exact-compatibility-signature-and-report-unmatched-inputs.md
---

# Expand and Merge Bounded Structured Key Pairs with Stable Deduplication

## Idea and Problem

Expand first-seen unique structured left keys against ordered structured right records, then merge those generated pairs after optional existing rows with stable deduplication.

Every complete pair is an exact tuple of supported scalar fields. Existing
rows are considered first, generated rows use left-major Cartesian order, and
the first occurrence of an equal complete pair wins. Dtype, shape, count,
Cartesian-product, and byte checks happen before the result array is allocated.

## When to Use

Use this algorithm for bounded, already materialized NumPy structured arrays
whose equality semantics are exact. The caller supplies the exact ordered field
names for each side. Left and right schemas must be disjoint, packed structured
dtypes made only of scalar Boolean, signed or unsigned integer, fixed-width
bytes, or fixed-width Unicode fields. Existing rows, when supplied, must match
the exact combined dtype. There are no implicit or reserved fields: the two
declared name tuples exhaust the combined output schema.

An empty left or right array generates no pairs, but stable unique existing
rows are still returned. With no existing rows, either empty side produces a
newly owned read-only empty array with the combined dtype. This operation does
not rank, recommend, invoke a model, fetch records, or persist its result.

## Implementation

```python
import re

import numpy as np

type StructuredValue = bool | int | bytes | str
type StructuredRow = tuple[StructuredValue, ...]

_FIELD_NAME = re.compile(r"[a-z][a-z0-9_]{0,31}\Z", re.ASCII).fullmatch
_SUPPORTED_FIELD_KINDS = frozenset({"b", "i", "u", "S", "U"})
_MAX_FIELDS_PER_SIDE = 8
_MAX_FIELD_BYTES = 1_024
_MAX_RECORD_BYTES = 4_096
_MAX_LEFT_ROWS = 50_000
_MAX_RIGHT_ROWS = 50_000
_MAX_EXISTING_ROWS = 200_000
_MAX_CARTESIAN_ROWS = 250_000
_MAX_PROSPECTIVE_ROWS = 400_000
_MAX_INPUT_BYTES = 64 * 1_024 * 1_024
_MAX_PROSPECTIVE_BYTES = 64 * 1_024 * 1_024


def _validated_field_names(value: object, *, role: str) -> tuple[str, ...]:
    if type(value) is not tuple:
        raise TypeError(f"{role} field names must be an exact tuple")
    if not 1 <= len(value) <= _MAX_FIELDS_PER_SIDE:
        raise ValueError(f"{role} field count is outside the supported range")

    names: list[str] = []
    seen: set[str] = set()
    for name in value:
        if type(name) is not str or _FIELD_NAME(name) is None:
            raise ValueError(f"{role} fields must use conservative ASCII names")
        if name in seen:
            raise ValueError(f"{role} field names must be unique")
        seen.add(name)
        names.append(name)
    return tuple(names)


def _validate_array(
    array: object,
    *,
    expected_names: tuple[str, ...],
    role: str,
    maximum_rows: int,
) -> np.ndarray:
    if type(array) is not np.ndarray:
        raise TypeError(f"{role} must be an exact NumPy ndarray")
    if array.ndim != 1:
        raise ValueError(f"{role} must be one-dimensional")
    if len(array) > maximum_rows:
        raise ValueError(f"{role} exceeds the supported row limit")
    if array.dtype.names != expected_names:
        raise TypeError(f"{role} must have the exact declared field names and order")
    if array.dtype.metadata is not None or array.dtype.isalignedstruct:
        raise TypeError(f"{role} must use a plain packed structured dtype")

    next_offset = 0
    for name in expected_names:
        field_info = array.dtype.fields[name]
        if len(field_info) != 2:
            raise TypeError(f"{role} field titles are not supported")
        field_dtype, offset = field_info
        if offset != next_offset:
            raise TypeError(f"{role} fields must not overlap or contain padding")
        if (
            field_dtype.names is not None
            or field_dtype.subdtype is not None
            or field_dtype.metadata is not None
            or field_dtype.kind not in _SUPPORTED_FIELD_KINDS
        ):
            raise TypeError(f"{role} fields must be scalar bool, integer, bytes, or Unicode values")
        if not 1 <= field_dtype.itemsize <= _MAX_FIELD_BYTES:
            raise ValueError(f"{role} contains a field outside the byte-width limit")
        next_offset += field_dtype.itemsize

    if array.dtype.itemsize != next_offset:
        raise TypeError(f"{role} records must not contain trailing padding")
    if array.dtype.itemsize > _MAX_RECORD_BYTES:
        raise ValueError(f"{role} record width exceeds the supported limit")
    return array


def _python_value(value: np.generic) -> StructuredValue:
    converted = value.item()
    if type(converted) not in (bool, int, bytes, str):
        raise TypeError("a structured scalar could not be converted exactly")
    return converted


def _row_key(
    array: np.ndarray,
    names: tuple[str, ...],
    row_index: int,
) -> StructuredRow:
    return tuple(_python_value(array[name][row_index]) for name in names)


def expand_and_merge_structured_pairs(
    left_keys: np.ndarray,
    right_records: np.ndarray,
    *,
    left_field_names: tuple[str, ...],
    right_field_names: tuple[str, ...],
    existing_pairs: np.ndarray | None = None,
) -> np.ndarray:
    left_names = _validated_field_names(left_field_names, role="left")
    right_names = _validated_field_names(right_field_names, role="right")
    if set(left_names) & set(right_names):
        raise ValueError("left and right field names must be disjoint")

    left = _validate_array(
        left_keys,
        expected_names=left_names,
        role="left keys",
        maximum_rows=_MAX_LEFT_ROWS,
    )
    right = _validate_array(
        right_records,
        expected_names=right_names,
        role="right records",
        maximum_rows=_MAX_RIGHT_ROWS,
    )
    pair_names = left_names + right_names
    pair_dtype = np.dtype(
        [
            *((name, left.dtype.fields[name][0]) for name in left_names),
            *((name, right.dtype.fields[name][0]) for name in right_names),
        ]
    )
    if pair_dtype.itemsize > _MAX_RECORD_BYTES:
        raise ValueError("combined pair width exceeds the supported limit")

    if existing_pairs is None:
        existing = None
        existing_count = 0
        existing_bytes = 0
    else:
        existing = _validate_array(
            existing_pairs,
            expected_names=pair_names,
            role="existing pairs",
            maximum_rows=_MAX_EXISTING_ROWS,
        )
        if existing.dtype != pair_dtype:
            raise TypeError("existing pairs must match the exact combined dtype")
        existing_count = len(existing)
        existing_bytes = existing.nbytes

    input_bytes = left.nbytes + right.nbytes + existing_bytes
    if input_bytes > _MAX_INPUT_BYTES:
        raise ValueError("input arrays exceed the aggregate byte limit")

    unique_left_rows: list[StructuredRow] = []
    seen_left_rows: set[StructuredRow] = set()
    for row_index in range(len(left)):
        row = _row_key(left, left_names, row_index)
        if row not in seen_left_rows:
            seen_left_rows.add(row)
            unique_left_rows.append(row)

    cartesian_rows = len(unique_left_rows) * len(right)
    if cartesian_rows > _MAX_CARTESIAN_ROWS:
        raise ValueError("the Cartesian product exceeds the supported row limit")
    prospective_rows = existing_count + cartesian_rows
    if prospective_rows > _MAX_PROSPECTIVE_ROWS:
        raise ValueError("existing and generated pairs exceed the row limit")
    if prospective_rows * pair_dtype.itemsize > _MAX_PROSPECTIVE_BYTES:
        raise ValueError("prospective pairs exceed the NumPy payload byte limit")

    ordered_pairs: list[StructuredRow] = []
    seen_pairs: set[StructuredRow] = set()

    def remember(pair: StructuredRow) -> None:
        if pair not in seen_pairs:
            seen_pairs.add(pair)
            ordered_pairs.append(pair)

    if existing is not None:
        for row_index in range(len(existing)):
            remember(_row_key(existing, pair_names, row_index))

    right_rows = tuple(_row_key(right, right_names, row_index) for row_index in range(len(right)))
    for left_row in unique_left_rows:
        for right_row in right_rows:
            remember(left_row + right_row)

    result = np.empty(len(ordered_pairs), dtype=pair_dtype)
    for row_index, pair in enumerate(ordered_pairs):
        result[row_index] = pair
    result.setflags(write=False)
    return result
```

## Example

```python
left_dtype = np.dtype([("scope", "U8"), ("level", "u1")])
right_dtype = np.dtype([("label", "U8"), ("enabled", "?")])
pair_dtype = np.dtype(
    [
        ("scope", "U8"),
        ("level", "u1"),
        ("label", "U8"),
        ("enabled", "?"),
    ]
)

left = np.array([("blue", 2), ("red", 1), ("blue", 2)], dtype=left_dtype)
right = np.array(
    [("pen", True), ("cup", False), ("pen", True)],
    dtype=right_dtype,
)
existing = np.array(
    [
        ("red", 1, "cup", False),
        ("red", 1, "cup", False),
        ("blue", 2, "pen", True),
    ],
    dtype=pair_dtype,
)
before = (left.copy(), right.copy(), existing.copy())

pairs = expand_and_merge_structured_pairs(
    left,
    right,
    left_field_names=("scope", "level"),
    right_field_names=("label", "enabled"),
    existing_pairs=existing,
)

try:
    pairs[0] = pairs[0]
except ValueError:
    assignment_rejected = True
else:
    assignment_rejected = False

empty = expand_and_merge_structured_pairs(
    np.empty(0, dtype=left_dtype),
    np.empty(0, dtype=right_dtype),
    left_field_names=("scope", "level"),
    right_field_names=("label", "enabled"),
)
assert (
    pairs.tolist()
    == [
        ("red", 1, "cup", False),
        ("blue", 2, "pen", True),
        ("blue", 2, "cup", False),
        ("red", 1, "pen", True),
    ]
    and all(
        np.array_equal(current, original)
        for current, original in zip((left, right, existing), before, strict=True)
    )
    and pairs.flags.owndata
    and not pairs.flags.writeable
    and assignment_rejected
    and empty.shape == (0,)
    and empty.dtype == pair_dtype
    and empty.flags.owndata
    and not empty.flags.writeable
)
```

## Trade-offs and Limitations

For `l` left rows, `u` first-seen unique left rows, `r` right rows, `e`
existing rows, `f` distinct output pairs, and `k` total fields, expected time is
`O((l + r + e + u * r) * k)` with ordinary hash-table behavior. Python keys
and sets use `O((u + r + f) * k)` logical scalar references, and the returned
NumPy payload uses `O(f * w)` bytes for record width `w`. The prospective byte
limit covers NumPy payload bytes; bounded prospective rows, field counts, and
field widths separately limit Python bookkeeping overhead.

Schema validation intentionally rejects ndarray subclasses, masked arrays,
object fields, floats, complex numbers, datetime-like values, nested records,
subarrays, dtype metadata, titles, padding, and overlapping fields. Fixed-width
byte and Unicode values are compared by their logical NumPy scalar values
rather than raw padding bytes. Inputs are only read, but concurrent mutation is
unsupported. The read-only flag prevents ordinary result assignment, not
hostile memory tampering. No ranking, recommendation, model execution,
fetching, or persistence is performed.

## Related Snippets

<!-- catalog:related:start -->
- [Group a One-Dimensional NumPy Structured Array into Stable Index Sets](../algorithms-data-structures/group-a-one-dimensional-numpy-structured-array-into-stable-index-sets.md)
- [Merge Bounded Row Batches with a First-Seen Schema Union](merge-bounded-row-batches-with-a-first-seen-schema-union.md)
- [Group Items by an Exact Compatibility Signature and Report Unmatched Inputs](group-items-by-an-exact-compatibility-signature-and-report-unmatched-inputs.md)
<!-- catalog:related:end -->
