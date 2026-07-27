---
title: "Derive an Other Bucket from Exact pandas Totals"
snippet_type: integration
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies:
  - name: numpy
    version: "2.5.1"
  - name: pandas
    version: "3.0.5"
related:
  - aggregate-pandas-groups-into-explicit-flat-columns.md
  - build-a-bounded-structural-profile-of-a-pandas-dataframe.md
  - ../algorithms-data-structures/apportion-a-non-negative-integer-total-without-rounding-drift.md
---

# Derive an Other Bucket from Exact pandas Totals

## Idea and Problem

Replace each declared total row with the exact non-negative amount not accounted for by that group's detail rows.

Subtracting with Python integers avoids NumPy's fixed-width summation overflow.
The transformation keeps every row in place: detail rows retain their order,
and the sole total for each group becomes an `Other` row containing the
residual. Validation happens before a new frame is constructed, so an invalid
group cannot produce a partial result.

## When to Use

Use this integration for a bounded in-memory table whose total is authoritative
and whose visible buckets must be completed without rounding. The frame must
have only the declared group columns, one bucket column, and one value column.
Group and bucket columns use Python-backed pandas string dtypes, the value
column is native NumPy `int64`, and the positional index is an exact
`RangeIndex`. Labels are non-empty, non-missing, stripped printable strings.

Each observed group must contain exactly one row with the explicit total label.
Detail bucket labels must be unique within a group, and the requested residual
label must not already occur. This is a good fit for exact counts or integer
minor units. Convert decimals to exact integers before calling it; do not use
the function to infer totals from rounded floating-point values.

## Implementation

```python
import numpy as np
import pandas as pd


_MAX_FRAME_ROWS = 100_000
_MAX_FRAME_COLUMNS = 10
_MAX_FRAME_CELLS = 1_000_000
_MAX_GROUP_COLUMNS = 8
_MAX_COLUMN_CHARACTERS = 128
_MAX_LABEL_CHARACTERS = 4_096
_INT64_MAX = int(np.iinfo(np.int64).max)


def _bounded_text(value: object, *, field: str, limit: int) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if not value or len(value) > limit:
        raise ValueError(f"{field} length is outside the supported range")
    if value != value.strip() or not value.isprintable():
        raise ValueError(f"{field} must be stripped printable text")
    return value


def _declared_columns(
    group_columns: object,
    *,
    bucket_column: object,
    value_column: object,
) -> tuple[tuple[str, ...], str, str]:
    if type(group_columns) is not tuple:
        raise TypeError("group_columns must be an exact tuple")
    if not 1 <= len(group_columns) <= _MAX_GROUP_COLUMNS:
        raise ValueError("group column count is outside the supported range")

    groups = tuple(
        _bounded_text(
            column,
            field="group column",
            limit=_MAX_COLUMN_CHARACTERS,
        )
        for column in group_columns
    )
    bucket = _bounded_text(
        bucket_column,
        field="bucket column",
        limit=_MAX_COLUMN_CHARACTERS,
    )
    value = _bounded_text(
        value_column,
        field="value column",
        limit=_MAX_COLUMN_CHARACTERS,
    )
    declared = (*groups, bucket, value)
    if len(set(declared)) != len(declared):
        raise ValueError("declared column names must be unique")
    return groups, bucket, value


def _frame_columns(
    frame: object,
    *,
    declared: tuple[str, ...],
) -> tuple[str, ...]:
    if type(frame) is not pd.DataFrame:
        raise TypeError("frame must be an exact pandas DataFrame")
    if frame.attrs:
        raise ValueError("frame attrs must be empty")

    row_count = len(frame)
    column_count = len(frame.columns)
    if row_count > _MAX_FRAME_ROWS:
        raise ValueError("frame exceeds the supported row limit")
    if column_count > _MAX_FRAME_COLUMNS:
        raise ValueError("frame exceeds the supported column limit")
    if row_count * column_count > _MAX_FRAME_CELLS:
        raise ValueError("frame exceeds the supported cell limit")
    if type(frame.index) is not pd.RangeIndex:
        raise TypeError("frame must use an exact RangeIndex")
    if frame.index.name is not None:
        _bounded_text(
            frame.index.name,
            field="index name",
            limit=_MAX_COLUMN_CHARACTERS,
        )
    if type(frame.columns) is not pd.Index:
        raise TypeError("frame columns must use an exact pandas Index")

    columns = tuple(
        _bounded_text(
            column,
            field="frame column",
            limit=_MAX_COLUMN_CHARACTERS,
        )
        for column in frame.columns
    )
    if len(set(columns)) != len(columns):
        raise ValueError("frame column names must be unique")
    if len(columns) != len(declared) or set(columns) != set(declared):
        raise ValueError("frame columns must exactly match the declaration")
    return columns


def _text_values(series: pd.Series, *, field: str) -> tuple[str, ...]:
    if type(series) is not pd.Series:
        raise TypeError(f"{field} selection must produce an exact pandas Series")
    dtype = series.dtype
    if type(dtype) is not pd.StringDtype or dtype.storage != "python":
        raise TypeError(f"{field} must use a Python-backed pandas string dtype")
    if type(series.array) is not pd.arrays.StringArray:
        raise TypeError(f"{field} array implementation is not supported")

    values = []
    for value in series.array:
        if type(value) is not str:
            raise ValueError(f"{field} labels must not be missing")
        values.append(
            _bounded_text(
                value,
                field=f"{field} label",
                limit=_MAX_LABEL_CHARACTERS,
            )
        )
    return tuple(values)


def _non_negative_int64_values(
    series: pd.Series,
    *,
    field: str,
) -> tuple[int, ...]:
    if type(series) is not pd.Series:
        raise TypeError(f"{field} selection must produce an exact pandas Series")
    dtype = series.dtype
    if not (
        isinstance(dtype, np.dtype)
        and dtype.metadata is None
        and dtype == np.dtype("int64")
        and dtype.isnative
    ):
        raise TypeError(f"{field} must use the native NumPy int64 dtype")
    if type(series.array) is not pd.arrays.NumpyExtensionArray:
        raise TypeError(f"{field} array implementation is not supported")

    values = tuple(int(value) for value in series.array)
    if any(value < 0 for value in values):
        raise ValueError(f"{field} values must be non-negative")
    return values


def derive_other_bucket(
    frame: pd.DataFrame,
    *,
    group_columns: tuple[str, ...],
    bucket_column: str,
    value_column: str,
    total_label: str,
    other_label: str,
) -> pd.DataFrame:
    groups, bucket, value = _declared_columns(
        group_columns,
        bucket_column=bucket_column,
        value_column=value_column,
    )
    total = _bounded_text(
        total_label,
        field="total label",
        limit=_MAX_LABEL_CHARACTERS,
    )
    other = _bounded_text(
        other_label,
        field="other label",
        limit=_MAX_LABEL_CHARACTERS,
    )
    if total == other:
        raise ValueError("total and other labels must be distinct")

    columns = _frame_columns(frame, declared=(*groups, bucket, value))
    group_arrays = tuple(
        _text_values(frame[column], field=f"group column {column!r}")
        for column in groups
    )
    buckets = _text_values(frame[bucket], field=f"bucket column {bucket!r}")
    amounts = _non_negative_int64_values(
        frame[value],
        field=f"value column {value!r}",
    )
    group_keys = tuple(zip(*group_arrays, strict=True))

    totals: dict[tuple[str, ...], int] = {}
    detail_sums: dict[tuple[str, ...], int] = {}
    detail_buckets: dict[tuple[str, ...], set[str]] = {}
    groups_seen: set[tuple[str, ...]] = set()

    for group_key, bucket_name, amount in zip(
        group_keys,
        buckets,
        amounts,
        strict=True,
    ):
        groups_seen.add(group_key)
        if bucket_name == other:
            raise ValueError("the other label already exists in the frame")
        if bucket_name == total:
            if group_key in totals:
                raise ValueError("a group contains more than one total row")
            totals[group_key] = amount
            continue

        known_buckets = detail_buckets.setdefault(group_key, set())
        if bucket_name in known_buckets:
            raise ValueError("detail bucket labels must be unique within a group")
        known_buckets.add(bucket_name)

        current_sum = detail_sums.get(group_key, 0)
        if amount > _INT64_MAX - current_sum:
            raise OverflowError("a detail sum exceeds native int64")
        detail_sums[group_key] = current_sum + amount

    if set(totals) != groups_seen:
        raise ValueError("every group must contain exactly one total row")

    residuals: dict[tuple[str, ...], int] = {}
    for group_key in groups_seen:
        residual = totals[group_key] - detail_sums.get(group_key, 0)
        if residual < 0:
            raise ValueError("a group's detail sum exceeds its total")
        residuals[group_key] = residual

    output_buckets = tuple(
        other if bucket_name == total else bucket_name
        for bucket_name in buckets
    )
    output_amounts = tuple(
        residuals[group_key] if bucket_name == total else amount
        for group_key, bucket_name, amount in zip(
            group_keys,
            buckets,
            amounts,
            strict=True,
        )
    )

    output_data: dict[str, object] = {}
    for column in columns:
        if column == bucket:
            output_data[column] = pd.array(
                output_buckets,
                dtype=frame[bucket].dtype,
            )
        elif column == value:
            output_data[column] = np.array(output_amounts, dtype=np.int64)
        else:
            output_data[column] = frame[column].array.copy()

    return pd.DataFrame(
        output_data,
        index=frame.index.copy(deep=True),
        copy=True,
    )
```

## Example

```python
text_dtype = pd.StringDtype(storage="python", na_value=pd.NA)
frame = pd.DataFrame(
    {
        "region": pd.array(
            ["north", "south", "north", "north", "south", "south"],
            dtype=text_dtype,
        ),
        "bucket": pd.array(
            ["known-a", "Total", "Total", "known-b", "known-a", "known-b"],
            dtype=text_dtype,
        ),
        "amount": np.array([7, 20, 12, 3, 4, 6], dtype=np.int64),
    },
    index=pd.RangeIndex(10, 16, name="position"),
)
before = frame.copy(deep=True)

completed = derive_other_bucket(
    frame,
    group_columns=("region",),
    bucket_column="bucket",
    value_column="amount",
    total_label="Total",
    other_label="Other",
)


class ActiveLabel:
    hash_calls = 0

    def __hash__(self) -> int:
        type(self).hash_calls += 1
        return 1


active_frame = pd.DataFrame(
    {
        "region": np.array([ActiveLabel()], dtype=object),
        "bucket": pd.array(["Total"], dtype=text_dtype),
        "amount": np.array([1], dtype=np.int64),
    }
)
ActiveLabel.hash_calls = 0
try:
    derive_other_bucket(
        active_frame,
        group_columns=("region",),
        bucket_column="bucket",
        value_column="amount",
        total_label="Total",
        other_label="Other",
    )
except TypeError:
    active_label_rejected = True
else:
    active_label_rejected = False

assert (
    completed.to_dict("records"),
    completed.index.equals(frame.index),
    completed["amount"].dtype,
    frame.equals(before),
    active_label_rejected,
    ActiveLabel.hash_calls,
) == (
    [
        {"region": "north", "bucket": "known-a", "amount": 7},
        {"region": "south", "bucket": "Other", "amount": 10},
        {"region": "north", "bucket": "Other", "amount": 2},
        {"region": "north", "bucket": "known-b", "amount": 3},
        {"region": "south", "bucket": "known-a", "amount": 4},
        {"region": "south", "bucket": "known-b", "amount": 6},
    ],
    True,
    np.dtype("int64"),
    True,
    True,
    0,
)
```

## Trade-offs and Limitations

Validation and construction are linear in the bounded cell count. The function
copies every output column and keeps per-group totals, sums, and detail-label
sets in memory. The fixed cell limit does not measure the full pandas object
overhead, so applications with tighter budgets should impose a lower limit
before materializing the frame.

The label and dtype restrictions deliberately exclude object columns,
categoricals, nullable integers, non-native arrays, custom indexes, and active
extension types. Labels and groups use exact, case-sensitive equality. The
function preserves the input column order, `RangeIndex`, and row order, but it
does not sort groups, combine duplicate details, repair a bad total, aggregate
across frames, or apply persistence policy. Concurrent mutation of the caller's
frame is outside the contract. Shape and dtype mistakes raise `TypeError` or
`ValueError`; arithmetic outside exact native `int64` raises `OverflowError`.

## Related Snippets

<!-- catalog:related:start -->
- [Aggregate pandas Groups into Explicit Flat Columns](aggregate-pandas-groups-into-explicit-flat-columns.md)
- [Build a Bounded Structural Profile of a pandas DataFrame](build-a-bounded-structural-profile-of-a-pandas-dataframe.md)
- [Apportion a Non-Negative Integer Total Without Rounding Drift](../algorithms-data-structures/apportion-a-non-negative-integer-total-without-rounding-drift.md)
<!-- catalog:related:end -->
