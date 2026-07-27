---
title: "Score Feature Importances Against Bounded Null Runs"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies:
  - name: numpy
    version: "2.5.1"
related:
  - measure-drift-between-two-fixed-bin-count-distributions-with-psi.md
  - encode-categories-with-out-of-fold-smoothed-target-means.md
  - fit-pca-with-numpy-and-report-cumulative-explained-variance.md
---

# Score Feature Importances Against Bounded Null Runs

## Idea and Problem

Compare each observed feature importance with an aligned matrix of importances from fixed null runs and report its strict empirical rank.

For a higher-is-better importance metric, the rank is the share of null values
strictly below the observed value. Separate below, equal, and greater counts
make ties visible instead of silently treating them as evidence. The function
only reduces already computed values; it does not fit a model, shuffle a
target, select features, or claim statistical significance.

## When to Use

Use this algorithm after generating a bounded set of comparable null runs with
the same feature order, preprocessing, model settings, folds, and importance
definition as the observed run. It is useful as a descriptive diagnostic when
raw importance magnitudes are difficult to compare across features.

The caller must establish that larger values mean stronger importance. Use a
separately reviewed experiment or inference procedure when null generation,
cross-validation leakage, correlated features, repeated selection, or a
formal p-value matters. Do not choose a production feature set from this rank
alone.

## Implementation

```python
from dataclasses import dataclass

import numpy as np


_MAX_FEATURES = 512
_MAX_NULL_RUNS = 10_000
_MAX_COMPARISONS = 2_000_000
_MAX_FEATURE_NAME_CHARS = 128
_SUPPORTED_DTYPES = (np.dtype("float32"), np.dtype("float64"))


@dataclass(frozen=True, slots=True)
class NullImportanceScore:
    feature: str
    observed: float
    null_runs: int
    below: int
    equal: int
    greater: int
    strict_rank: float


def _feature_names(value: object) -> tuple[str, ...]:
    if type(value) is not tuple:
        raise TypeError("feature_names must be an exact tuple")
    if not 1 <= len(value) <= _MAX_FEATURES:
        raise ValueError("feature count is outside the supported range")

    names = []
    for name in value:
        if type(name) is not str:
            raise TypeError("feature names must be exact strings")
        if not name or len(name) > _MAX_FEATURE_NAME_CHARS:
            raise ValueError("feature name length is outside the supported range")
        if name != name.strip() or not name.isprintable():
            raise ValueError("feature names must be stripped printable text")
        names.append(name)
    if len(set(names)) != len(names):
        raise ValueError("feature names must be unique")
    return tuple(names)


def _float_array(
    value: object,
    *,
    field: str,
    dimensions: int,
) -> np.ndarray:
    if type(value) is not np.ndarray:
        raise TypeError(f"{field} must be an exact NumPy array")
    if value.ndim != dimensions:
        raise ValueError(f"{field} has an invalid number of dimensions")
    if value.dtype not in _SUPPORTED_DTYPES or not value.dtype.isnative:
        raise TypeError(f"{field} must use native float32 or float64 values")
    return value


def score_null_importances(
    feature_names: tuple[str, ...],
    observed: np.ndarray,
    null_runs: np.ndarray,
) -> tuple[NullImportanceScore, ...]:
    names = _feature_names(feature_names)
    observed_array = _float_array(
        observed,
        field="observed",
        dimensions=1,
    )
    null_array = _float_array(
        null_runs,
        field="null_runs",
        dimensions=2,
    )

    feature_count = len(names)
    if observed_array.shape != (feature_count,):
        raise ValueError("observed must contain one value per feature")
    run_count, null_feature_count = null_array.shape
    if null_feature_count != feature_count:
        raise ValueError("null_runs columns must align with feature_names")
    if not 1 <= run_count <= _MAX_NULL_RUNS:
        raise ValueError("null run count is outside the supported range")
    if run_count * feature_count > _MAX_COMPARISONS:
        raise ValueError("null comparison count exceeds the supported limit")
    if not bool(np.isfinite(observed_array).all()):
        raise ValueError("observed must contain only finite values")
    if not bool(np.isfinite(null_array).all()):
        raise ValueError("null_runs must contain only finite values")

    observed_values = observed_array.astype(np.float64, copy=True)
    null_values = null_array.astype(np.float64, copy=True)

    below_counts = np.sum(
        null_values < observed_values,
        axis=0,
        dtype=np.int64,
    )
    equal_counts = np.sum(
        null_values == observed_values,
        axis=0,
        dtype=np.int64,
    )

    scores = []
    for index, name in enumerate(names):
        below = int(below_counts[index])
        equal = int(equal_counts[index])
        greater = run_count - below - equal
        scores.append(
            NullImportanceScore(
                feature=name,
                observed=float(observed_values[index]),
                null_runs=run_count,
                below=below,
                equal=equal,
                greater=greater,
                strict_rank=below / run_count,
            )
        )
    return tuple(scores)
```

## Example

```python
feature_names = ("signal", "noise", "inverse")
observed = np.array([0.8, 0.4, -0.1], dtype=np.float64)
null_runs = np.array(
    [
        [0.1, 0.5, -0.1],
        [0.7, 0.4, -0.2],
        [0.9, 0.3, 0.0],
        [0.2, 0.6, -0.1],
    ],
    dtype=np.float64,
)
observed_before = observed.copy()
null_before = null_runs.copy()

scores = score_null_importances(feature_names, observed, null_runs)

assert (
    tuple(
        (
            score.feature,
            score.below,
            score.equal,
            score.greater,
            score.strict_rank,
        )
        for score in scores
    ),
    np.array_equal(observed, observed_before),
    np.array_equal(null_runs, null_before),
) == (
    (
        ("signal", 3, 0, 1, 0.75),
        ("noise", 1, 1, 2, 0.25),
        ("inverse", 1, 2, 1, 0.25),
    ),
    True,
    True,
)
```

## Trade-offs and Limitations

The calculation uses `O(runs * features)` comparisons and temporarily creates
comparison arrays of the same shape as the null matrix. Inputs are copied to
stable `float64` snapshots, so the function also needs bounded additional
memory. Exact floating-point equality defines ties; use a separately specified
tolerance policy if the null generator does not produce meaningfully exact
values.

A high strict rank only says that the observed importance exceeds many values
in the supplied null matrix. It is not a calibrated p-value, a correction for
multiple comparisons, evidence of causality, or proof that a feature improves
held-out performance. Results inherit every choice and defect in the process
that produced the observed and null importances.

## Related Snippets

<!-- catalog:related:start -->
- [Measure Drift Between Two Fixed-Bin Count Distributions with PSI](measure-drift-between-two-fixed-bin-count-distributions-with-psi.md)
- [Encode Categories with Out-of-Fold Smoothed Target Means](encode-categories-with-out-of-fold-smoothed-target-means.md)
- [Fit PCA with NumPy and Report Cumulative Explained Variance](fit-pca-with-numpy-and-report-cumulative-explained-variance.md)
<!-- catalog:related:end -->
