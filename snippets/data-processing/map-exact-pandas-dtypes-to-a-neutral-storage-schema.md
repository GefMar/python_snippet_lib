---
title: "Map Exact pandas Dtypes to a Neutral Storage Schema"
snippet_type: integration
use_cases:
  - data-transformation
  - interoperability
  - validation
tested_python:
  - "3.14"
dependencies:
  - name: numpy
    version: "2.5.1"
  - name: pandas
    version: "3.0.5"
related:
  - build-a-bounded-structural-profile-of-a-pandas-dataframe.md
  - audit-candidate-pandas-downcasts-before-applying-them.md
  - read-bounded-csv-text-into-pandas-under-an-explicit-schema.md
---

# Map Exact pandas Dtypes to a Neutral Storage Schema

## Idea and Problem

Map a deliberately small set of exact pandas dtypes to immutable, provider-neutral storage descriptors without reading any cell values.

Each descriptor records a column name, a neutral storage type, and whether the
source dtype can represent a missing value. That last field is a capability of
the dtype: it does not claim that the current column contains a missing value.
The result preserves input-column order and can be translated separately into
a concrete database, file-format, or interchange schema.

## When to Use

Use this integration when a bounded in-memory DataFrame has already crossed a
trusted parsing boundary and a later adapter needs only its dtype-level shape.
The accepted set is closed: native Boolean, signed and unsigned integer, and
32- or 64-bit floating-point NumPy dtypes; nullable pandas Boolean, integer,
and floating-point dtypes; Python-backed pandas strings; and timezone-naive
`datetime64[ns]`.

The DataFrame must be exact rather than subclassed, have no attrs, use a
one-level passive index, and have unique bounded exact-text column names. Add a
reviewed mapping explicitly when another dtype is required. In particular,
object, category, timezone-aware datetime, Arrow-backed, timedelta, complex,
structured, non-native, and unknown extension dtypes are rejected.

## Implementation

```python
from dataclasses import dataclass

import numpy as np
import pandas as pd


_MAX_ROWS = 100_000
_MAX_COLUMNS = 256
_MAX_NAME_CHARACTERS = 128
_MAX_NAME_UTF8_BYTES = 512
_MAX_TOTAL_NAME_UTF8_BYTES = 16_384
_PASSIVE_INDEX_TYPES = (
    pd.Index,
    pd.RangeIndex,
    pd.DatetimeIndex,
    pd.TimedeltaIndex,
)


@dataclass(frozen=True, slots=True)
class StorageField:
    name: str
    storage_type: str
    supports_missing: bool


_EXTENSION_DTYPES = {
    pd.BooleanDtype: ("boolean", pd.arrays.BooleanArray),
    pd.Int8Dtype: ("int8", pd.arrays.IntegerArray),
    pd.Int16Dtype: ("int16", pd.arrays.IntegerArray),
    pd.Int32Dtype: ("int32", pd.arrays.IntegerArray),
    pd.Int64Dtype: ("int64", pd.arrays.IntegerArray),
    pd.UInt8Dtype: ("uint8", pd.arrays.IntegerArray),
    pd.UInt16Dtype: ("uint16", pd.arrays.IntegerArray),
    pd.UInt32Dtype: ("uint32", pd.arrays.IntegerArray),
    pd.UInt64Dtype: ("uint64", pd.arrays.IntegerArray),
    pd.Float32Dtype: ("float32", pd.arrays.FloatingArray),
    pd.Float64Dtype: ("float64", pd.arrays.FloatingArray),
}


def _bounded_text(value: object, *, field: str) -> tuple[str, int]:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if not value or len(value) > _MAX_NAME_CHARACTERS:
        raise ValueError(f"{field} length is outside the supported range")
    if value != value.strip() or not value.isprintable():
        raise ValueError(f"{field} must be stripped printable text")
    try:
        byte_count = len(value.encode("utf-8"))
    except UnicodeEncodeError as error:
        raise ValueError(f"{field} must be valid UTF-8 text") from error
    if byte_count > _MAX_NAME_UTF8_BYTES:
        raise ValueError(f"{field} exceeds the supported UTF-8 byte limit")
    return value, byte_count


def _validate_optional_axis_name(value: object, *, field: str) -> None:
    if value is not None:
        _bounded_text(value, field=field)


def _column_names(columns: pd.Index) -> tuple[str, ...]:
    _validate_optional_axis_name(columns.name, field="column-axis name")
    if type(columns) is pd.RangeIndex and len(columns) == 0:
        return ()
    if type(columns) is not pd.Index:
        raise TypeError("columns must use a passive one-level pandas Index")
    if len(columns) > _MAX_COLUMNS:
        raise ValueError("frame exceeds the supported column limit")

    dtype = columns.dtype
    array = columns.array
    if isinstance(dtype, np.dtype):
        if (
            dtype.kind != "O"
            or dtype.metadata is not None
            or type(array) is not pd.arrays.NumpyExtensionArray
        ):
            raise TypeError("columns must use a passive text dtype")
    elif (
        type(dtype) is pd.StringDtype
        and dtype.storage == "python"
        and type(array) is pd.arrays.StringArray
    ):
        pass
    else:
        raise TypeError("columns must use a passive text dtype")

    names = []
    total_bytes = 0
    for value in array:
        name, byte_count = _bounded_text(value, field="column name")
        total_bytes += byte_count
        if total_bytes > _MAX_TOTAL_NAME_UTF8_BYTES:
            raise ValueError("column names exceed the total UTF-8 byte limit")
        names.append(name)
    if len(set(names)) != len(names):
        raise ValueError("column names must be unique")
    return tuple(names)


def _native_storage_type(dtype: np.dtype) -> tuple[str, bool] | None:
    if dtype.metadata is not None or not dtype.isnative:
        return None
    if dtype.kind == "b" and dtype.itemsize == 1:
        return "boolean", False
    if dtype.kind in {"i", "u"} and dtype.itemsize in {1, 2, 4, 8}:
        prefix = "int" if dtype.kind == "i" else "uint"
        return f"{prefix}{dtype.itemsize * 8}", False
    if dtype.kind == "f" and dtype.itemsize in {4, 8}:
        return f"float{dtype.itemsize * 8}", True
    if dtype == np.dtype("datetime64[ns]"):
        return "timestamp-ns", True
    return None


def _storage_field(name: str, series: pd.Series) -> StorageField:
    dtype = series.dtype
    array = series.array
    if isinstance(dtype, np.dtype):
        mapped = _native_storage_type(dtype)
        expected_array_type = (
            pd.arrays.DatetimeArray
            if dtype.kind == "M"
            else pd.arrays.NumpyExtensionArray
        )
        if mapped is None or type(array) is not expected_array_type:
            raise TypeError(f"column {name!r} has an unsupported dtype")
        storage_type, supports_missing = mapped
        return StorageField(name, storage_type, supports_missing)

    extension = _EXTENSION_DTYPES.get(type(dtype))
    if extension is not None:
        storage_type, expected_array_type = extension
        if type(array) is not expected_array_type:
            raise TypeError(f"column {name!r} has an unsupported array type")
        return StorageField(name, storage_type, True)

    if (
        type(dtype) is pd.StringDtype
        and dtype.storage == "python"
        and type(array) is pd.arrays.StringArray
    ):
        return StorageField(name, "utf8", True)
    raise TypeError(f"column {name!r} has an unsupported dtype")


def map_exact_pandas_dtypes(
    frame: pd.DataFrame,
) -> tuple[StorageField, ...]:
    if type(frame) is not pd.DataFrame:
        raise TypeError("frame must be an exact pandas DataFrame")
    if frame.attrs:
        raise ValueError("frame attrs must be empty")
    if type(frame.index) not in _PASSIVE_INDEX_TYPES:
        raise TypeError("frame must use a passive one-level index")
    _validate_optional_axis_name(frame.index.name, field="index name")
    if len(frame.index) > _MAX_ROWS:
        raise ValueError("frame exceeds the supported row limit")

    names = _column_names(frame.columns)
    fields = []
    for position, name in enumerate(names):
        series = frame.iloc[:, position]
        if type(series) is not pd.Series:
            raise TypeError("column selection must return an exact pandas Series")
        fields.append(_storage_field(name, series))
    return tuple(fields)
```

## Example

```python
import numpy as np
import pandas as pd


frame = pd.DataFrame(
    {
        "enabled": pd.Series([True, pd.NA], dtype="boolean"),
        "attempts": pd.Series([1, 2], dtype=np.int16),
        "ratio": pd.Series([1.0, 2.0], dtype=np.float64),
        "label": pd.Series(
            ["ready", None],
            dtype=pd.StringDtype(storage="python"),
        ),
        "recorded_at": pd.Series(
            ["2026-01-01", None],
            dtype="datetime64[ns]",
        ),
    }
)
before = frame.copy(deep=True)

schema = map_exact_pandas_dtypes(frame)

assert schema == (
    StorageField("enabled", "boolean", True),
    StorageField("attempts", "int16", False),
    StorageField("ratio", "float64", True),
    StorageField("label", "utf8", True),
    StorageField("recorded_at", "timestamp-ns", True),
)
pd.testing.assert_frame_equal(frame, before)
assert tuple(field.name for field in schema) == tuple(frame.columns)
```

## Trade-offs and Limitations

This is a dtype mapper, not schema inference. It never inspects values, so it
does not calculate observed nullability, text lengths, numeric ranges, decimal
precision, uniqueness, or key constraints. For example, every supported
floating-point dtype is marked as missing-capable even if its current values
are all finite, while a native integer dtype is not.

The neutral names deliberately omit provider-specific encodings and limits.
Translate them in a separate storage adapter and define overflow, timestamp,
and missing-value behavior there. The function neither casts nor copies column
values and does not mutate the input. Its strict dtype and array checks trade
automatic coverage for predictable behavior; unsupported dtypes need an
explicitly reviewed extension rather than a fallback through `str(dtype)`.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Bounded Structural Profile of a pandas DataFrame](build-a-bounded-structural-profile-of-a-pandas-dataframe.md)
- [Audit Candidate pandas Downcasts Before Applying Them](audit-candidate-pandas-downcasts-before-applying-them.md)
- [Read Bounded CSV Text into pandas Under an Explicit Schema](read-bounded-csv-text-into-pandas-under-an-explicit-schema.md)
<!-- catalog:related:end -->
