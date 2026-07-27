---
title: "Select a Forecast Vector Only When It Beats a Frozen Baseline"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - assemble-out-of-fold-scores-from-explicit-validation-splits.md
  - detect-a-recent-drop-against-a-disjoint-pandas-baseline-window.md
  - ../data-processing/select-one-record-per-key-with-an-explicit-ranking-rule.md
---

# Select a Forecast Vector Only When It Beats a Frozen Baseline

## Idea and Problem

Select a precomputed forecast vector only when its finite mean absolute error is strictly lower than that of one frozen baseline.

Every vector is evaluated against the same ordered observations. Scores are
returned in baseline-first, candidate-input order, while selection uses strict
improvement: a baseline tie keeps the baseline, and equally scoring candidates
retain the first candidate's priority.

## When to Use

Use this algorithm after data splitting, model fitting, and prediction are
already complete. The actual, baseline, and candidate tuples must describe the
same non-empty horizon in the same order. This boundary is useful when a simple
reference forecast should remain the default unless a named alternative proves
better under one transparent metric.

Freeze the evaluation vectors before calling the function and decide elsewhere
whether the holdout is representative. The function deliberately accepts no
timestamps, missing values, model objects, callbacks, or training data.

## Implementation

```python
import math
from dataclasses import dataclass


_MAX_VECTOR_LENGTH = 100_000
_MAX_CANDIDATES = 64
_MAX_NAME_CHARACTERS = 128
_MAX_NAME_UTF8_BYTES = 512
_MAX_TOTAL_NAME_UTF8_BYTES = 8_192
_MAX_PAIR_COMPARISONS = 1_000_000


@dataclass(frozen=True, slots=True)
class NamedPrediction:
    name: str
    values: tuple[float, ...]


@dataclass(frozen=True, slots=True)
class VectorScore:
    candidate_name: str | None
    mean_absolute_error: float


@dataclass(frozen=True, slots=True)
class ForecastChoice:
    selected_candidate: str | None
    scores: tuple[VectorScore, ...]


def _bounded_name(value: object) -> tuple[str, int]:
    if type(value) is not str:
        raise TypeError("candidate name must be an exact string")
    if not value or len(value) > _MAX_NAME_CHARACTERS:
        raise ValueError("candidate name length is outside the supported range")
    if value != value.strip() or not value.isprintable():
        raise ValueError("candidate name must be stripped printable text")
    try:
        byte_count = len(value.encode("utf-8"))
    except UnicodeEncodeError as error:
        raise ValueError("candidate name must be valid UTF-8 text") from error
    if byte_count > _MAX_NAME_UTF8_BYTES:
        raise ValueError("candidate name exceeds the UTF-8 byte limit")
    return value, byte_count


def _vector_shape(
    value: object,
    *,
    field: str,
    expected_length: int | None = None,
) -> tuple[float, ...]:
    if type(value) is not tuple:
        raise TypeError(f"{field} must be an exact tuple")
    if expected_length is None:
        if not 1 <= len(value) <= _MAX_VECTOR_LENGTH:
            raise ValueError(f"{field} length is outside the supported range")
    elif len(value) != expected_length:
        raise ValueError(f"{field} must align with the actual vector")
    return value


def _validate_finite_vector(
    values: tuple[float, ...],
    *,
    field: str,
) -> None:
    for value in values:
        if type(value) is not float:
            raise TypeError(f"{field} must contain exact floats")
        if not math.isfinite(value):
            raise ValueError(f"{field} must contain only finite values")


def _mean_absolute_error(
    actual: tuple[float, ...],
    predicted: tuple[float, ...],
) -> float:
    def absolute_errors():
        for observed, estimate in zip(actual, predicted, strict=True):
            difference = observed - estimate
            if not math.isfinite(difference):
                raise OverflowError("a forecast error is not finite")
            yield abs(difference)

    try:
        total = math.fsum(absolute_errors())
    except (OverflowError, ValueError) as error:
        raise OverflowError("the absolute-error sum is not finite") from error
    if not math.isfinite(total):
        raise OverflowError("the absolute-error sum is not finite")
    result = total / len(actual)
    if not math.isfinite(result):
        raise OverflowError("the mean absolute error is not finite")
    return result


def select_forecast_vector(
    actual: tuple[float, ...],
    baseline: tuple[float, ...],
    candidates: tuple[NamedPrediction, ...],
) -> ForecastChoice:
    actual_values = _vector_shape(actual, field="actual")
    horizon = len(actual_values)
    baseline_values = _vector_shape(
        baseline,
        field="baseline",
        expected_length=horizon,
    )
    if type(candidates) is not tuple:
        raise TypeError("candidates must be an exact tuple")
    if not 1 <= len(candidates) <= _MAX_CANDIDATES:
        raise ValueError("candidate count is outside the supported range")
    if horizon * (len(candidates) + 1) > _MAX_PAIR_COMPARISONS:
        raise ValueError("forecast scoring work exceeds the supported limit")

    names: set[str] = set()
    total_name_bytes = 0
    candidate_values: list[tuple[float, ...]] = []
    for candidate in candidates:
        if type(candidate) is not NamedPrediction:
            raise TypeError("candidates must contain exact NamedPrediction values")
        name, byte_count = _bounded_name(candidate.name)
        if name in names:
            raise ValueError("candidate names must be unique")
        names.add(name)
        total_name_bytes += byte_count
        if total_name_bytes > _MAX_TOTAL_NAME_UTF8_BYTES:
            raise ValueError("candidate names exceed the total UTF-8 byte limit")
        candidate_values.append(
            _vector_shape(
                candidate.values,
                field=f"candidate {name!r}",
                expected_length=horizon,
            )
        )

    _validate_finite_vector(actual_values, field="actual")
    _validate_finite_vector(baseline_values, field="baseline")
    for candidate, values in zip(candidates, candidate_values, strict=True):
        _validate_finite_vector(values, field=f"candidate {candidate.name!r}")

    baseline_score = _mean_absolute_error(actual_values, baseline_values)
    scores = [VectorScore(None, baseline_score)]
    selected_name = None
    selected_score = baseline_score
    for candidate, values in zip(candidates, candidate_values, strict=True):
        score = _mean_absolute_error(actual_values, values)
        scores.append(VectorScore(candidate.name, score))
        if score < selected_score:
            selected_name = candidate.name
            selected_score = score

    return ForecastChoice(selected_name, tuple(scores))
```

## Example

```python
actual = (8.0, 11.0, 13.0, 10.0)
baseline = (9.0, 10.0, 12.0, 11.0)
candidates = (
    NamedPrediction("candidate-one", (8.5, 10.5, 12.5, 10.5)),
    NamedPrediction("candidate-two", (7.5, 11.5, 13.5, 9.5)),
    NamedPrediction("baseline-tie", (7.0, 12.0, 14.0, 9.0)),
)
snapshot = (actual, baseline, candidates)

choice = select_forecast_vector(actual, baseline, candidates)
tie_choice = select_forecast_vector(
    actual,
    baseline,
    (NamedPrediction("equal", baseline),),
)

assert (
    choice
    == ForecastChoice(
        selected_candidate="candidate-one",
        scores=(
            VectorScore(None, 1.0),
            VectorScore("candidate-one", 0.5),
            VectorScore("candidate-two", 0.5),
            VectorScore("baseline-tie", 1.0),
        ),
    )
    and tie_choice.selected_candidate is None
    and (actual, baseline, candidates) == snapshot
)
```

## Trade-offs and Limitations

Mean absolute error is easy to interpret but remains scale-dependent and gives
every horizon position equal influence. The function does not establish that
the vectors came from a chronological, leakage-free evaluation, and it does
not estimate whether a small observed improvement is statistically meaningful.

Only exact finite Python floats are accepted. Floating-point subtraction and
summation can still overflow for extreme finite inputs, so every derived error
and score is checked and rejected when it becomes non-finite. The fixed vector,
candidate, name, and comparison limits bound validation and scoring work. No
input is modified, and no fitting, splitting, timestamp alignment, imputation,
prediction, callback execution, pandas operation, persistence, or I/O occurs.

## Related Snippets

<!-- catalog:related:start -->
- [Assemble Out-of-Fold Scores from Explicit Validation Splits](assemble-out-of-fold-scores-from-explicit-validation-splits.md)
- [Detect a Recent Drop Against a Disjoint pandas Baseline Window](detect-a-recent-drop-against-a-disjoint-pandas-baseline-window.md)
- [Select One Record per Key with an Explicit Ranking Rule](../data-processing/select-one-record-per-key-with-an-explicit-ranking-rule.md)
<!-- catalog:related:end -->
