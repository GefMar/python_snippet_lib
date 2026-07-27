---
title: "Audit Candidate pandas Downcasts Before Applying Them"
snippet_type: integration
use_cases:
  - data-transformation
  - performance-optimization
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies:
  - name: numpy
    version: "2.5.1"
  - name: pandas
    version: "3.0.3"
related:
  - audit-pandas-missing-value-shares-against-column-policies.md
  - aggregate-pandas-groups-into-explicit-flat-columns.md
  - ../python-language/make-a-defensive-copy-at-a-mutable-input-boundary.md
---

# Audit Candidate pandas Downcasts Before Applying Them

## Idea and Problem

Measure a small set of lossless pandas downcast candidates and apply one only when its observed deep column memory is strictly smaller.

For native NumPy integers and floats, the implementation tries narrower dtypes
from smallest to largest. For object or pandas string columns containing only
text and missing values, it tries a categorical representation. Every
candidate is converted back to the original dtype before acceptance: missing
positions must be identical and all remaining restored values must compare
equal. This deliberately treats all missing representations as equivalent and
treats positive and negative floating-point zero as equal.

## When to Use

Use this integration for a bounded in-memory frame when reducing its measured
footprint is useful but silent numeric coercion is unacceptable. It is a good
preflight before retaining a frame for repeated analysis. The function copies
the frame, visits columns in input order, and returns an immutable decision for
every column, including columns for which no supported candidate exists.

The text path never tries to interpret object strings as numbers. The accepted
boundary contains native NumPy columns plus object or pandas string columns
whose values are bounded exact strings or passive missing sentinels. Other
pandas extension dtypes, existing categoricals, non-native NumPy dtypes, and
arbitrary Python objects are rejected before copying or deep measurement. Add
separately reviewed policies when those types need conversion rather than
broadening this conservative boundary.

## Implementation

```python
from dataclasses import dataclass

import numpy as np
import pandas as pd


_MAX_FRAME_ROWS = 100_000
_MAX_FRAME_COLUMNS = 256
_MAX_COLUMN_CHARS = 256
_MAX_TEXT_CHARS = 4_096
_INTEGER_TARGETS = {
    "i": tuple(np.dtype(name) for name in ("int8", "int16", "int32", "int64")),
    "u": tuple(
        np.dtype(name) for name in ("uint8", "uint16", "uint32", "uint64")
    ),
}
_FLOAT_TARGETS = tuple(
    np.dtype(name) for name in ("float16", "float32", "float64")
)


@dataclass(frozen=True, slots=True)
class DowncastDecision:
    column: str
    before_dtype: str
    candidate_dtype: str | None
    after_dtype: str
    before_bytes: int
    candidate_bytes: int | None
    after_bytes: int
    applied: bool
    reason: str


def _column_name(value: object) -> str:
    if type(value) is not str:
        raise TypeError("column names must be exact strings")
    if not value or len(value) > _MAX_COLUMN_CHARS:
        raise ValueError("column name length is outside the supported range")
    if value != value.strip() or not value.isprintable():
        raise ValueError("column names must be stripped printable text")
    return value


def _frame_columns(frame: object) -> tuple[str, ...]:
    if type(frame) is not pd.DataFrame:
        raise TypeError("frame must be an exact pandas DataFrame")
    if frame.attrs:
        raise ValueError("frame attrs must be empty")
    if len(frame) > _MAX_FRAME_ROWS:
        raise ValueError("frame exceeds the supported row limit")
    columns = tuple(_column_name(column) for column in frame.columns)
    if len(columns) > _MAX_FRAME_COLUMNS:
        raise ValueError("frame exceeds the supported column limit")
    if len(set(columns)) != len(columns):
        raise ValueError("frame column names must be unique")
    return columns


def _is_text_missing(value: object) -> bool:
    return (
        value is None
        or value is pd.NA
        or value is pd.NaT
        or (type(value) is float and value != value)
    )


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
    if isinstance(dtype, np.dtype):
        if not dtype.isnative:
            raise TypeError("non-native index dtypes are not supported")
        if dtype.kind != "O":
            return
    elif type(dtype) is not pd.StringDtype:
        raise TypeError("frame index dtype is outside the passive boundary")
    for label in index:
        _passive_scalar(label, field="index label")


def _validate_passive_column(series: pd.Series) -> None:
    dtype = series.dtype
    if isinstance(dtype, np.dtype):
        if not dtype.isnative:
            raise TypeError("non-native NumPy dtypes are not supported")
        if dtype.kind != "O":
            return
    elif type(dtype) is not pd.StringDtype:
        raise TypeError("only native NumPy and pandas string dtypes are supported")

    for value in series.array:
        if _is_text_missing(value):
            continue
        if type(value) is not str:
            raise TypeError("object columns may contain only exact strings or missing values")
        if len(value) > _MAX_TEXT_CHARS:
            raise ValueError("text value exceeds the supported length")


def _same_values_and_missing(
    original: pd.Series,
    restored: pd.Series,
) -> bool:
    original_missing = original.isna()
    restored_missing = restored.isna()
    if not original_missing.equals(restored_missing):
        return False
    positions = (~original_missing).to_numpy(dtype=bool)
    return original.iloc[positions].equals(restored.iloc[positions])


def _numeric_targets(series: pd.Series) -> tuple[np.dtype, ...]:
    dtype = series.dtype
    if not isinstance(dtype, np.dtype) or not dtype.isnative:
        return ()
    if dtype.kind in _INTEGER_TARGETS:
        targets = tuple(
            candidate
            for candidate in _INTEGER_TARGETS[dtype.kind]
            if candidate.itemsize < dtype.itemsize
        )
        if series.empty:
            return targets
        minimum = int(series.min())
        maximum = int(series.max())
        return tuple(
            candidate
            for candidate in targets
            if np.iinfo(candidate).min <= minimum
            and maximum <= np.iinfo(candidate).max
        )
    if dtype.kind == "f":
        return tuple(
            candidate
            for candidate in _FLOAT_TARGETS
            if candidate.itemsize < dtype.itemsize
        )
    return ()


def _text_candidate(series: pd.Series) -> pd.Series | None:
    dtype = series.dtype
    if dtype != object and not isinstance(dtype, pd.StringDtype):
        return None
    non_missing = series.loc[~series.isna()]
    if non_missing.empty or any(type(value) is not str for value in non_missing):
        return None
    return series.astype("category", copy=True)


def _candidate(
    series: pd.Series,
) -> tuple[pd.Series | None, bool, str]:
    targets = _numeric_targets(series)
    first_numeric: pd.Series | None = None
    for target in targets:
        with np.errstate(over="ignore", invalid="ignore"):
            converted = series.astype(target, copy=True)
            restored = converted.astype(series.dtype, copy=True)
        if first_numeric is None:
            first_numeric = converted
        if _same_values_and_missing(series, restored):
            return converted, True, "candidate_round_trip_is_equivalent"
    if first_numeric is not None:
        return first_numeric, False, "candidate_round_trip_changed_values"

    text = _text_candidate(series)
    if text is None:
        return None, False, "already_narrow_or_unsupported_dtype"
    restored = text.astype(series.dtype, copy=True)
    return (
        text,
        _same_values_and_missing(series, restored),
        "candidate_round_trip_is_equivalent",
    )


def audit_and_apply_downcasts(
    frame: pd.DataFrame,
) -> tuple[pd.DataFrame, tuple[DowncastDecision, ...]]:
    columns = _frame_columns(frame)
    _validate_passive_index(frame.index)
    for column in columns:
        _validate_passive_column(frame[column])
    result = pd.DataFrame(
        {
            column: frame[column].array.copy()
            for column in columns
        },
        index=frame.index.copy(deep=True),
        copy=True,
    )
    decisions = []

    for column in columns:
        original = frame[column]
        before_dtype = str(original.dtype)
        before_bytes = int(original.memory_usage(index=False, deep=True))
        candidate, equivalent, candidate_reason = _candidate(original)

        if candidate is None:
            decisions.append(
                DowncastDecision(
                    column=column,
                    before_dtype=before_dtype,
                    candidate_dtype=None,
                    after_dtype=before_dtype,
                    before_bytes=before_bytes,
                    candidate_bytes=None,
                    after_bytes=before_bytes,
                    applied=False,
                    reason=candidate_reason,
                )
            )
            continue

        candidate_dtype = str(candidate.dtype)
        candidate_bytes = int(candidate.memory_usage(index=False, deep=True))
        if not equivalent:
            reason = "candidate_round_trip_changed_values"
            applied = False
        elif candidate_bytes >= before_bytes:
            reason = "candidate_memory_is_not_smaller"
            applied = False
        else:
            result[column] = candidate
            reason = "smaller_equivalent_candidate_applied"
            applied = True

        decisions.append(
            DowncastDecision(
                column=column,
                before_dtype=before_dtype,
                candidate_dtype=candidate_dtype,
                after_dtype=str(result[column].dtype),
                before_bytes=before_bytes,
                candidate_bytes=candidate_bytes,
                after_bytes=int(
                    result[column].memory_usage(index=False, deep=True)
                ),
                applied=applied,
                reason=reason,
            )
        )

    return result, tuple(decisions)
```

## Example

```python
row_count = 128
frame = pd.DataFrame(
    {
        "small_count": list(range(row_count)),
        "ratio": [1.5, 2.0] * (row_count // 2),
        "label": ["long-alpha", "long-beta"] * (row_count // 2),
        "precise": [0.1] * row_count,
    }
)
frame["label"] = frame["label"].astype("object")
before = frame.copy(deep=True)

compact, decisions = audit_and_apply_downcasts(frame)
by_column = {decision.column: decision for decision in decisions}

assert (
    tuple(decision.column for decision in decisions),
    tuple(by_column[name].applied for name in frame.columns),
    tuple(str(compact[name].dtype) for name in frame.columns),
    by_column["precise"].reason,
    all(
        decision.after_bytes < decision.before_bytes
        for decision in decisions
        if decision.applied
    ),
    compact.astype(frame.dtypes.to_dict()).equals(frame),
    frame.equals(before),
    compact is not frame,
) == (
    ("small_count", "ratio", "label", "precise"),
    (True, True, True, False),
    ("int8", "float16", "category", "float64"),
    "candidate_round_trip_changed_values",
    True,
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

Each candidate requires at least one conversion, a full equivalence scan, and
a deep memory measurement. Category measurement can traverse every text
value, while conversions temporarily retain both representations. Input attrs
must be empty; the result is rebuilt from validated arrays and a passive index
rather than inheriting frame metadata. Use an out-of-core profiler when the
frame does not fit the stated limits.

Deep byte counts depend on pandas, NumPy, the Python allocator, string storage,
and platform details; they are observations for this frame, not portable
memory guarantees. Exact float round trips reject useful but
precision-losing conversions, and the equivalence policy does not preserve NaN
payloads or the sign bit of zero. Categories help mainly when values repeat and
can make later updates or interchange less convenient. Arbitrary object
payloads are intentionally rejected instead of being passed through an
ownership boundary that cannot recursively isolate them.

## Related Snippets

<!-- catalog:related:start -->
- [Audit pandas Missing-Value Shares Against Column Policies](audit-pandas-missing-value-shares-against-column-policies.md)
- [Aggregate pandas Groups into Explicit Flat Columns](aggregate-pandas-groups-into-explicit-flat-columns.md)
- [Make a Defensive Copy at a Mutable Input Boundary](../python-language/make-a-defensive-copy-at-a-mutable-input-boundary.md)
<!-- catalog:related:end -->
