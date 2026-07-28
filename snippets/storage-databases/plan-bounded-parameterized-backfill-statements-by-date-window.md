---
title: "Plan Bounded Parameterized Backfill Statements by Date Window"
snippet_type: pattern
use_cases:
  - automation
  - persistence
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - plan-an-additive-sqlite-column-projection.md
  - project-a-dataclass-into-a-validated-insert-row.md
  - split-a-half-open-utc-range-across-ordered-storage-tiers.md
---

# Plan Bounded Parameterized Backfill Statements by Date Window

## Idea and Problem

Turn a finite half-open date range and a closed set of reviewed statements into immutable, date-major execution inputs without changing any SQL text.

The important boundary is between SQL and data. Each statement's text remains
exactly as supplied, while its fixed bound values are copied and combined with
two reserved date parameters. Window-by-statement expansion is secondary: the
planner first proves that the range, definitions, payload sizes, and complete
Cartesian product fit explicit limits, then allocates the result.

## When to Use

Use this pattern after statement owners have fixed and reviewed the SQL and
chosen named DB-API parameters for one backfill operation. It is useful when a
separate executor needs deterministic `[start, end)` slices in caller-supplied
statement order, but must never receive SQL with values rendered into it.

This planner deliberately does not discover SQL or identifiers. Keep statement
files, query generation, database access, transaction policy, retries, and
completion tracking in other components.

## Implementation

```python
import math
import re
from collections.abc import Mapping
from dataclasses import dataclass
from datetime import date, timedelta
from types import MappingProxyType

_MAX_RANGE_DAYS = 3_720
_MAX_STATEMENTS = 24
_MAX_PARAMETERS_PER_STATEMENT = 24
_MAX_VALUE_BYTES = 8_192
_MAX_SQL_BYTES = 96 * 1_024
_MAX_PARAMETER_BYTES = 3 * 1_024 * 1_024
_MAX_PLAN_ITEMS = 6_000
_MAX_OUTPUT_BYTES = 12 * 1_024 * 1_024
_MAX_INTEGER = (1 << 63) - 1
_WINDOW_START_PARAMETER = "segment_open"
_WINDOW_END_PARAMETER = "segment_close"
_WINDOW_PARAMETERS = frozenset({_WINDOW_START_PARAMETER, _WINDOW_END_PARAMETER})
_STATEMENT_ID = re.compile(r"[a-z][a-z0-9_]{0,47}", re.ASCII)
_PARAMETER_NAME = re.compile(r"[A-Za-z_][A-Za-z0-9_]{0,63}", re.ASCII)

type BoundScalar = None | bool | int | float | str | bytes | date


@dataclass(frozen=True, slots=True)
class FixedStatement:
    statement_id: str
    sql: str
    fixed_parameters: dict[str, BoundScalar]


@dataclass(frozen=True, slots=True)
class BackfillStatementPlan:
    window_start: date
    window_end: date
    statement_id: str
    sql: str
    parameters: Mapping[str, BoundScalar]


@dataclass(frozen=True, slots=True)
class _CheckedStatement:
    statement_id: str
    sql: str
    fixed_parameters: tuple[tuple[str, BoundScalar], ...]
    sql_bytes: int
    parameter_bytes: int


def _utf8_size(value: str, field: str) -> int:
    try:
        return len(value.encode("utf-8"))
    except UnicodeEncodeError as error:
        raise ValueError(f"{field} must be valid UTF-8 text") from error


def _bound_value_size(value: object, field: str) -> int:
    value_type = type(value)
    if value is None:
        size = 1
    elif value_type is bool:
        size = 1
    elif value_type is int:
        if not -_MAX_INTEGER - 1 <= value <= _MAX_INTEGER:
            raise ValueError(f"{field} is outside the signed 64-bit range")
        size = len(str(value).encode("ascii"))
    elif value_type is float:
        if not math.isfinite(value):
            raise ValueError(f"{field} must be finite")
        size = len(repr(value).encode("ascii"))
    elif value_type is str:
        if len(value) > _MAX_VALUE_BYTES:
            raise ValueError(f"{field} exceeds the value byte limit")
        size = _utf8_size(value, field)
    elif value_type is bytes:
        size = len(value)
    elif value_type is date:
        size = 10
    else:
        raise TypeError(
            f"{field} must be an exact None, bool, int, float, str, bytes, or date scalar"
        )

    if size > _MAX_VALUE_BYTES:
        raise ValueError(f"{field} exceeds the value byte limit")
    return size


def _check_statement(
    definition: object,
    position: int,
) -> _CheckedStatement:
    field = f"statements[{position}]"
    if type(definition) is not FixedStatement:
        raise TypeError(f"{field} must be an exact FixedStatement")
    if type(definition.statement_id) is not str:
        raise TypeError(f"{field}.statement_id must be an exact string")
    if _STATEMENT_ID.fullmatch(definition.statement_id) is None:
        raise ValueError(f"{field}.statement_id has an invalid shape")
    if type(definition.sql) is not str:
        raise TypeError(f"{field}.sql must be an exact string")
    if not definition.sql:
        raise ValueError(f"{field}.sql must not be empty")
    if len(definition.sql) > _MAX_SQL_BYTES:
        raise ValueError(f"{field}.sql exceeds the SQL byte limit")

    sql_bytes = _utf8_size(definition.sql, f"{field}.sql")
    if sql_bytes > _MAX_SQL_BYTES:
        raise ValueError(f"{field}.sql exceeds the SQL byte limit")

    supplied = definition.fixed_parameters
    if type(supplied) is not dict:
        raise TypeError(f"{field}.fixed_parameters must be an exact dict")
    if len(supplied) > _MAX_PARAMETERS_PER_STATEMENT:
        raise ValueError(f"{field} has too many fixed parameters")

    parameter_items = tuple(supplied.items())
    parameter_bytes = 0
    for name, value in parameter_items:
        if type(name) is not str:
            raise TypeError(f"{field} parameter names must be exact strings")
        if _PARAMETER_NAME.fullmatch(name) is None:
            raise ValueError(f"{field} has an invalid parameter name")
        if name in _WINDOW_PARAMETERS:
            raise ValueError(f"{field} uses a reserved window parameter")
        parameter_bytes += _utf8_size(name, f"{field} parameter name")
        parameter_bytes += _bound_value_size(value, f"{field}[{name!r}]")

    return _CheckedStatement(
        statement_id=definition.statement_id,
        sql=definition.sql,
        fixed_parameters=parameter_items,
        sql_bytes=sql_bytes,
        parameter_bytes=parameter_bytes,
    )


def plan_backfill_statements(
    start: date,
    end: date,
    window_days: int,
    statements: tuple[FixedStatement, ...],
) -> tuple[BackfillStatementPlan, ...]:
    if type(start) is not date or type(end) is not date:
        raise TypeError("start and end must be exact date values")
    if type(window_days) is not int:
        raise TypeError("window_days must be an exact integer")
    if window_days <= 0:
        raise ValueError("window_days must be positive")
    if end <= start:
        raise ValueError("end must be later than start")

    range_days = (end - start).days
    if range_days > _MAX_RANGE_DAYS:
        raise ValueError("the requested date range exceeds the day limit")
    if type(statements) is not tuple:
        raise TypeError("statements must be an exact tuple")
    if not 1 <= len(statements) <= _MAX_STATEMENTS:
        raise ValueError("statements has an unsupported cardinality")

    checked = tuple(
        _check_statement(definition, position) for position, definition in enumerate(statements)
    )
    if len({statement.statement_id for statement in checked}) != len(checked):
        raise ValueError("statement IDs must be unique")

    window_count = (range_days + window_days - 1) // window_days
    plan_count = window_count * len(checked)
    if plan_count > _MAX_PLAN_ITEMS:
        raise ValueError("the window-by-statement plan exceeds the item limit")

    sql_bytes = sum(statement.sql_bytes for statement in checked)
    if sql_bytes > _MAX_SQL_BYTES:
        raise ValueError("the statements exceed the aggregate SQL byte limit")
    fixed_parameter_bytes = sum(statement.parameter_bytes for statement in checked)
    window_parameter_bytes = len(_WINDOW_START_PARAMETER) + len(_WINDOW_END_PARAMETER) + 20
    planned_sql_bytes = window_count * sql_bytes
    planned_parameter_bytes = window_count * (
        fixed_parameter_bytes + len(checked) * window_parameter_bytes
    )
    if planned_parameter_bytes > _MAX_PARAMETER_BYTES:
        raise ValueError("the expanded parameters exceed the byte limit")
    if planned_sql_bytes + planned_parameter_bytes > _MAX_OUTPUT_BYTES:
        raise ValueError("the expanded plan exceeds the aggregate byte limit")

    plans: list[BackfillStatementPlan] = []
    window_start = start
    while window_start < end:
        remaining_days = (end - window_start).days
        window_end = (
            end if remaining_days <= window_days else window_start + timedelta(days=window_days)
        )
        for statement in checked:
            parameters = dict(statement.fixed_parameters)
            parameters[_WINDOW_START_PARAMETER] = window_start
            parameters[_WINDOW_END_PARAMETER] = window_end
            plans.append(
                BackfillStatementPlan(
                    window_start=window_start,
                    window_end=window_end,
                    statement_id=statement.statement_id,
                    sql=statement.sql,
                    parameters=MappingProxyType(parameters),
                )
            )
        window_start = window_end

    return tuple(plans)
```

## Example

```python
from datetime import datetime

measurement_parameters = {
    "metric_family": "surface_temperature",
    "minimum_samples": 12,
}
definitions = (
    FixedStatement(
        statement_id="rescale_daily_measurements",
        sql=(
            "UPDATE daily_measurements\n"
            "SET scale_factor = :scale_factor\n"
            "WHERE observed_on >= :segment_open\n"
            "  AND observed_on < :segment_close\n"
            "  AND metric_family = :metric_family\n"
            "  AND sample_count >= :minimum_samples"
        ),
        fixed_parameters={
            **measurement_parameters,
            "scale_factor": 1.25,
        },
    ),
    FixedStatement(
        statement_id="record_window_sample_total",
        sql=(
            "INSERT INTO measurement_window_totals "
            "(window_date, metric_family, sample_total)\n"
            "SELECT :segment_open, :metric_family, SUM(sample_count)\n"
            "FROM daily_measurements\n"
            "WHERE observed_on >= :segment_open\n"
            "  AND observed_on < :segment_close\n"
            "  AND metric_family = :metric_family"
        ),
        fixed_parameters={"metric_family": "air_pressure"},
    ),
)

plans = plan_backfill_statements(
    date(2031, 4, 3),
    date(2031, 4, 10),
    3,
    definitions,
)
definitions[0].fixed_parameters["metric_family"] = "changed-later"

mutation_rejected = False
try:
    plans[0].parameters["metric_family"] = "changed-again"
except TypeError:
    mutation_rejected = True

strict_type_rejections = 0
for bad_start, bad_width in (
    (datetime(2031, 4, 3), 3),
    (date(2031, 4, 3), True),
):
    try:
        plan_backfill_statements(
            bad_start,
            date(2031, 4, 10),
            bad_width,
            definitions,
        )
    except TypeError:
        strict_type_rejections += 1

assert tuple((plan.window_start, plan.window_end, plan.statement_id) for plan in plans) == (
    (date(2031, 4, 3), date(2031, 4, 6), "rescale_daily_measurements"),
    (date(2031, 4, 3), date(2031, 4, 6), "record_window_sample_total"),
    (date(2031, 4, 6), date(2031, 4, 9), "rescale_daily_measurements"),
    (date(2031, 4, 6), date(2031, 4, 9), "record_window_sample_total"),
    (date(2031, 4, 9), date(2031, 4, 10), "rescale_daily_measurements"),
    (date(2031, 4, 9), date(2031, 4, 10), "record_window_sample_total"),
)
assert plans[0].sql is definitions[0].sql
assert dict(plans[0].parameters) == {
    "metric_family": "surface_temperature",
    "minimum_samples": 12,
    "scale_factor": 1.25,
    "segment_open": date(2031, 4, 3),
    "segment_close": date(2031, 4, 6),
}
assert mutation_rejected and strict_type_rejections == 2
```

## Trade-offs and Limitations

The function validates at most 3,720 dates, 24 statements, and 24 fixed
parameters per statement. It measures SQL and text as UTF-8, bytes as their raw
length, numeric values by a stable textual representation, and dates as their
ten-byte ISO form. These limits bound this planner's logical payload; they do
not predict a driver's wire encoding or Python object overhead.

Only exact immutable scalar types cross the output boundary. `datetime` is not
accepted as a `date`, and `bool` is handled as its own type rather than as an
`int`. Exact input dictionaries are snapshotted before expansion, and each
result exposes a read-only mapping over a private copy. The tuple, frozen plan
records, scalar values, and mappings therefore do not retain mutable input
aliases.

Statement definitions are trusted: the planner does not parse placeholders,
inspect schemas, or establish dialect compatibility. It never loads template
files, renders or replaces SQL, interpolates identifiers or values, connects to
a database, executes, commits, or retries. Producing a plan makes no claim
about transactionality, idempotency, completion, or a successful backfill;
those properties belong to the executor and the reviewed statements.

## Related Snippets

<!-- catalog:related:start -->
- [Plan an Additive SQLite Column Projection](plan-an-additive-sqlite-column-projection.md)
- [Project a Dataclass into a Validated Insert Row](project-a-dataclass-into-a-validated-insert-row.md)
- [Split a Half-Open UTC Range Across Ordered Storage Tiers](split-a-half-open-utc-range-across-ordered-storage-tiers.md)
<!-- catalog:related:end -->
