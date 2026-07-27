---
title: "Reshape Repeated pandas Column Families with Wide-to-Long"
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
  - aggregate-pandas-groups-into-explicit-flat-columns.md
  - enforce-a-many-to-one-pandas-left-merge-contract.md
---

# Reshape Repeated pandas Column Families with Wide-to-Long

## Idea and Problem

Turn repeated period-suffixed pandas column families into one row per identifier and numeric period.

The wrapper validates the complete family grammar before delegating the actual
reshape to `pandas.wide_to_long`. Every metric stub must expose the same
contiguous period set, identifiers must be non-missing and unique, and the
result is explicitly sorted by original input row and then period.

## When to Use

Use this integration when a bounded table stores the same measurements in
period-suffixed columns and a long table is easier to group, plot, or export.
Column names follow the fixed `<stub>__<period>` grammar: stubs are bounded
ASCII identifiers, the separator is `__`, and periods are canonical unsigned
ASCII decimals without leading zeroes. The function accepts one to eight ID
columns and one to thirty-two metric stubs.

The input must be an exact DataFrame with empty attrs, unique exact-string
columns, a passive one-level index, and supported passive values. Object
payloads and unknown extension dtypes are rejected before copying, null
checks, duplicate detection, or reshaping. The original index is validated but
deliberately omitted from the returned table; include it as an explicit ID
column if it carries domain meaning.

## Implementation

```python
import numpy as np
import pandas as pd


_SEPARATOR = "__"
_PERIOD_COLUMN = "period"
_MAX_FRAME_ROWS = 100_000
_MAX_OUTPUT_ROWS = 500_000
_MAX_FRAME_COLUMNS = 256
_MAX_ID_COLUMNS = 8
_MAX_METRIC_STUBS = 32
_MAX_COLUMN_CHARS = 256
_MAX_TEXT_CHARS = 4_096
_MAX_PERIOD = 1_000_000


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


def _validate_passive_series(series: pd.Series, *, field: str) -> None:
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
        return
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


def _name_tuple(
    values: object,
    *,
    field: str,
    maximum: int,
) -> tuple[str, ...]:
    if type(values) is not tuple:
        raise TypeError(f"{field} must be an exact tuple")
    if not 1 <= len(values) <= maximum:
        raise ValueError(f"{field} count is outside the supported range")
    names = tuple(_bounded_name(value, field=field) for value in values)
    if len(set(names)) != len(names):
        raise ValueError(f"{field} must be unique")
    return names


def _metric_stub(value: str) -> str:
    if _SEPARATOR in value:
        raise ValueError("metric stubs must not contain the fixed separator")
    if not ("A" <= value[0] <= "Z" or "a" <= value[0] <= "z"):
        raise ValueError("metric stubs must start with an ASCII letter")
    if any(
        not (
            "A" <= character <= "Z"
            or "a" <= character <= "z"
            or "0" <= character <= "9"
            or character == "_"
        )
        for character in value[1:]
    ):
        raise ValueError("metric stubs must be ASCII identifiers")
    return value


def _canonical_period(suffix: str) -> int:
    if not suffix or any(not "0" <= character <= "9" for character in suffix):
        raise ValueError("period suffixes must contain only ASCII decimals")
    if len(suffix) > 1 and suffix[0] == "0":
        raise ValueError("period suffixes must not contain leading zeroes")
    period = int(suffix)
    if period > _MAX_PERIOD:
        raise ValueError("a period suffix exceeds the supported limit")
    return period


def _family_periods(
    columns: tuple[str, ...],
    *,
    id_columns: tuple[str, ...],
    metric_stubs: tuple[str, ...],
) -> tuple[int, ...]:
    families = {stub: {} for stub in metric_stubs}
    for column in columns:
        if column in id_columns:
            continue
        matches = tuple(
            stub
            for stub in metric_stubs
            if column.startswith(f"{stub}{_SEPARATOR}")
        )
        if len(matches) != 1:
            raise ValueError("every non-ID column must belong to one metric family")
        stub = matches[0]
        suffix = column[len(stub) + len(_SEPARATOR) :]
        period = _canonical_period(suffix)
        if period in families[stub]:
            raise ValueError("a metric family contains a duplicate numeric period")
        families[stub][period] = column

    expected = tuple(sorted(families[metric_stubs[0]]))
    if not expected:
        raise ValueError("every metric family must contain at least one period")
    if any(tuple(sorted(families[stub])) != expected for stub in metric_stubs):
        raise ValueError("every metric family must expose the same period set")
    if any(right != left + 1 for left, right in zip(expected, expected[1:])):
        raise ValueError("the shared period set must be contiguous")
    return expected


def _private_position_column(occupied: set[str]) -> str:
    candidate = "__input_position__"
    while candidate in occupied:
        candidate = f"_{candidate}"
    return candidate


def reshape_repeated_families(
    frame: pd.DataFrame,
    *,
    id_columns: tuple[str, ...],
    metric_stubs: tuple[str, ...],
) -> pd.DataFrame:
    columns = _frame_columns(frame)
    _validate_passive_index(frame.index)
    ids = _name_tuple(
        id_columns,
        field="ID columns",
        maximum=_MAX_ID_COLUMNS,
    )
    stubs = tuple(
        _metric_stub(value)
        for value in _name_tuple(
            metric_stubs,
            field="metric stubs",
            maximum=_MAX_METRIC_STUBS,
        )
    )
    if not set(ids).issubset(columns):
        raise ValueError("every ID column must exist in the frame")
    if _PERIOD_COLUMN in ids or _PERIOD_COLUMN in stubs:
        raise ValueError("the period output name must remain reserved")
    if set(ids).intersection(stubs):
        raise ValueError("ID columns and metric stubs must not overlap")
    family_prefixes = tuple(f"{stub}{_SEPARATOR}" for stub in stubs)
    if any(
        left != right and right.startswith(left)
        for left in family_prefixes
        for right in family_prefixes
    ):
        raise ValueError("metric family prefixes must not overlap")
    if any(
        identifier.startswith(prefix)
        for identifier in ids
        for prefix in family_prefixes
    ):
        raise ValueError("ID columns must not overlap a metric family")
    periods = _family_periods(
        columns,
        id_columns=ids,
        metric_stubs=stubs,
    )
    if len(frame) * len(periods) > _MAX_OUTPUT_ROWS:
        raise ValueError("the long result would exceed the supported row limit")

    selected = []
    for position, column in enumerate(columns):
        series = frame.iloc[:, position]
        if type(series) is not pd.Series:
            raise TypeError("column selection must produce an exact pandas Series")
        _validate_passive_series(series, field=f"column {column!r}")
        selected.append(series)

    working = pd.DataFrame(
        {
            column: series.array.copy()
            for column, series in zip(columns, selected, strict=True)
        },
        copy=True,
    )
    key_frame = working.loc[:, list(ids)]
    if key_frame.isna().to_numpy(dtype=bool).any():
        raise ValueError("ID tuples must not contain missing values")
    if bool(key_frame.duplicated(keep=False).any()):
        raise ValueError("ID tuples must be unique")

    position_column = _private_position_column(set(columns))
    working[position_column] = np.arange(len(working), dtype=np.int64)
    long = pd.wide_to_long(
        working,
        stubnames=list(stubs),
        i=list(ids),
        j=_PERIOD_COLUMN,
        sep=_SEPARATOR,
        suffix="[0-9]+",
    ).reset_index()
    if len(long) != len(frame) * len(periods):
        raise RuntimeError("wide_to_long produced an unexpected row count")
    long = long.sort_values(
        [position_column, _PERIOD_COLUMN],
        kind="stable",
        ignore_index=True,
    )
    return long.loc[:, [*ids, _PERIOD_COLUMN, *stubs]].copy()
```

## Example

```python
wide = pd.DataFrame(
    {
        "account_id": np.array([20, 10], dtype=np.int64),
        "visits__2": np.array([4, 2], dtype=np.int64),
        "cost__1": np.array([1.25, 3.5], dtype=np.float64),
        "visits__1": np.array([3, 1], dtype=np.int64),
        "cost__2": np.array([2.0, 4.25], dtype=np.float64),
    },
    index=pd.RangeIndex(10, 12, name="source_row"),
)
before = wide.copy(deep=True)

long = reshape_repeated_families(
    wide,
    id_columns=("account_id",),
    metric_stubs=("visits", "cost"),
)
expected = pd.DataFrame(
    {
        "account_id": [20, 20, 10, 10],
        "period": [1, 2, 1, 2],
        "visits": [3, 4, 1, 2],
        "cost": [1.25, 2.0, 3.5, 4.25],
    }
)

rejections = []
for invalid in (
    wide.drop(columns="cost__2"),
    wide.assign(account_id=[20, 20]),
    wide.rename(columns={"visits__2": "visits__02"}),
):
    try:
        reshape_repeated_families(
            invalid,
            id_columns=("account_id",),
            metric_stubs=("visits", "cost"),
        )
    except ValueError:
        rejections.append(True)

assert (
    long.equals(expected),
    wide.equals(before),
    rejections,
) == (True, True, [True, True, True])
```

## Trade-offs and Limitations

Validation scans every supported column, while duplicate detection hashes the
ID tuples. Reshaping allocates a long frame whose row count is `input rows *
periods`; the explicit output limit bounds that expansion but does not make it
streaming. A database unpivot or partitioned engine is a better fit when the
wide table approaches memory limits.

The grammar intentionally rejects signed, fractional, padded, non-ASCII, or
non-contiguous period labels, unrelated payload columns, nullable IDs, and
arbitrary suffix regexes. It also discards the original index and frame attrs.
These choices make matching and ordering deterministic, but a different naming
scheme or sparse family needs its own explicit policy. Dtypes may be promoted
when period columns within one family differ, following pandas concatenation
rules.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Bounded Structural Profile of a pandas DataFrame](build-a-bounded-structural-profile-of-a-pandas-dataframe.md)
- [Aggregate pandas Groups into Explicit Flat Columns](aggregate-pandas-groups-into-explicit-flat-columns.md)
- [Enforce a Many-to-One pandas Left-Merge Contract](enforce-a-many-to-one-pandas-left-merge-contract.md)
<!-- catalog:related:end -->
