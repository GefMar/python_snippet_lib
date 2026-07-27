---
title: "Detect a Recent Drop Against a Disjoint pandas Baseline Window"
snippet_type: integration
use_cases:
  - observability
  - validation
tested_python:
  - "3.14"
dependencies:
  - name: pandas
    version: "3.0.3"
related:
  - flag-groupwise-numeric-outliers-with-iqr-fences.md
  - measure-drift-between-two-fixed-bin-count-distributions-with-psi.md
  - ../data-processing/measure-time-in-a-state-within-a-half-open-window.md
---

# Detect a Recent Drop Against a Disjoint pandas Baseline Window

## Idea and Problem

Compare recent finite measurements with an earlier non-overlapping baseline and return a diagnostic drop decision.

For an evaluation time `T`, the baseline is
`[T - recent_window - baseline_window, T - recent_window)` and the recent
window is `[T - recent_window, T)`. A recent point is bad only when it is
strictly below `baseline_mean - standard_deviations * baseline_population_std`.
Minimum sample counts and an explicit freshness interval prevent sparse or
stale data from producing a positive decision. The freshness interval is also
half-open, so a point exactly at its lower boundary counts as fresh.

## When to Use

Use this bounded check for a regularly interpreted UTC metric when a short
recent interval can be compared meaningfully with its immediately preceding
history. Choose the windows, sample requirements, deviation multiplier, and
bad-point count from domain requirements before evaluating the series.

The baseline and recent windows must represent comparable operating periods.
This simple rule does not handle seasonality, trend, irregular sampling bias,
or multiple-testing effects. Use a time-series model or a domain-specific
control chart when those assumptions matter.

## Implementation

```python
import math
from dataclasses import dataclass
from enum import StrEnum

import pandas as pd


_MAX_SERIES_ITEMS = 100_000
_MAX_SAMPLE_THRESHOLD = 100_000
_MAX_WINDOW = pd.Timedelta(days=365)
_MAX_STANDARD_DEVIATIONS = 100.0


class DropReason(StrEnum):
    INSUFFICIENT_BASELINE = "insufficient-baseline"
    INSUFFICIENT_RECENT = "insufficient-recent"
    STALE_RECENT = "stale-recent"
    DROP_DETECTED = "drop-detected"
    TOO_FEW_BAD_POINTS = "too-few-bad-points"


@dataclass(frozen=True, slots=True)
class RecentDropDecision:
    is_drop: bool
    reasons: tuple[DropReason, ...]
    baseline_start: pd.Timestamp
    recent_start: pd.Timestamp
    evaluated_at: pd.Timestamp
    baseline_count: int
    recent_count: int
    baseline_mean: float | None
    baseline_population_std: float | None
    threshold: float | None
    bad_count: int
    bad_timestamps: tuple[pd.Timestamp, ...]
    latest_recent_at: pd.Timestamp | None


def _exact_utc_timestamp(value: object, *, field: str) -> pd.Timestamp:
    if type(value) is not pd.Timestamp:
        raise TypeError(f"{field} must be a pandas Timestamp")
    if value.tz is None or str(value.tz) != "UTC":
        raise ValueError(f"{field} must use the UTC timezone")
    return value


def _positive_window(value: object, *, field: str) -> pd.Timedelta:
    if type(value) is not pd.Timedelta:
        raise TypeError(f"{field} must be a pandas Timedelta")
    if not pd.Timedelta(0) < value <= _MAX_WINDOW:
        raise ValueError(f"{field} is outside the supported range")
    return value


def _sample_threshold(value: object, *, field: str, minimum: int) -> int:
    if type(value) is not int:
        raise TypeError(f"{field} must be an integer")
    if not minimum <= value <= _MAX_SAMPLE_THRESHOLD:
        raise ValueError(f"{field} is outside the supported range")
    return value


def _finite_float(value: object, *, field: str) -> float:
    if type(value) not in (int, float):
        raise TypeError(f"{field} must be an integer or float")
    try:
        result = float(value)
    except OverflowError as error:
        raise ValueError(f"{field} must fit in a finite float") from error
    if not math.isfinite(result):
        raise ValueError(f"{field} must be finite")
    return result


def _population_stats(values: list[float]) -> tuple[float, float]:
    scale = max(map(abs, values))
    if scale == 0.0:
        return 0.0, 0.0
    scaled = [value / scale for value in values]
    scaled_mean = math.fsum(scaled) / len(scaled)
    mean = scale * scaled_mean
    scaled_variance = math.fsum(
        (value - scaled_mean) ** 2 for value in scaled
    ) / len(scaled)
    population_std = scale * math.sqrt(max(0.0, scaled_variance))
    if not math.isfinite(mean) or not math.isfinite(population_std):
        raise ValueError("baseline statistics exceed the finite-float range")
    return mean, population_std


def detect_recent_drop(
    series: pd.Series,
    *,
    evaluated_at: pd.Timestamp,
    baseline_window: pd.Timedelta,
    recent_window: pd.Timedelta,
    maximum_staleness: pd.Timedelta,
    minimum_baseline_samples: int,
    minimum_recent_samples: int,
    minimum_bad_points: int,
    standard_deviations: float,
) -> RecentDropDecision:
    if type(series) is not pd.Series:
        raise TypeError("series must be a pandas Series")
    if len(series) > _MAX_SERIES_ITEMS:
        raise ValueError("series length exceeds the supported limit")
    if type(series.index) is not pd.DatetimeIndex:
        raise TypeError("series must have a pandas DatetimeIndex")
    if series.index.tz is None or str(series.index.tz) != "UTC":
        raise ValueError("series index must use the UTC timezone")
    if series.index.hasnans:
        raise ValueError("series index must not contain NaT")
    if not series.index.is_unique or not series.index.is_monotonic_increasing:
        raise ValueError("series index must be unique and strictly increasing")
    if (
        pd.api.types.is_bool_dtype(series.dtype)
        or pd.api.types.is_complex_dtype(series.dtype)
        or not pd.api.types.is_numeric_dtype(series.dtype)
    ):
        raise TypeError("series must have a real numeric dtype")
    if bool(series.isna().any()):
        raise ValueError("series values must not be missing")

    normalized: list[float] = []
    for value in series.array:
        try:
            number = float(value)
        except (TypeError, ValueError, OverflowError) as error:
            raise ValueError("series values must fit in finite floats") from error
        if not math.isfinite(number):
            raise ValueError("series values must be finite")
        normalized.append(number)

    evaluation = _exact_utc_timestamp(evaluated_at, field="evaluated_at")
    baseline_span = _positive_window(baseline_window, field="baseline_window")
    recent_span = _positive_window(recent_window, field="recent_window")
    freshness_span = _positive_window(
        maximum_staleness,
        field="maximum_staleness",
    )
    if freshness_span > recent_span:
        raise ValueError("maximum_staleness must not exceed recent_window")
    minimum_baseline = _sample_threshold(
        minimum_baseline_samples,
        field="minimum_baseline_samples",
        minimum=2,
    )
    minimum_recent = _sample_threshold(
        minimum_recent_samples,
        field="minimum_recent_samples",
        minimum=1,
    )
    minimum_bad = _sample_threshold(
        minimum_bad_points,
        field="minimum_bad_points",
        minimum=1,
    )
    deviation_count = _finite_float(
        standard_deviations,
        field="standard_deviations",
    )
    if not 0.0 <= deviation_count <= _MAX_STANDARD_DEVIATIONS:
        raise ValueError("standard_deviations is outside the supported range")

    try:
        recent_start = evaluation - recent_span
        baseline_start = recent_start - baseline_span
        freshness_start = evaluation - freshness_span
    except (OverflowError, pd.errors.OutOfBoundsDatetime) as error:
        raise ValueError("window boundaries are outside pandas Timestamp range") from error

    baseline_points: list[float] = []
    recent_points: list[tuple[pd.Timestamp, float]] = []
    for timestamp, value in zip(series.index, normalized, strict=True):
        if baseline_start <= timestamp < recent_start:
            baseline_points.append(value)
        elif recent_start <= timestamp < evaluation:
            recent_points.append((timestamp, value))

    mean: float | None = None
    population_std: float | None = None
    threshold: float | None = None
    bad_timestamps: tuple[pd.Timestamp, ...] = ()
    if baseline_points:
        mean, population_std = _population_stats(baseline_points)
        threshold = mean - deviation_count * population_std
        if not math.isfinite(threshold):
            raise ValueError("drop threshold exceeds the finite-float range")
        bad_timestamps = tuple(
            timestamp for timestamp, value in recent_points if value < threshold
        )

    reasons: list[DropReason] = []
    if len(baseline_points) < minimum_baseline:
        reasons.append(DropReason.INSUFFICIENT_BASELINE)
    if len(recent_points) < minimum_recent:
        reasons.append(DropReason.INSUFFICIENT_RECENT)
    latest_recent = recent_points[-1][0] if recent_points else None
    if latest_recent is None or latest_recent < freshness_start:
        reasons.append(DropReason.STALE_RECENT)

    has_blocker = bool(reasons)
    is_drop = not has_blocker and len(bad_timestamps) >= minimum_bad
    if not has_blocker:
        reasons.append(
            DropReason.DROP_DETECTED
            if is_drop
            else DropReason.TOO_FEW_BAD_POINTS
        )

    return RecentDropDecision(
        is_drop=is_drop,
        reasons=tuple(reasons),
        baseline_start=baseline_start,
        recent_start=recent_start,
        evaluated_at=evaluation,
        baseline_count=len(baseline_points),
        recent_count=len(recent_points),
        baseline_mean=mean,
        baseline_population_std=population_std,
        threshold=threshold,
        bad_count=len(bad_timestamps),
        bad_timestamps=bad_timestamps,
        latest_recent_at=latest_recent,
    )
```

## Example

```python
index = pd.date_range(
    "2026-01-10 06:00",
    periods=7,
    freq="h",
    tz="UTC",
)
metric = pd.Series([10.0, 10.0, 12.0, 12.0, 10.0, 9.0, 0.0], index=index)

decision = detect_recent_drop(
    metric,
    evaluated_at=pd.Timestamp("2026-01-10 12:00", tz="UTC"),
    baseline_window=pd.Timedelta(hours=4),
    recent_window=pd.Timedelta(hours=2),
    maximum_staleness=pd.Timedelta(minutes=90),
    minimum_baseline_samples=4,
    minimum_recent_samples=2,
    minimum_bad_points=1,
    standard_deviations=1.0,
)

# The value at 10:00 equals the threshold and is not bad; 12:00 is excluded.
assert (
    (
        decision.is_drop,
        decision.reasons,
        decision.baseline_count,
        decision.recent_count,
        decision.baseline_mean,
        decision.threshold,
        decision.bad_timestamps,
    )
    == (
        True,
        (DropReason.DROP_DETECTED,),
        4,
        2,
        11.0,
        10.0,
        (pd.Timestamp("2026-01-10 11:00", tz="UTC"),),
    )
    and decision.baseline_population_std is not None
    and math.isclose(
        decision.baseline_population_std,
        1.0,
        rel_tol=1e-12,
    )
)
```

## Trade-offs and Limitations

The function materializes at most the bounded series length and scans it once;
the dominant cost is `O(n)` time and memory. It converts supported pandas
numeric values to floats, so very large integers can lose exact precision.
Population standard deviation uses `N`, not `N - 1`, and a zero-variance
baseline makes every strictly smaller recent value bad.

Points outside the two windows, including points exactly at the evaluation
time, do not affect the decision. Missing and non-finite values are rejected
rather than imputed, and a stale or undersampled result is always non-dropping
even when some available values are below the provisional threshold. The
result is a bounded diagnostic rule, not a general anomaly detector or proof
of a causal regression.

## Related Snippets

<!-- catalog:related:start -->
- [Flag Groupwise Numeric Outliers with IQR Fences](flag-groupwise-numeric-outliers-with-iqr-fences.md)
- [Measure Drift Between Two Fixed-Bin Count Distributions with PSI](measure-drift-between-two-fixed-bin-count-distributions-with-psi.md)
- [Measure Time in a State Within a Half-Open Window](../data-processing/measure-time-in-a-state-within-a-half-open-window.md)
<!-- catalog:related:end -->
