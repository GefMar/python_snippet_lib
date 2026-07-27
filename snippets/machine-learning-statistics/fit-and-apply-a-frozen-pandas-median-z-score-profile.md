---
title: "Fit and Apply a Frozen pandas Median Z-Score Profile"
snippet_type: integration
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies:
  - name: pandas
    version: "3.0.5"
related:
  - fit-and-apply-fixed-quantile-clipping-bounds.md
  - fit-pca-with-numpy-and-report-cumulative-explained-variance.md
  - ../data-processing/audit-pandas-missing-value-shares-against-column-policies.md
---

# Fit and Apply a Frozen pandas Median Z-Score Profile

## Idea and Problem

Fit median imputation and population z-score statistics once on a bounded pandas training frame, then apply the frozen profile to later frames without changing the inputs or learned state.

Each selected column keeps its training median, the mean and population
standard deviation after median imputation, and a z-score divisor. A constant
column receives a divisor of `1.0`, so its transformed training values are
zero. Every transformed feature is paired with a Boolean missing-value
indicator because imputation otherwise hides which values were absent.

## When to Use

Use this integration when one exact pandas `DataFrame` establishes numeric
preprocessing state that must be reused unchanged. Fitting accepts 1 to 10,000
rows and an ordered tuple of 1 to 16 selected columns; transformation accepts
0 to 10,000 rows, and either frame may contain at most 256 total columns. Every
input column label must be a unique, NFC-normalized, stripped printable exact
string of at most 48 characters. A selected column's non-missing cells must all
be real numbers; this also permits object-backed columns whose actual values
satisfy the same narrow scalar contract.

Non-missing values must be finite real numbers within
`[-1_000_000_000.0, 1_000_000_000.0]`. pandas-recognized missing scalars are
imputed, but Boolean, complex, text, infinite, and out-of-range values are
rejected. The generated `__z_score` and `__is_missing` labels must not already
exist in either input frame, which prevents an application step from silently
overwriting or ambiguously reusing a column name.

## Implementation

```python
import math
import unicodedata
from dataclasses import dataclass
from numbers import Real

import pandas as pd

_FIT_ROW_CEILING = 10_000
_APPLY_ROW_CEILING = 10_000
_FRAME_COLUMN_CEILING = 256
_FEATURE_CEILING = 16
_INPUT_LABEL_CEILING = 48
_OUTPUT_LABEL_CEILING = 64
_SAFE_ABSOLUTE_VALUE = 1_000_000_000.0
_SCORE_SUFFIX = "__z_score"
_INDICATOR_SUFFIX = "__is_missing"


def _bounded_label(value: object, *, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if not 1 <= len(value) <= _INPUT_LABEL_CEILING:
        raise ValueError(f"{field} length is outside the supported range")
    if value != value.strip() or not value.isprintable():
        raise ValueError(f"{field} must be stripped printable text")
    if value != unicodedata.normalize("NFC", value):
        raise ValueError(f"{field} must use NFC normalization")
    return value


def _output_labels(input_label: str) -> tuple[str, str]:
    score_label = f"{input_label}{_SCORE_SUFFIX}"
    indicator_label = f"{input_label}{_INDICATOR_SUFFIX}"
    if max(len(score_label), len(indicator_label)) > _OUTPUT_LABEL_CEILING:
        raise ValueError("generated output label exceeds the supported range")
    return score_label, indicator_label


def _frame_labels(
    frame: object,
    *,
    minimum_rows: int,
    maximum_rows: int,
) -> tuple[str, ...]:
    if type(frame) is not pd.DataFrame:
        raise TypeError("frame must be an exact pandas DataFrame")
    if not minimum_rows <= len(frame) <= maximum_rows:
        raise ValueError("frame row count is outside the supported range")
    if len(frame.columns) > _FRAME_COLUMN_CEILING:
        raise ValueError("frame column count exceeds the supported range")
    labels = tuple(_bounded_label(label, field="frame column label") for label in frame.columns)
    if len(set(labels)) != len(labels):
        raise ValueError("frame column labels must be unique")
    return labels


def _selected_labels(
    values: object,
    *,
    frame_labels: tuple[str, ...],
) -> tuple[str, ...]:
    if type(values) is not tuple:
        raise TypeError("selected_labels must be an exact tuple")
    if not 1 <= len(values) <= _FEATURE_CEILING:
        raise ValueError("selected column count is outside the supported range")
    selected = tuple(_bounded_label(value, field="selected column label") for value in values)
    if len(set(selected)) != len(selected):
        raise ValueError("selected column labels must be unique")
    if not set(selected).issubset(frame_labels):
        raise ValueError("every selected column must exist in the frame")
    _ensure_output_labels_available(selected, frame_labels=frame_labels)
    return selected


def _ensure_output_labels_available(
    selected: tuple[str, ...],
    *,
    frame_labels: tuple[str, ...],
) -> None:
    generated = tuple(
        output_label for input_label in selected for output_label in _output_labels(input_label)
    )
    if len(set(generated)) != len(generated):
        raise ValueError("generated output labels must be unique")
    if set(generated).intersection(frame_labels):
        raise ValueError("generated output labels collide with frame columns")


def _finite_or_missing(value: object, *, label: str) -> float | None:
    if not pd.api.types.is_scalar(value):
        raise TypeError(f"column {label!r} must contain scalar values")
    try:
        is_missing = bool(pd.isna(value))
    except (TypeError, ValueError) as error:
        raise TypeError(f"column {label!r} has an unsupported value") from error
    if is_missing:
        return None
    if isinstance(value, bool) or not isinstance(value, Real):
        raise TypeError(f"column {label!r} must contain real numbers or missing values")
    try:
        normalized = float(value)
    except (OverflowError, TypeError, ValueError) as error:
        raise ValueError(f"column {label!r} values must fit in finite floats") from error
    if not math.isfinite(normalized):
        raise ValueError(f"column {label!r} values must be finite")
    if abs(normalized) > _SAFE_ABSOLUTE_VALUE:
        raise ValueError(f"column {label!r} value exceeds the safe magnitude bound")
    return normalized


def _numeric_column(frame: pd.DataFrame, label: str) -> tuple[float | None, ...]:
    series = frame[label]
    dtype = series.dtype
    if (
        isinstance(dtype, (pd.CategoricalDtype, pd.SparseDtype))
        or pd.api.types.is_bool_dtype(dtype)
        or pd.api.types.is_complex_dtype(dtype)
    ):
        raise TypeError(f"column {label!r} must use a supported real representation")
    if len(series) == 0 and not pd.api.types.is_numeric_dtype(dtype):
        raise TypeError(f"empty column {label!r} must retain a real numeric dtype")
    return tuple(_finite_or_missing(value, label=label) for value in series.array)


def _median(non_missing: list[float]) -> float:
    ordered = sorted(non_missing)
    middle = len(ordered) // 2
    if len(ordered) % 2:
        return ordered[middle]
    return math.fsum((ordered[middle - 1], ordered[middle])) / 2.0


def _population_standard_deviation(
    values: tuple[float, ...],
    *,
    center: float,
) -> float:
    deviations = tuple(value - center for value in values)
    largest_deviation = max((abs(value) for value in deviations), default=0.0)
    if largest_deviation == 0.0:
        return 0.0
    relative_square_sum = math.fsum((value / largest_deviation) ** 2 for value in deviations)
    return largest_deviation * math.sqrt(relative_square_sum / len(values))


@dataclass(frozen=True, slots=True)
class FrozenColumnRule:
    input_label: str
    fill_median: float
    imputed_mean: float
    population_standard_deviation: float
    z_divisor: float
    score_label: str
    indicator_label: str

    def __post_init__(self) -> None:
        input_label = _bounded_label(self.input_label, field="rule input label")
        expected_score, expected_indicator = _output_labels(input_label)
        if (self.score_label, self.indicator_label) != (
            expected_score,
            expected_indicator,
        ):
            raise ValueError("rule output labels do not match its input label")
        for field, value in (
            ("fill_median", self.fill_median),
            ("imputed_mean", self.imputed_mean),
            ("population_standard_deviation", self.population_standard_deviation),
            ("z_divisor", self.z_divisor),
        ):
            if type(value) is not float or not math.isfinite(value):
                raise TypeError(f"{field} must be a finite float")
        if abs(self.fill_median) > _SAFE_ABSOLUTE_VALUE:
            raise ValueError("fill_median exceeds the safe magnitude bound")
        if abs(self.imputed_mean) > _SAFE_ABSOLUTE_VALUE:
            raise ValueError("imputed_mean exceeds the safe magnitude bound")
        if not 0.0 <= self.population_standard_deviation <= _SAFE_ABSOLUTE_VALUE:
            raise ValueError("population_standard_deviation is outside its bounds")
        expected_divisor = (
            1.0 if self.population_standard_deviation == 0.0 else self.population_standard_deviation
        )
        if self.z_divisor != expected_divisor:
            raise ValueError("z_divisor does not match the fitted spread")


@dataclass(frozen=True, slots=True)
class FrozenMedianZScoreProfile:
    fitted_row_count: int
    ordered_rules: tuple[FrozenColumnRule, ...]

    def __post_init__(self) -> None:
        if (
            type(self.fitted_row_count) is not int
            or not 1 <= self.fitted_row_count <= _FIT_ROW_CEILING
        ):
            raise ValueError("fitted_row_count is outside the supported range")
        if type(self.ordered_rules) is not tuple:
            raise TypeError("ordered_rules must be an exact tuple")
        if not 1 <= len(self.ordered_rules) <= _FEATURE_CEILING:
            raise ValueError("ordered_rules count is outside the supported range")
        if any(type(rule) is not FrozenColumnRule for rule in self.ordered_rules):
            raise TypeError("ordered_rules must contain exact FrozenColumnRule values")
        input_labels = tuple(rule.input_label for rule in self.ordered_rules)
        if len(set(input_labels)) != len(input_labels):
            raise ValueError("profile input labels must be unique")
        output_labels = tuple(
            label
            for rule in self.ordered_rules
            for label in (rule.score_label, rule.indicator_label)
        )
        if len(set(output_labels)) != len(output_labels):
            raise ValueError("profile output labels must be unique")
        if set(output_labels).intersection(input_labels):
            raise ValueError("profile output labels must not reuse input labels")


def fit_frozen_median_z_score_profile(
    frame: pd.DataFrame,
    *,
    selected_labels: tuple[str, ...],
) -> FrozenMedianZScoreProfile:
    frame_labels = _frame_labels(
        frame,
        minimum_rows=1,
        maximum_rows=_FIT_ROW_CEILING,
    )
    selected = _selected_labels(selected_labels, frame_labels=frame_labels)

    rules: list[FrozenColumnRule] = []
    for label in selected:
        values = _numeric_column(frame, label)
        non_missing = [value for value in values if value is not None]
        if not non_missing:
            raise ValueError(f"training column {label!r} must not be all missing")
        fill_median = _median(non_missing)
        imputed = tuple(fill_median if value is None else value for value in values)
        imputed_mean = math.fsum(imputed) / len(imputed)
        population_spread = _population_standard_deviation(
            imputed,
            center=imputed_mean,
        )
        score_label, indicator_label = _output_labels(label)
        rules.append(
            FrozenColumnRule(
                input_label=label,
                fill_median=fill_median,
                imputed_mean=imputed_mean,
                population_standard_deviation=population_spread,
                z_divisor=1.0 if population_spread == 0.0 else population_spread,
                score_label=score_label,
                indicator_label=indicator_label,
            )
        )
    return FrozenMedianZScoreProfile(
        fitted_row_count=len(frame),
        ordered_rules=tuple(rules),
    )


def apply_frozen_median_z_score_profile(
    frame: pd.DataFrame,
    profile: FrozenMedianZScoreProfile,
) -> pd.DataFrame:
    if type(profile) is not FrozenMedianZScoreProfile:
        raise TypeError("profile must be an exact FrozenMedianZScoreProfile")
    frame_labels = _frame_labels(
        frame,
        minimum_rows=0,
        maximum_rows=_APPLY_ROW_CEILING,
    )
    selected = tuple(rule.input_label for rule in profile.ordered_rules)
    if not set(selected).issubset(frame_labels):
        raise ValueError("every profile input column must exist in the frame")
    _ensure_output_labels_available(selected, frame_labels=frame_labels)

    output: dict[str, pd.api.extensions.ExtensionArray] = {}
    for rule in profile.ordered_rules:
        scores: list[float] = []
        missing_flags: list[bool] = []
        for value in _numeric_column(frame, rule.input_label):
            is_missing = value is None
            prepared = rule.fill_median if is_missing else value
            score = (prepared - rule.imputed_mean) / rule.z_divisor
            if not math.isfinite(score):
                raise ValueError(f"column {rule.input_label!r} produces a non-finite z-score")
            scores.append(score)
            missing_flags.append(is_missing)
        output[rule.score_label] = pd.array(scores, dtype="float64")
        output[rule.indicator_label] = pd.array(missing_flags, dtype="bool")

    return pd.DataFrame(
        output,
        index=frame.index.copy(deep=True),
        copy=True,
    )
```

## Example

```python
training = pd.DataFrame(
    {
        "temperature": pd.array([10.0, pd.NA, 14.0, 18.0], dtype="Float64"),
        "pressure": [5.0, 5.0, 5.0, 5.0],
    },
    index=["fit-1", "fit-2", "fit-3", "fit-4"],
)
future = pd.DataFrame(
    {
        "temperature": [14.0, None, 1_000_000_000.0],
        "pressure": [5.0, None, 9.0],
    },
    index=["future-1", "future-2", "future-3"],
)
training_before = training.copy(deep=True)
future_before = future.copy(deep=True)

profile = fit_frozen_median_z_score_profile(
    training,
    selected_labels=("temperature", "pressure"),
)
profile_state = (
    profile.fitted_row_count,
    tuple(
        (
            rule.input_label,
            rule.fill_median,
            rule.imputed_mean,
            rule.population_standard_deviation,
            rule.z_divisor,
        )
        for rule in profile.ordered_rules
    ),
)
transformed = apply_frozen_median_z_score_profile(future, profile)
empty = apply_frozen_median_z_score_profile(future.iloc[:0].copy(), profile)
current_profile_state = (
    profile.fitted_row_count,
    tuple(
        (
            rule.input_label,
            rule.fill_median,
            rule.imputed_mean,
            rule.population_standard_deviation,
            rule.z_divisor,
        )
        for rule in profile.ordered_rules
    ),
)

assert (
    tuple(transformed.columns),
    transformed.index.equals(future.index),
    transformed["temperature__is_missing"].tolist(),
    transformed["pressure__z_score"].tolist(),
    math.isfinite(transformed.loc["future-3", "temperature__z_score"]),
    transformed.loc["future-3", "temperature__z_score"] > 300_000_000.0,
    empty.shape,
    empty.index.equals(future.iloc[:0].index),
    profile.ordered_rules[1].population_standard_deviation,
    profile.ordered_rules[1].z_divisor,
    profile_state == current_profile_state,
    training.equals(training_before),
    future.equals(future_before),
) == (
    (
        "temperature__z_score",
        "temperature__is_missing",
        "pressure__z_score",
        "pressure__is_missing",
    ),
    True,
    [False, True, False],
    [0.0, 0.0, 4.0],
    True,
    True,
    (0, 4),
    True,
    0.0,
    1.0,
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

Fitting copies and sorts each selected column's non-missing values, costing
`O(features * rows * log(rows))` time in the worst case and bounded working
memory. Transformation is linear in selected cells and allocates a new frame.
`math.fsum` reduces summation error, while the two-pass, rescaled spread
calculation avoids directly summing large squared deviations. Float conversion
can still lose integer precision, and a z-score that cannot be represented as
a finite float is rejected instead of being emitted.

Median imputation is robust to isolated extremes when choosing the replacement
value, but the fitted mean and population spread remain sensitive to them.
The missing indicator doubles the output-column count, and a zero-variation
feature carries no scale information even though the explicit `1.0` divisor
keeps transformation defined. The 256-column ceiling, one-billion magnitude
limit, and label rules are deliberate safety boundaries; change them only after
reviewing numerical range, memory, naming, and downstream compatibility
together.

## Related Snippets

<!-- catalog:related:start -->
- [Fit and Apply Fixed Quantile Clipping Bounds](fit-and-apply-fixed-quantile-clipping-bounds.md)
- [Fit PCA with NumPy and Report Cumulative Explained Variance](fit-pca-with-numpy-and-report-cumulative-explained-variance.md)
- [Audit pandas Missing-Value Shares Against Column Policies](../data-processing/audit-pandas-missing-value-shares-against-column-policies.md)
<!-- catalog:related:end -->
