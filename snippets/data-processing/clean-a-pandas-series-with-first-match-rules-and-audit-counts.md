---
title: "Clean a pandas Series with First-Match Rules and Audit Counts"
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
  - audit-pandas-missing-value-shares-against-column-policies.md
  - normalize-optional-csv-columns-in-a-single-pass.md
  - ../python-language/make-a-defensive-copy-at-a-mutable-input-boundary.md
---

# Clean a pandas Series with First-Match Rules and Audit Counts

## Idea and Problem

Apply ordered scalar replacements to a pandas Series while making overlapping rule matches and first-match ownership visible.

Every predicate receives its own isolated copy of the original passive-scalar
Series, so no rule observes replacements made by an earlier rule. A rule's match count
includes all of its true positions, but its applied count excludes positions
already owned by an earlier rule. Missing values in a nullable Boolean mask are
treated as false.

## When to Use

Use this integration for a bounded Series and a small, trusted rule table when
replacement precedence is intentional and audit counts are required. Each
rule must have a unique descriptive name, one scalar replacement, and a
predicate that returns an equally long Boolean or nullable-Boolean Series with
the exact same index order. The input must use a Boolean, numeric, string,
datetime, or timedelta dtype and a passive one-level index; object payloads,
categoricals, arbitrary index objects, and mutable replacements are rejected
before any predicate runs.

Predicates should be deterministic and free of external side effects. The
function protects the caller's Series from pandas-level assignment even when a
predicate raises or returns a malformed mask, but it cannot undo external I/O
performed by arbitrary callback code.

## Implementation

```python
from collections.abc import Callable
from dataclasses import dataclass

import pandas as pd
from pandas.api.types import is_bool_dtype


_MAX_SERIES_ITEMS = 100_000
_MAX_RULES = 64
_MAX_RULE_NAME_CHARS = 128
_MAX_TEXT_CHARS = 4_096


SeriesPredicate = Callable[[pd.Series], pd.Series]


@dataclass(frozen=True, slots=True)
class SeriesReplacementRule:
    name: str
    replacement: object
    predicate: SeriesPredicate


@dataclass(frozen=True, slots=True)
class RuleApplicationCount:
    name: str
    matched: int
    applied: int
    shadowed: int


def _rule_name(value: object) -> str:
    if type(value) is not str:
        raise TypeError("rule names must be exact strings")
    if not value or len(value) > _MAX_RULE_NAME_CHARS:
        raise ValueError("rule name length is outside the supported range")
    if value != value.strip() or not value.isprintable():
        raise ValueError("rule names must be stripped printable text")
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


def _validate_passive_index(index: pd.Index) -> None:
    if isinstance(index, pd.MultiIndex):
        raise TypeError("MultiIndex values are outside the supported boundary")
    _passive_scalar(index.name, field="index name")
    dtype = index.dtype
    if (
        pd.api.types.is_bool_dtype(dtype)
        or pd.api.types.is_integer_dtype(dtype)
        or pd.api.types.is_float_dtype(dtype)
        or pd.api.types.is_datetime64_any_dtype(dtype)
        or pd.api.types.is_timedelta64_dtype(dtype)
    ):
        return
    if isinstance(dtype, pd.StringDtype) or dtype == object:
        for label in index:
            _passive_scalar(label, field="index label")
        return
    raise TypeError("series index dtype is outside the passive boundary")


def _validate_passive_series(series: pd.Series) -> None:
    dtype = series.dtype
    if (
        pd.api.types.is_bool_dtype(dtype)
        or pd.api.types.is_integer_dtype(dtype)
        or pd.api.types.is_float_dtype(dtype)
        or pd.api.types.is_datetime64_any_dtype(dtype)
        or pd.api.types.is_timedelta64_dtype(dtype)
    ):
        pass
    elif isinstance(dtype, pd.StringDtype):
        for value in series.array:
            if value is not pd.NA:
                _passive_scalar(value, field="series text value")
    else:
        raise TypeError("series dtype is outside the passive scalar boundary")
    _validate_passive_index(series.index)
    _passive_scalar(series.name, field="series name")


def _isolated_copy(series: pd.Series) -> pd.Series:
    return pd.Series(
        series.array.copy(),
        index=series.index.copy(deep=True),
        dtype=series.dtype,
        name=series.name,
        copy=True,
    )


def _rules(values: object) -> tuple[SeriesReplacementRule, ...]:
    if type(values) is not tuple:
        raise TypeError("rules must be an exact tuple")
    if not 1 <= len(values) <= _MAX_RULES:
        raise ValueError("rule count is outside the supported range")

    rules = []
    for value in values:
        if type(value) is not SeriesReplacementRule:
            raise TypeError("rules must contain SeriesReplacementRule values")
        name = _rule_name(value.name)
        replacement = _passive_scalar(
            value.replacement,
            field="rule replacement",
        )
        if not callable(value.predicate):
            raise TypeError("every predicate must be callable")
        rules.append(
            SeriesReplacementRule(name, replacement, value.predicate)
        )

    names = tuple(rule.name for rule in rules)
    if len(set(names)) != len(names):
        raise ValueError("rule names must be unique")
    return tuple(rules)


def clean_series_first_match(
    series: pd.Series,
    rules: tuple[SeriesReplacementRule, ...],
) -> tuple[pd.Series, tuple[RuleApplicationCount, ...]]:
    if type(series) is not pd.Series:
        raise TypeError("series must be an exact pandas Series")
    if len(series) > _MAX_SERIES_ITEMS:
        raise ValueError("series exceeds the supported item limit")
    _validate_passive_series(series)
    rule_snapshot = _rules(rules)

    original = _isolated_copy(series)
    cleaned = _isolated_copy(series)
    owned = bytearray(len(series))
    counts = []

    for rule in rule_snapshot:
        predicate_input = _isolated_copy(original)
        mask = rule.predicate(predicate_input)
        if type(mask) is not pd.Series:
            raise TypeError("a predicate must return an exact pandas Series")
        if len(mask) != len(original):
            raise ValueError("a predicate mask must align exactly with the input")
        _validate_passive_index(mask.index)
        if not mask.index.equals(original.index):
            raise ValueError("a predicate mask must align exactly with the input")
        if not is_bool_dtype(mask.dtype):
            raise TypeError("a predicate mask must have a Boolean dtype")

        matches = mask.fillna(False).to_numpy(dtype=bool)
        positions = [
            position
            for position, matched in enumerate(matches)
            if matched and not owned[position]
        ]
        for position in positions:
            owned[position] = 1
        if positions:
            cleaned.iloc[positions] = rule.replacement

        matched_count = int(matches.sum())
        applied_count = len(positions)
        counts.append(
            RuleApplicationCount(
                name=rule.name,
                matched=matched_count,
                applied=applied_count,
                shadowed=matched_count - applied_count,
            )
        )

    return cleaned, tuple(counts)
```

## Example

```python
values = pd.Series(
    ["ready", " ", "unknown", None, "unknown"],
    index=["a", "b", "c", "d", "e"],
    dtype="string",
)
before = values.copy(deep=True)
rules = (
    SeriesReplacementRule(
        "blank",
        pd.NA,
        lambda original: original.str.strip().eq(""),
    ),
    SeriesReplacementRule(
        "placeholder",
        pd.NA,
        lambda original: original.str.strip().isin(["", "unknown"]),
    ),
    SeriesReplacementRule(
        "missing",
        "(missing)",
        lambda original: original.isna(),
    ),
)

cleaned, report = clean_series_first_match(values, rules)


def change_copy_then_fail(candidate: pd.Series) -> pd.Series:
    candidate.iloc[0] = "changed only in the predicate copy"
    raise RuntimeError("predicate failed")


rejections = []
try:
    clean_series_first_match(
        values,
        (
            SeriesReplacementRule(
                "misaligned",
                "x",
                lambda original: pd.Series([True], index=["elsewhere"]),
            ),
        ),
    )
except ValueError:
    rejections.append("misaligned")

try:
    clean_series_first_match(
        values,
        (SeriesReplacementRule("failing", "x", change_copy_then_fail),),
    )
except RuntimeError:
    rejections.append("raised")

assert (
    report,
    cleaned.isna().tolist(),
    cleaned.iloc[list((0, 3))].tolist(),
    values.equals(before),
    cleaned is not values,
    rejections,
) == (
    (
        RuleApplicationCount("blank", matched=1, applied=1, shadowed=0),
        RuleApplicationCount("placeholder", matched=3, applied=2, shadowed=1),
        RuleApplicationCount("missing", matched=1, applied=1, shadowed=0),
    ),
    [False, True, True, False, True],
    ["ready", "(missing)"],
    True,
    True,
    ["misaligned", "raised"],
)
```

## Trade-offs and Limitations

Each rule scans the Series, allocates a predicate copy and mask, and may trigger
a dtype-aware pandas assignment. Runtime is `O(items * rules)`, and peak memory
includes several Series-sized objects. An incompatible replacement can raise
instead of silently widening a dtype; callers should choose replacements that
fit the original dtype or convert under a separate explicit policy.

Match counts describe the original data, not values produced by earlier
replacements. This makes precedence auditable but cannot express a staged
pipeline in which one cleanup enables another. The narrow passive boundary is
deliberate: nested objects, decimal objects, categorical values, MultiIndex
labels, and custom scalar classes need their own ownership and equality
contracts. Series attributes are not copied into the returned value or
predicate inputs. Predicates remain trusted callbacks and can still perform
external side effects that this function cannot roll back.

## Related Snippets

<!-- catalog:related:start -->
- [Audit pandas Missing-Value Shares Against Column Policies](audit-pandas-missing-value-shares-against-column-policies.md)
- [Normalize Optional CSV Columns in a Single Pass](normalize-optional-csv-columns-in-a-single-pass.md)
- [Make a Defensive Copy at a Mutable Input Boundary](../python-language/make-a-defensive-copy-at-a-mutable-input-boundary.md)
<!-- catalog:related:end -->
