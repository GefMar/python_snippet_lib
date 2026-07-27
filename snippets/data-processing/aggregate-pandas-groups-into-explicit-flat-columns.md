---
title: "Aggregate pandas Groups into Explicit Flat Columns"
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
    version: "3.0.3"
related:
  - audit-candidate-pandas-downcasts-before-applying-them.md
  - enforce-a-many-to-one-pandas-left-merge-contract.md
  - select-one-record-per-key-with-an-explicit-ranking-rule.md
---

# Aggregate pandas Groups into Explicit Flat Columns

## Idea and Problem

Produce one row per observed pandas group with caller-chosen flat output names and a deliberately small aggregation vocabulary.

Each immutable specification names its source column, output column, and one
built-in operation. Pandas named aggregation avoids hierarchical column labels,
while an internal first-position aggregation makes group order explicit even
after aggregation. Missing group keys are retained, and unobserved categorical
levels do not create synthetic rows.

## When to Use

Use this integration when a bounded frame is already in memory and a report or
interchange table needs exactly one row per group. The caller must name between
one and eight group columns and between one and sixty-four aggregations. Source
dtypes must support their selected pandas operations.

The operation allowlist is intentionally limited to `count`, `max`, `mean`,
`min`, and `sum`. It prevents arbitrary aggregation callbacks from hiding I/O,
mutation, or unbounded work. Build a separate reviewed function when a domain
requires weighted statistics, quantiles, ordered collections, or custom null
semantics. Group keys and aggregation sources must use native NumPy numeric or
Boolean dtypes, exact pandas string dtypes, or exact text-category dtypes;
object payloads, third-party extension dtypes, and arbitrary category values
are rejected before pandas grouping begins. Text categories are also capped at
ten thousand declared values, including unused categories. Input frame attrs
must be empty so column selection cannot propagate active metadata.

## Implementation

```python
from dataclasses import dataclass

import numpy as np
import pandas as pd


_MAX_FRAME_ROWS = 100_000
_MAX_FRAME_COLUMNS = 256
_MAX_GROUP_COLUMNS = 8
_MAX_AGGREGATIONS = 64
_MAX_COLUMN_CHARS = 256
_MAX_TEXT_VALUE_CHARS = 4_096
_MAX_TEXT_CATEGORIES = 10_000
_ALLOWED_OPERATIONS = frozenset({"count", "max", "mean", "min", "sum"})


@dataclass(frozen=True, slots=True)
class AggregationSpec:
    output: str
    source: str
    operation: str


def _column_name(value: object, *, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if not value or len(value) > _MAX_COLUMN_CHARS:
        raise ValueError(f"{field} length is outside the supported range")
    if value != value.strip() or not value.isprintable():
        raise ValueError(f"{field} must be stripped printable text")
    return value


def _frame_columns(frame: object) -> tuple[str, ...]:
    if type(frame) is not pd.DataFrame:
        raise TypeError("frame must be an exact pandas DataFrame")
    if frame.attrs:
        raise ValueError("frame attrs must be empty")
    if len(frame) > _MAX_FRAME_ROWS:
        raise ValueError("frame exceeds the supported row limit")
    columns = tuple(
        _column_name(column, field="frame column") for column in frame.columns
    )
    if len(columns) > _MAX_FRAME_COLUMNS:
        raise ValueError("frame exceeds the supported column limit")
    if len(set(columns)) != len(columns):
        raise ValueError("frame column names must be unique")
    return columns


def _group_columns(
    values: object,
    *,
    known_columns: tuple[str, ...],
) -> tuple[str, ...]:
    if type(values) is not tuple:
        raise TypeError("group_columns must be an exact tuple")
    if not 1 <= len(values) <= _MAX_GROUP_COLUMNS:
        raise ValueError("group column count is outside the supported range")
    names = tuple(
        _column_name(value, field="group column") for value in values
    )
    if len(set(names)) != len(names):
        raise ValueError("group columns must be unique")
    if not set(names).issubset(known_columns):
        raise ValueError("every group column must exist in the frame")
    return names


def _aggregation_specs(
    values: object,
    *,
    known_columns: tuple[str, ...],
    group_columns: tuple[str, ...],
) -> tuple[AggregationSpec, ...]:
    if type(values) is not tuple:
        raise TypeError("aggregations must be an exact tuple")
    if not 1 <= len(values) <= _MAX_AGGREGATIONS:
        raise ValueError("aggregation count is outside the supported range")

    specs = []
    for value in values:
        if type(value) is not AggregationSpec:
            raise TypeError("aggregations must contain AggregationSpec values")
        output = _column_name(value.output, field="aggregation output")
        source = _column_name(value.source, field="aggregation source")
        if source not in known_columns:
            raise ValueError("every aggregation source must exist in the frame")
        if type(value.operation) is not str:
            raise TypeError("aggregation operations must be exact strings")
        if value.operation not in _ALLOWED_OPERATIONS:
            raise ValueError("an aggregation operation is not allowed")
        specs.append(AggregationSpec(output, source, value.operation))

    outputs = tuple(spec.output for spec in specs)
    if len(set(outputs)) != len(outputs):
        raise ValueError("aggregation output names must be unique")
    if set(outputs).intersection(group_columns):
        raise ValueError("aggregation outputs must not replace group columns")
    return tuple(specs)


def _private_position_column(occupied: set[str]) -> str:
    candidate = "__first_input_position__"
    while candidate in occupied:
        candidate = f"_{candidate}"
    return candidate


def _passive_column_kind(series: pd.Series, *, field: str) -> str:
    dtype = series.dtype
    if type(dtype) is pd.CategoricalDtype:
        if len(series.cat.categories) > _MAX_TEXT_CATEGORIES:
            raise ValueError(f"{field} category count exceeds the supported limit")
        for category in series.cat.categories:
            if type(category) is not str:
                raise TypeError(f"{field} categories must contain exact text")
            if not category or len(category) > _MAX_TEXT_VALUE_CHARS:
                raise ValueError(f"{field} category text is outside the limit")
        return "categorical-text"
    if type(dtype) is pd.StringDtype:
        for value in series.array:
            if value is pd.NA or (type(value) is float and value != value):
                continue
            if type(value) is not str:
                raise TypeError(f"{field} must contain exact text")
            if len(value) > _MAX_TEXT_VALUE_CHARS:
                raise ValueError(f"{field} text exceeds the supported limit")
        return "text"
    if isinstance(dtype, np.dtype):
        if not dtype.isnative:
            raise TypeError(f"{field} must use a native NumPy dtype")
        if dtype.kind == "b":
            return "boolean"
        if dtype.kind in {"i", "u"}:
            return "integer"
        if dtype.kind == "f":
            return "float"
    raise TypeError(
        f"{field} must use a passive Boolean, integer, float, string, "
        "or text-category dtype"
    )


def _validate_operation_source(
    series: pd.Series,
    *,
    operation: str,
    field: str,
) -> None:
    kind = _passive_column_kind(series, field=field)
    if operation in {"sum", "mean"} and kind not in {"integer", "float"}:
        raise TypeError(f"{operation} requires an integer or float source")
    if operation in {"min", "max"} and kind not in {
        "integer",
        "float",
        "text",
    }:
        raise TypeError(f"{operation} requires an integer, float, or text source")


def aggregate_groups(
    frame: pd.DataFrame,
    *,
    group_columns: tuple[str, ...],
    aggregations: tuple[AggregationSpec, ...],
) -> pd.DataFrame:
    columns = _frame_columns(frame)
    groups = _group_columns(group_columns, known_columns=columns)
    specs = _aggregation_specs(
        aggregations,
        known_columns=columns,
        group_columns=groups,
    )
    for group in groups:
        _passive_column_kind(frame[group], field=f"group column {group!r}")
    for spec in specs:
        _validate_operation_source(
            frame[spec.source],
            operation=spec.operation,
            field=f"aggregation source {spec.source!r}",
        )

    output_names = tuple(spec.output for spec in specs)
    position_column = _private_position_column(
        set(columns).union(output_names)
    )
    needed_columns = tuple(
        dict.fromkeys((*groups, *(spec.source for spec in specs)))
    )
    working = pd.DataFrame(
        {
            column: frame[column].array.copy()
            for column in needed_columns
        },
        copy=True,
    )
    working[position_column] = range(len(working))

    named_aggregations = {
        spec.output: pd.NamedAgg(column=spec.source, aggfunc=spec.operation)
        for spec in specs
    }
    named_aggregations[position_column] = pd.NamedAgg(
        column=position_column,
        aggfunc="min",
    )

    grouped = working.groupby(
        list(groups),
        as_index=False,
        sort=False,
        observed=True,
        dropna=False,
    )
    result = grouped.agg(**named_aggregations)
    result = result.sort_values(position_column, kind="stable")
    return result.loc[:, [*groups, *output_names]].reset_index(drop=True)
```

## Example

```python
frame = pd.DataFrame(
    {
        "region": pd.Categorical(
            ["west", None, "west", "east", None],
            categories=["east", "west", "unused"],
        ),
        "amount": [2, 4, 3, 8, 6],
        "note": ["a", None, "b", "c", "d"],
    }
)
before = frame.copy(deep=True)

summary = aggregate_groups(
    frame,
    group_columns=("region",),
    aggregations=(
        AggregationSpec("total_amount", "amount", "sum"),
        AggregationSpec("mean_amount", "amount", "mean"),
        AggregationSpec("present_notes", "note", "count"),
    ),
)

assert (
    tuple(summary.columns),
    summary["region"].astype("object").fillna("<missing>").tolist(),
    summary["total_amount"].tolist(),
    summary["mean_amount"].tolist(),
    summary["present_notes"].tolist(),
    "unused" not in summary["region"].astype("object").tolist(),
    frame.equals(before),
) == (
    ("region", "total_amount", "mean_amount", "present_notes"),
    ["west", "<missing>", "east"],
    [5, 10, 8],
    [2.5, 5.0, 8.0],
    [2, 1, 1],
    True,
    True,
)
```

## Trade-offs and Limitations

The function copies only the required bounded column arrays into a fresh
RangeIndex frame, performs pandas grouping, and stores the complete result in
memory. Input frame attrs must be empty; the caller's index and Series
attributes are deliberately not carried into the aggregate. Object-dtype values,
non-text categories, datetimes, timedeltas, and extension types outside the
explicit passive allowlist are rejected. A selected pandas operation can still
fail for a supported dtype; such failures propagate without exposing the
caller frame to user-defined hashing, comparison, or arithmetic methods.

`count` excludes missing source values, while the other operations follow the
corresponding pandas null and dtype semantics. Those details may differ from a
database or another dataframe engine. First appearance is defined by the
smallest input position in each observed group; it is not a sorted business
order. The output intentionally collapses every group to one row, so it cannot
represent list-valued detail, rolling windows, transform-sized results, or
cross-group calculations.

## Related Snippets

<!-- catalog:related:start -->
- [Audit Candidate pandas Downcasts Before Applying Them](audit-candidate-pandas-downcasts-before-applying-them.md)
- [Enforce a Many-to-One pandas Left-Merge Contract](enforce-a-many-to-one-pandas-left-merge-contract.md)
- [Select One Record per Key with an Explicit Ranking Rule](select-one-record-per-key-with-an-explicit-ranking-rule.md)
<!-- catalog:related:end -->
