---
title: "Build a Bounded Structural Profile of a pandas DataFrame"
snippet_type: integration
use_cases:
  - data-transformation
  - observability
  - validation
tested_python:
  - "3.14"
dependencies:
  - name: numpy
    version: "2.5.1"
  - name: pandas
    version: "3.0.5"
related:
  - reshape-repeated-pandas-column-families-with-wide-to-long.md
  - audit-pandas-missing-value-shares-against-column-policies.md
  - audit-candidate-pandas-downcasts-before-applying-them.md
---

# Build a Bounded Structural Profile of a pandas DataFrame

## Idea and Problem

Summarize the shape, uniqueness, missingness, and cardinality of a bounded pandas DataFrame without retaining row samples or cell values in the report.

The result contains immutable scalar facts in input-column order. Optional key
columns add a tuple-uniqueness check, while the index is checked independently.
This makes the profile useful as a compact structural diagnostic rather than a
second, potentially sensitive copy of the data.

## When to Use

Use this integration at an in-memory validation boundary when downstream code
needs a small schema-and-quality snapshot. The implementation accepts exact
DataFrames with unique exact-string columns, empty attrs, a passive one-level
index, and at most eight distinct key columns. Supported values are deliberately
limited to native NumPy Boolean, integer, floating-point, datetime, and
timedelta columns plus Python-backed pandas string columns.

The narrow dtype boundary is a safety property. Object columns, categoricals,
timezone-aware arrays, nullable numeric extensions, third-party extensions,
and non-native NumPy dtypes are rejected before values are copied or hashed.
Add a separately reviewed validator for any additional dtype instead of
allowing its equality, hashing, or copy hooks implicitly.

## Implementation

```python
from dataclasses import dataclass

import numpy as np
import pandas as pd


_MAX_FRAME_ROWS = 100_000
_MAX_FRAME_COLUMNS = 256
_MAX_KEY_COLUMNS = 8
_MAX_COLUMN_CHARS = 256
_MAX_TEXT_CHARS = 4_096


@dataclass(frozen=True, slots=True)
class ColumnStructure:
    name: str
    dtype: str
    missing_count: int
    non_missing_count: int
    distinct_count: int


@dataclass(frozen=True, slots=True)
class DataFrameStructure:
    row_count: int
    column_count: int
    index_unique: bool
    key_columns: tuple[str, ...]
    key_tuples_unique: bool | None
    columns: tuple[ColumnStructure, ...]


def _bounded_name(value: object, *, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if not value or len(value) > _MAX_COLUMN_CHARS:
        raise ValueError(f"{field} length is outside the supported range")
    if value != value.strip() or not value.isprintable():
        raise ValueError(f"{field} must be stripped printable text")
    return value


def _passive_scalar(value: object, *, field: str) -> object:
    if value is None or value is pd.NA or value is pd.NaT:
        return value
    if type(value) in (bool, int, float, pd.Timestamp, pd.Timedelta):
        return value
    if type(value) in (str, bytes):
        if len(value) > _MAX_TEXT_CHARS:
            raise ValueError(f"{field} exceeds the supported length")
        return value
    raise TypeError(f"{field} must be a passive scalar")


def _frame_columns(frame: object) -> tuple[str, ...]:
    if type(frame) is not pd.DataFrame:
        raise TypeError("frame must be an exact pandas DataFrame")
    if frame.attrs:
        raise ValueError("frame attrs must be empty")
    if len(frame) > _MAX_FRAME_ROWS:
        raise ValueError("frame exceeds the supported row limit")
    if type(frame.columns) is not pd.Index:
        raise TypeError("frame columns must use a passive pandas Index")
    _passive_scalar(frame.columns.name, field="column index name")
    column_dtype = frame.columns.dtype
    column_array = frame.columns.array
    if isinstance(column_dtype, np.dtype) and column_dtype.kind == "O":
        if type(column_array) is not pd.arrays.NumpyExtensionArray:
            raise TypeError("frame column array implementation is not supported")
    elif (
        type(column_dtype) is pd.StringDtype
        and column_dtype.storage == "python"
        and type(column_array) is pd.arrays.StringArray
    ):
        pass
    else:
        raise TypeError("frame columns must use a passive text dtype")
    columns = tuple(
        _bounded_name(column, field="frame column")
        for column in column_array
    )
    if len(columns) > _MAX_FRAME_COLUMNS:
        raise ValueError("frame exceeds the supported column limit")
    if len(set(columns)) != len(columns):
        raise ValueError("frame column names must be unique")
    return columns


def _validate_passive_index(index: pd.Index) -> None:
    if type(index) not in (
        pd.Index,
        pd.RangeIndex,
        pd.DatetimeIndex,
        pd.TimedeltaIndex,
    ):
        raise TypeError("frame index type is outside the passive boundary")
    _passive_scalar(index.name, field="index name")
    dtype = index.dtype
    array = index.array
    if isinstance(dtype, np.dtype):
        if not dtype.isnative:
            raise TypeError("frame index must use a native dtype")
        expected_array_type = {
            "M": pd.arrays.DatetimeArray,
            "m": pd.arrays.TimedeltaArray,
        }.get(dtype.kind, pd.arrays.NumpyExtensionArray)
        if type(array) is not expected_array_type:
            raise TypeError("frame index array implementation is not supported")
        if dtype.kind in {"b", "i", "u", "f", "M", "m"}:
            return
        if dtype.kind != "O":
            raise TypeError("frame index dtype is outside the passive boundary")
    elif type(dtype) is pd.StringDtype and dtype.storage == "python":
        if type(array) is not pd.arrays.StringArray:
            raise TypeError("frame index array implementation is not supported")
    else:
        raise TypeError("frame index dtype is outside the passive boundary")
    for label in array:
        _passive_scalar(label, field="index label")


def _validate_passive_series(series: pd.Series, *, field: str) -> str:
    dtype = series.dtype
    array = series.array
    if isinstance(dtype, np.dtype):
        if not dtype.isnative:
            raise TypeError(f"{field} must use a native NumPy dtype")
        if dtype.kind not in {"b", "i", "u", "f", "M", "m"}:
            raise TypeError(f"{field} dtype is outside the passive boundary")
        expected_array_type = {
            "M": pd.arrays.DatetimeArray,
            "m": pd.arrays.TimedeltaArray,
        }.get(dtype.kind, pd.arrays.NumpyExtensionArray)
        if type(array) is not expected_array_type:
            raise TypeError(f"{field} array implementation is not supported")
        return str(dtype)
    if type(dtype) is not pd.StringDtype or dtype.storage != "python":
        raise TypeError(f"{field} dtype is outside the passive boundary")
    if type(array) is not pd.arrays.StringArray:
        raise TypeError(f"{field} array implementation is not supported")
    for value in array:
        if value is pd.NA or (type(value) is float and value != value):
            continue
        if type(value) is not str:
            raise TypeError(f"{field} must contain exact text")
        if len(value) > _MAX_TEXT_CHARS:
            raise ValueError(f"{field} text exceeds the supported length")
    return str(dtype)


def _key_columns(
    values: object,
    *,
    known_columns: tuple[str, ...],
) -> tuple[str, ...]:
    if type(values) is not tuple:
        raise TypeError("key_columns must be an exact tuple")
    if len(values) > _MAX_KEY_COLUMNS:
        raise ValueError("key column count exceeds the supported limit")
    keys = tuple(
        _bounded_name(value, field="key column") for value in values
    )
    if len(set(keys)) != len(keys):
        raise ValueError("key columns must be unique")
    if not set(keys).issubset(known_columns):
        raise ValueError("every key column must exist in the frame")
    return keys


def build_structural_profile(
    frame: pd.DataFrame,
    *,
    key_columns: tuple[str, ...] = (),
) -> DataFrameStructure:
    columns = _frame_columns(frame)
    _validate_passive_index(frame.index)
    keys = _key_columns(key_columns, known_columns=columns)

    selected = []
    dtype_labels = []
    for position, column in enumerate(columns):
        series = frame.iloc[:, position]
        if type(series) is not pd.Series:
            raise TypeError("column selection must produce an exact pandas Series")
        dtype_labels.append(
            _validate_passive_series(series, field=f"column {column!r}")
        )
        selected.append(series)

    working = pd.DataFrame(
        {
            column: series.array.copy()
            for column, series in zip(columns, selected, strict=True)
        },
        index=frame.index.copy(deep=True),
        copy=True,
    )

    profiles = []
    for column, dtype_label in zip(columns, dtype_labels, strict=True):
        series = working[column]
        missing_count = int(series.isna().sum())
        profiles.append(
            ColumnStructure(
                name=column,
                dtype=dtype_label,
                missing_count=missing_count,
                non_missing_count=len(series) - missing_count,
                distinct_count=int(series.nunique(dropna=True)),
            )
        )

    if keys:
        key_tuples_unique = not bool(
            working.loc[:, list(keys)].duplicated(keep=False).any()
        )
    else:
        key_tuples_unique = None

    return DataFrameStructure(
        row_count=len(working),
        column_count=len(columns),
        index_unique=bool(working.index.is_unique),
        key_columns=keys,
        key_tuples_unique=key_tuples_unique,
        columns=tuple(profiles),
    )
```

## Example

```python
text_dtype = pd.StringDtype(storage="python", na_value=pd.NA)
frame = pd.DataFrame(
    {
        "record_id": np.array([10, 20, 30], dtype=np.int64),
        "region": pd.array(["north", "south", "north"], dtype=text_dtype),
        "score": np.array([1.5, np.nan, 1.5], dtype=np.float64),
    },
    index=pd.Index(["row-a", "row-b", "row-c"], name="source"),
)
before = frame.copy(deep=True)

profile = build_structural_profile(frame, key_columns=("record_id",))


class ActiveValue:
    hash_calls = 0

    def __hash__(self) -> int:
        type(self).hash_calls += 1
        return 1


active_frame = pd.DataFrame({"payload": [ActiveValue()]})
ActiveValue.hash_calls = 0
try:
    build_structural_profile(active_frame)
except TypeError:
    active_object_rejected = True
else:
    active_object_rejected = False

assert (
    profile,
    frame.equals(before),
    active_object_rejected,
    ActiveValue.hash_calls,
) == (
    DataFrameStructure(
        row_count=3,
        column_count=3,
        index_unique=True,
        key_columns=("record_id",),
        key_tuples_unique=True,
        columns=(
            ColumnStructure("record_id", "int64", 0, 3, 3),
            ColumnStructure("region", "string", 0, 3, 2),
            ColumnStructure("score", "float64", 1, 2, 1),
        ),
    ),
    True,
    True,
    0,
)
```

## Trade-offs and Limitations

Each column is validated, copied, scanned for missing values, and hashed for
distinct counting. Runtime is approximately `O(rows * columns)`, while peak
memory includes a frame-sized snapshot and pandas' temporary hash tables. The
fixed limits bound the work but can still be too generous for wide values or a
tight memory budget; lower them at the application boundary when necessary.

Distinct counts exclude missing values, and duplicate key tuples use pandas'
equality and null semantics. The report intentionally omits minima, maxima,
quantiles, frequencies, and samples because those fields can disclose data and
need domain-specific policies. It is a diagnostic snapshot, not a stable
schema format or a substitute for data-contract validation. Concurrent caller
mutation is outside the contract.

## Related Snippets

<!-- catalog:related:start -->
- [Reshape Repeated pandas Column Families with Wide-to-Long](reshape-repeated-pandas-column-families-with-wide-to-long.md)
- [Audit pandas Missing-Value Shares Against Column Policies](audit-pandas-missing-value-shares-against-column-policies.md)
- [Audit Candidate pandas Downcasts Before Applying Them](audit-candidate-pandas-downcasts-before-applying-them.md)
<!-- catalog:related:end -->
