---
title: "Audit pandas Missing-Value Shares Against Column Policies"
snippet_type: integration
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies:
  - name: pandas
    version: "3.0.3"
related:
  - enforce-a-many-to-one-pandas-left-merge-contract.md
  - collect-expected-parse-failures-without-stopping-a-batch.md
  - ../machine-learning-statistics/flag-groupwise-numeric-outliers-with-iqr-fences.md
---

# Audit pandas Missing-Value Shares Against Column Policies

## Idea and Problem

Measure pandas missing-value shares for explicitly governed columns and compare each share with an immutable policy without changing the frame.

Every policy names a known column, a maximum accepted share, a descriptive
action, and a reason for the rule. The result is a tuple of immutable records
ordered with violations first, then by decreasing missing share and column
name. A share violates its policy only when it is strictly greater than the
threshold, so a value exactly on the boundary is accepted.

## When to Use

Use this integration at an in-memory data boundary when a bounded pandas frame
has a small, explicit set of column-level completeness requirements. It is
useful for import preflights and quality reports where `pandas.isna` semantics
are the intended definition of missingness. An empty frame has a missing share
of `0.0` for every governed column.

The returned action and reason are explanatory text only. The function does
not repair data, drop rows, emit alerts, or execute any policy action. Keep
those effects in a separate, reviewed step after interpreting the report.

## Implementation

```python
import math
from collections.abc import Iterable
from dataclasses import dataclass
from itertools import islice

import pandas as pd


_MAX_FRAME_ROWS = 100_000
_MAX_FRAME_COLUMNS = 256
_MAX_POLICIES = 256
_MAX_COLUMN_UTF8_BYTES = 256
_MAX_DESCRIPTION_UTF8_BYTES = 512


def _column_name(value: str) -> str:
    if not isinstance(value, str):
        raise TypeError("column names must be text")
    if not value or value != value.strip() or not value.isprintable():
        raise ValueError("column names must be stripped printable text")
    if len(value.encode("utf-8")) > _MAX_COLUMN_UTF8_BYTES:
        raise ValueError("a column name exceeds the supported UTF-8 byte limit")
    return value


def _description(value: str, *, name: str) -> str:
    if not isinstance(value, str):
        raise TypeError(f"{name} must be text")
    if not value or value != value.strip() or not value.isprintable():
        raise ValueError(f"{name} must be stripped printable text")
    try:
        encoded = value.encode("utf-8")
    except UnicodeEncodeError as error:
        raise ValueError(f"{name} must be valid UTF-8 text") from error
    if len(encoded) > _MAX_DESCRIPTION_UTF8_BYTES:
        raise ValueError(f"{name} exceeds the supported UTF-8 byte limit")
    return value


@dataclass(frozen=True, slots=True)
class MissingValuePolicy:
    column: str
    max_missing_share: float
    action: str
    reason: str

    def __post_init__(self) -> None:
        _column_name(self.column)
        if type(self.max_missing_share) is not float:
            raise TypeError("max_missing_share must be a float")
        if not math.isfinite(self.max_missing_share):
            raise ValueError("max_missing_share must be finite")
        if not 0.0 <= self.max_missing_share <= 1.0:
            raise ValueError("max_missing_share must be between zero and one")
        _description(self.action, name="action")
        _description(self.reason, name="reason")


@dataclass(frozen=True, slots=True)
class MissingValueAuditRecord:
    column: str
    missing_count: int
    row_count: int
    missing_share: float
    max_missing_share: float
    violated: bool
    action: str
    reason: str


def _frame_columns(frame: pd.DataFrame) -> tuple[str, ...]:
    if not isinstance(frame, pd.DataFrame):
        raise TypeError("frame must be a pandas DataFrame")
    if len(frame) > _MAX_FRAME_ROWS:
        raise ValueError("frame exceeds the supported row limit")
    columns = tuple(frame.columns)
    if len(columns) > _MAX_FRAME_COLUMNS:
        raise ValueError("frame exceeds the supported column limit")
    for column in columns:
        _column_name(column)
    if len(set(columns)) != len(columns):
        raise ValueError("frame columns must be unique")
    return columns


def _policy_snapshot(
    policies: Iterable[MissingValuePolicy],
    *,
    known_columns: tuple[str, ...],
) -> tuple[MissingValuePolicy, ...]:
    if isinstance(policies, (str, bytes)):
        raise TypeError("policies must be a non-text iterable")
    try:
        snapshot = tuple(islice(policies, _MAX_POLICIES + 1))
    except TypeError as error:
        raise TypeError("policies must be an iterable") from error
    if not 1 <= len(snapshot) <= _MAX_POLICIES:
        raise ValueError("policy count is outside the supported range")
    if any(type(policy) is not MissingValuePolicy for policy in snapshot):
        raise TypeError("every policy must be an immutable MissingValuePolicy")

    policy_columns = tuple(policy.column for policy in snapshot)
    if len(set(policy_columns)) != len(policy_columns):
        raise ValueError("policy columns must be unique")
    if not set(policy_columns).issubset(known_columns):
        raise ValueError("every policy column must exist in the frame")
    return snapshot


def audit_missing_value_shares(
    frame: pd.DataFrame,
    policies: Iterable[MissingValuePolicy],
) -> tuple[MissingValueAuditRecord, ...]:
    columns = _frame_columns(frame)
    policy_snapshot = _policy_snapshot(policies, known_columns=columns)
    row_count = len(frame)

    records = []
    for policy in policy_snapshot:
        missing_count = int(frame[policy.column].isna().sum())
        missing_share = missing_count / row_count if row_count else 0.0
        records.append(
            MissingValueAuditRecord(
                column=policy.column,
                missing_count=missing_count,
                row_count=row_count,
                missing_share=missing_share,
                max_missing_share=policy.max_missing_share,
                violated=missing_share > policy.max_missing_share,
                action=policy.action,
                reason=policy.reason,
            )
        )

    return tuple(
        sorted(
            records,
            key=lambda record: (
                not record.violated,
                -record.missing_share,
                record.column,
            ),
        )
    )
```

## Example

```python
frame = pd.DataFrame(
    {
        "account_id": [10, None, 30, None],
        "note": ["ready", None, "", None],
        "amount": [7.0, 11.0, 5.0, 13.0],
    }
)
frame_before = frame.copy(deep=True)
policies = (
    MissingValuePolicy(
        "account_id",
        0.5,
        "Review unresolved identifiers",
        "Identifiers should normally be present",
    ),
    MissingValuePolicy(
        "note",
        0.25,
        "Inspect note ingestion",
        "Notes may be absent only occasionally",
    ),
    MissingValuePolicy(
        "amount",
        0.0,
        "Reject incomplete monetary rows",
        "Amounts are required for aggregation",
    ),
)

report = audit_missing_value_shares(frame, policies)
empty_report = audit_missing_value_shares(
    pd.DataFrame({"note": pd.Series(dtype="object")}),
    (MissingValuePolicy("note", 0.0, "No action", "Empty input"),),
)

try:
    MissingValuePolicy("note", True, "Review", "Invalid threshold type")
except TypeError:
    boolean_threshold_rejected = True
else:
    boolean_threshold_rejected = False

assert (
    report,
    empty_report,
    frame.equals(frame_before),
    boolean_threshold_rejected,
) == (
    (
        MissingValueAuditRecord(
            "note", 2, 4, 0.5, 0.25, True,
            "Inspect note ingestion", "Notes may be absent only occasionally",
        ),
        MissingValueAuditRecord(
            "account_id", 2, 4, 0.5, 0.5, False,
            "Review unresolved identifiers", "Identifiers should normally be present",
        ),
        MissingValueAuditRecord(
            "amount", 0, 4, 0.0, 0.0, False,
            "Reject incomplete monetary rows", "Amounts are required for aggregation",
        ),
    ),
    (
        MissingValueAuditRecord(
            "note", 0, 0, 0.0, 0.0, False, "No action", "Empty input",
        ),
    ),
    True,
    True,
)
```

## Trade-offs and Limitations

The audit performs one pandas missingness scan per governed column and stores
one small record per policy. It intentionally follows pandas missing-value
semantics: values such as `None`, `NaN`, `NaT`, and `pd.NA` can be missing,
while an empty string is not missing. Different semantic rules require
normalization before this boundary.

Thresholds say nothing about whether the remaining values are correct,
representative, or safe to use. A frame can also change after this synchronous
check, and the descriptive action is not an enforcement mechanism. Coordinate
mutation externally, retain the report with the policy version, and perform
repairs or alerts separately.

## Related Snippets

<!-- catalog:related:start -->
- [Enforce a Many-to-One pandas Left-Merge Contract](enforce-a-many-to-one-pandas-left-merge-contract.md)
- [Collect Expected Parse Failures Without Stopping a Batch](collect-expected-parse-failures-without-stopping-a-batch.md)
- [Flag Groupwise Numeric Outliers with IQR Fences](../machine-learning-statistics/flag-groupwise-numeric-outliers-with-iqr-fences.md)
<!-- catalog:related:end -->
