---
title: "Create Past-Only pandas Lag and Rolling-Mean Columns"
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
  - detect-a-recent-drop-against-a-disjoint-pandas-baseline-window.md
  - flag-groupwise-numeric-outliers-with-iqr-fences.md
  - ../data-processing/aggregate-pandas-groups-into-explicit-flat-columns.md
---

# Create Past-Only pandas Lag and Rolling-Mean Columns

## Idea and Problem

Add deterministic lag and complete rolling-mean features that use only earlier rows from the same contiguous group.

For a row at position `i` within a group, lag `k` reads position `i - k`.
A rolling window `w` averages positions `[i - w, i)`, explicitly excluding
the current row. Positions without complete history remain `NaN`. The function
never sorts, fills, resamples, or crosses a group boundary, so input order is
also the feature-evaluation order.

## When to Use

Use this integration after establishing a bounded, already ordered feature
table. It accepts exactly one Python-backed string group column, one native
timezone-naive `datetime64[ns]` timestamp column, and one finite native
`float64` value column. Each group must occupy one contiguous block, and its
timestamps must increase strictly. Lag and window tuples are immutable,
positive, unique, and interpreted in rows rather than elapsed time.

This is suitable for preparing simple historical predictors when row spacing
is already meaningful. Split training and evaluation data under a reviewed
policy before fitting any model. The helper does not sort, resample,
interpolate, fit a model, or protect against leakage introduced elsewhere in a
pipeline.

## Implementation

```python
import math

import numpy as np
import pandas as pd


_MAX_ROWS = 100_000
_MAX_GROUPS = 10_000
_MAX_GROUP_CHARACTERS = 4_000_000
_MAX_FEATURES = 32
_MAX_LOOKBACK = 10_000
_MAX_OUTPUT_CELLS = 2_000_000
_MAX_FEATURE_WORK = 25_000_000
_MAX_COLUMN_CHARACTERS = 128
_MAX_GROUP_LABEL_CHARACTERS = 4_096


def _bounded_name(value: object, *, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if not value or len(value) > _MAX_COLUMN_CHARACTERS:
        raise ValueError(f"{field} length is outside the supported range")
    if value != value.strip() or not value.isprintable():
        raise ValueError(f"{field} must be stripped printable text")
    return value


def _steps(value: object, *, field: str) -> tuple[int, ...]:
    if type(value) is not tuple:
        raise TypeError(f"{field} must be an exact tuple")
    if len(value) > _MAX_FEATURES:
        raise ValueError(f"{field} contains too many entries")
    result: list[int] = []
    for step in value:
        if type(step) is not int:
            raise TypeError(f"{field} entries must be exact integers")
        if not 1 <= step <= _MAX_LOOKBACK:
            raise ValueError(f"{field} entries are outside the supported range")
        result.append(step)
    if len(set(result)) != len(result):
        raise ValueError(f"{field} entries must be unique")
    return tuple(result)


def _frame_columns(
    frame: object,
    *,
    declared: tuple[str, str, str],
) -> tuple[str, ...]:
    if type(frame) is not pd.DataFrame:
        raise TypeError("frame must be an exact pandas DataFrame")
    if frame.attrs:
        raise ValueError("frame attrs must be empty")
    if len(frame) > _MAX_ROWS:
        raise ValueError("frame exceeds the supported row limit")
    if type(frame.index) is not pd.RangeIndex:
        raise TypeError("frame must use an exact RangeIndex")
    if frame.index.name is not None:
        _bounded_name(frame.index.name, field="index name")
    if type(frame.columns) is not pd.Index:
        raise TypeError("frame columns must use an exact pandas Index")

    columns = tuple(
        _bounded_name(column, field="frame column") for column in frame.columns
    )
    if len(columns) != 3 or len(set(columns)) != 3:
        raise ValueError("frame must contain exactly three unique columns")
    if set(columns) != set(declared):
        raise ValueError("frame columns must exactly match the declaration")
    return columns


def _group_values(series: pd.Series) -> tuple[str, ...]:
    dtype = series.dtype
    if type(dtype) is not pd.StringDtype or dtype.storage != "python":
        raise TypeError("group column must use Python-backed pandas string dtype")
    if type(series.array) is not pd.arrays.StringArray:
        raise TypeError("group column array implementation is not supported")

    values: list[str] = []
    character_count = 0
    for value in series.array:
        if type(value) is not str:
            raise ValueError("group labels must not be missing")
        if (
            not value
            or len(value) > _MAX_GROUP_LABEL_CHARACTERS
            or value != value.strip()
            or not value.isprintable()
        ):
            raise ValueError("group labels must be bounded stripped printable text")
        character_count += len(value)
        if character_count > _MAX_GROUP_CHARACTERS:
            raise ValueError("group labels exceed the cumulative character limit")
        values.append(value)
    return tuple(values)


def _timestamp_values(series: pd.Series) -> np.ndarray:
    dtype = series.dtype
    if not (
        isinstance(dtype, np.dtype)
        and dtype.metadata is None
        and dtype == np.dtype("datetime64[ns]")
        and dtype.isnative
    ):
        raise TypeError("timestamp column must use native datetime64[ns]")
    if type(series.array) is not pd.arrays.DatetimeArray:
        raise TypeError("timestamp column array implementation is not supported")
    values = series.array.asi8
    if bool(np.any(values == np.iinfo(np.int64).min)):
        raise ValueError("timestamp values must not contain NaT")
    return values


def _float64_values(series: pd.Series) -> np.ndarray:
    dtype = series.dtype
    if not (
        isinstance(dtype, np.dtype)
        and dtype.metadata is None
        and dtype == np.dtype("float64")
        and dtype.isnative
    ):
        raise TypeError("value column must use native float64")
    if type(series.array) is not pd.arrays.NumpyExtensionArray:
        raise TypeError("value column array implementation is not supported")
    values = np.asarray(series.array, dtype=np.float64)
    if not bool(np.isfinite(values).all()):
        raise ValueError("value column must contain only finite values")
    return values


def _stable_mean(values: np.ndarray) -> float:
    scale = max(abs(float(value)) for value in values)
    if scale == 0.0:
        return 0.0
    mean = math.fsum(float(value) / scale for value in values) / len(values)
    result = mean * scale
    if not math.isfinite(result):
        raise ArithmeticError("a rolling mean is outside the finite-float range")
    return result


def create_past_only_features(
    frame: pd.DataFrame,
    *,
    group_column: str,
    timestamp_column: str,
    value_column: str,
    lags: tuple[int, ...],
    windows: tuple[int, ...],
) -> pd.DataFrame:
    group_name = _bounded_name(group_column, field="group_column")
    timestamp_name = _bounded_name(timestamp_column, field="timestamp_column")
    value_name = _bounded_name(value_column, field="value_column")
    declared = (group_name, timestamp_name, value_name)
    if len(set(declared)) != 3:
        raise ValueError("declared column names must be unique")

    lag_steps = _steps(lags, field="lags")
    window_steps = _steps(windows, field="windows")
    feature_count = len(lag_steps) + len(window_steps)
    if not 1 <= feature_count <= _MAX_FEATURES:
        raise ValueError("request must contain a supported number of features")

    columns = _frame_columns(frame, declared=declared)
    row_count = len(frame)
    if row_count * (len(columns) + feature_count) > _MAX_OUTPUT_CELLS:
        raise ValueError("requested output exceeds the supported cell limit")

    generated = tuple(
        [f"{value_name}_lag_{step}" for step in lag_steps]
        + [f"{value_name}_rolling_mean_{step}" for step in window_steps]
    )
    for name in generated:
        _bounded_name(name, field="generated column")
    if len(set(generated)) != len(generated) or set(generated) & set(columns):
        raise ValueError("generated column names must be unique and new")

    groups = _group_values(frame[group_name])
    timestamps = _timestamp_values(frame[timestamp_name])
    values = _float64_values(frame[value_name])

    segments: list[tuple[int, int]] = []
    closed_groups: set[str] = set()
    if row_count:
        start = 0
        for position in range(1, row_count):
            if groups[position] == groups[position - 1]:
                if timestamps[position] <= timestamps[position - 1]:
                    raise ValueError("timestamps must increase strictly within a group")
            else:
                closed_groups.add(groups[position - 1])
                if groups[position] in closed_groups:
                    raise ValueError("each group must occupy one contiguous block")
                segments.append((start, position))
                if len(segments) >= _MAX_GROUPS:
                    raise ValueError("frame exceeds the supported group limit")
                start = position
        segments.append((start, row_count))
        if len(segments) > _MAX_GROUPS:
            raise ValueError("frame exceeds the supported group limit")

    feature_work = 0
    for start, end in segments:
        length = end - start
        feature_work += sum(max(0, length - lag) for lag in lag_steps)
        feature_work += sum(
            max(0, length - window) * window for window in window_steps
        )
        if feature_work > _MAX_FEATURE_WORK:
            raise ValueError("requested features exceed the supported work limit")

    output = frame.copy(deep=True)
    for lag, name in zip(lag_steps, generated[: len(lag_steps)], strict=True):
        lagged = np.full(row_count, np.nan, dtype=np.float64)
        for start, end in segments:
            if end - start > lag:
                lagged[start + lag : end] = values[start : end - lag]
        output[name] = lagged

    rolling_names = generated[len(lag_steps) :]
    for window, name in zip(window_steps, rolling_names, strict=True):
        means = np.full(row_count, np.nan, dtype=np.float64)
        for start, end in segments:
            for position in range(start + window, end):
                means[position] = _stable_mean(values[position - window : position])
        output[name] = means
    return output
```

## Example

```python
text_dtype = pd.StringDtype(storage="python", na_value=pd.NA)
frame = pd.DataFrame(
    {
        "group": pd.array(["a", "a", "a", "b", "b", "b", "b"], dtype=text_dtype),
        "observed_at": pd.array(
            [
                "2026-01-01", "2026-01-02", "2026-01-03",
                "2026-02-01", "2026-02-02", "2026-02-03", "2026-02-04",
            ],
            dtype="datetime64[ns]",
        ),
        "metric": np.array([10.0, 20.0, 30.0, 100.0, 110.0, 90.0, 130.0]),
    }
)
before = frame.copy(deep=True)
featured = create_past_only_features(
    frame,
    group_column="group",
    timestamp_column="observed_at",
    value_column="metric",
    lags=(1, 2),
    windows=(2,),
)

future_changed = frame.copy(deep=True)
future_changed.loc[6, "metric"] = 999.0
changed_features = create_past_only_features(
    future_changed,
    group_column="group",
    timestamp_column="observed_at",
    value_column="metric",
    lags=(1, 2),
    windows=(2,),
)
feature_columns = ["metric_lag_1", "metric_lag_2", "metric_rolling_mean_2"]
expected = np.array(
    [
        [np.nan, np.nan, np.nan],
        [10.0, np.nan, np.nan],
        [20.0, 10.0, 15.0],
        [np.nan, np.nan, np.nan],
        [100.0, np.nan, np.nan],
        [110.0, 100.0, 105.0],
        [90.0, 110.0, 100.0],
    ]
)

assert (
    np.allclose(featured[feature_columns].to_numpy(), expected, equal_nan=True),
    np.allclose(
        featured[feature_columns].to_numpy(),
        changed_features[feature_columns].to_numpy(),
        equal_nan=True,
    ),
    featured.columns.tolist(),
    all(featured[name].dtype == np.dtype("float64") for name in feature_columns),
    frame.equals(before),
) == (
    True,
    True,
    ["group", "observed_at", "metric", *feature_columns],
    True,
    True,
)
```

## Trade-offs and Limitations

Lag construction is linear in the produced lag cells. Each complete rolling
window is rescanned to calculate a scaled `math.fsum`, which avoids overflow in
ordinary finite means but costs `O(rows * window)` time. The explicit work cap
rejects large combinations before feature arrays are allocated. Output cells,
group labels, group count, feature count, and maximum lookback are bounded
separately; these limits still do not model every pandas object-overhead byte.

Groups use exact case-sensitive labels, timestamps are timezone-naive, and
windows count rows rather than durations. Input rows must already have the
correct contiguous order; silently sorting could conceal an upstream error and
change the meaning of a previous row. Generated `float64` columns intentionally
contain leading `NaN` where history is incomplete. The returned frame is a
caller-owned copy, while concurrent mutation of the input during validation is
outside the contract.

## Related Snippets

<!-- catalog:related:start -->
- [Detect a Recent Drop Against a Disjoint pandas Baseline Window](detect-a-recent-drop-against-a-disjoint-pandas-baseline-window.md)
- [Flag Groupwise Numeric Outliers with IQR Fences](flag-groupwise-numeric-outliers-with-iqr-fences.md)
- [Aggregate pandas Groups into Explicit Flat Columns](../data-processing/aggregate-pandas-groups-into-explicit-flat-columns.md)
<!-- catalog:related:end -->
