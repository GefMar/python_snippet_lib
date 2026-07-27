---
title: "Fit PCA with NumPy and Report Cumulative Explained Variance"
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
  - score-feature-importances-against-bounded-null-runs.md
  - fit-and-apply-fixed-quantile-clipping-bounds.md
  - calculate-a-symmetrically-trimmed-mean.md
---

# Fit PCA with NumPy and Report Cumulative Explained Variance

## Idea and Problem

Fit principal component analysis to a bounded real matrix and report how much centered variation each singular direction explains.

The function centers every feature but does not standardize features, computes
singular values with `numpy.linalg.svd(..., compute_uv=False)`, and returns
immutable one-based component numbers, explained-variance ratios, and
cumulative ratios. A matrix with zero total centered variance has an explicit
all-zero report instead of producing a division by zero.

## When to Use

Use this report during exploratory analysis or inside a training-only
preprocessing fit when the question is how many principal directions account
for a chosen share of the observed variance. Rows must be comparable samples,
columns must use native `float32` or `float64` values, and feature scale must
already express the relative weighting you intend.

Fit the report only on training data when it informs a later model or feature
choice. Use a complete PCA implementation when transformed coordinates,
components, inverse transformation, whitening, incremental fitting, or model
persistence are required; this function intentionally returns none of those
artifacts.

## Implementation

```python
import math
from dataclasses import dataclass

import numpy as np


_MAX_ROWS = 10_000
_MAX_FEATURES = 256
_MAX_ELEMENTS = 100_000
_SUPPORTED_DTYPES = frozenset({np.dtype("float32"), np.dtype("float64")})


@dataclass(frozen=True, slots=True)
class PcaVarianceReport:
    components: tuple[int, ...]
    explained_variance_ratios: tuple[float, ...]
    cumulative_explained_variance_ratios: tuple[float, ...]


def pca_variance_report(samples: np.ndarray) -> PcaVarianceReport:
    if type(samples) is not np.ndarray:
        raise TypeError("samples must be an exact NumPy ndarray")
    if samples.ndim != 2:
        raise ValueError("samples must be two-dimensional")

    row_count, feature_count = samples.shape
    if not 2 <= row_count <= _MAX_ROWS:
        raise ValueError("row count is outside the supported range")
    if not 1 <= feature_count <= _MAX_FEATURES:
        raise ValueError("feature count is outside the supported range")
    if samples.size > _MAX_ELEMENTS:
        raise ValueError("matrix size exceeds the supported limit")
    if samples.dtype not in _SUPPORTED_DTYPES or not samples.dtype.isnative:
        raise TypeError("samples must use native float32 or float64 values")

    values = samples.astype(np.float64, copy=True)
    if not bool(np.isfinite(values).all()):
        raise ValueError("samples must contain only finite values")

    magnitude = float(np.max(np.abs(values), initial=0.0))
    numerically_scaled = values if magnitude == 0.0 else values / magnitude
    centered = numerically_scaled - numerically_scaled.mean(
        axis=0,
        dtype=np.float64,
        keepdims=True,
    )
    singular_values = np.linalg.svd(centered, compute_uv=False)

    component_count = min(row_count, feature_count)
    components = tuple(range(1, component_count + 1))
    largest = float(singular_values[0])
    if largest == 0.0:
        zeros = (0.0,) * component_count
        return PcaVarianceReport(components, zeros, zeros)

    relative_squares = np.square(singular_values / largest)
    total = float(relative_squares.sum(dtype=np.float64))
    ratios = tuple(float(value / total) for value in relative_squares)

    running = 0.0
    cumulative_values: list[float] = []
    for ratio in ratios:
        running = math.fsum((running, ratio))
        cumulative_values.append(min(1.0, max(0.0, running)))
    cumulative_values[-1] = 1.0

    return PcaVarianceReport(
        components=components,
        explained_variance_ratios=ratios,
        cumulative_explained_variance_ratios=tuple(cumulative_values),
    )
```

## Example

```python
samples = np.array(
    [
        [-2.0, -1.0],
        [0.0, 2.0],
        [2.0, -1.0],
    ],
    dtype=np.float64,
)
before = samples.copy()

report = pca_variance_report(samples)
constant_report = pca_variance_report(
    np.full((3, 2), 7.0, dtype=np.float64)
)

assert (
    report.components,
    np.allclose(report.explained_variance_ratios, (4 / 7, 3 / 7)),
    np.allclose(
        report.cumulative_explained_variance_ratios,
        (4 / 7, 1.0),
    ),
    constant_report.explained_variance_ratios,
    constant_report.cumulative_explained_variance_ratios,
    np.array_equal(samples, before),
) == ((1, 2), True, True, (0.0, 0.0), (0.0, 0.0), True)
```

## Trade-offs and Limitations

Dense SVD costs roughly `O(min(rows * features**2, features * rows**2))`
time and stores a copied dense matrix. The row, feature, and element bounds
keep that work finite, but this is not an incremental or sparse algorithm. A
single common numerical scale is applied internally before centering to avoid
overflow; unlike per-feature standardization, it does not change explained
variance ratios or the relative weighting between columns.

PCA is sensitive to units and outliers, and a high variance direction is not
necessarily useful for prediction. Rank-deficient data produces trailing zero
ratios, while a fully constant matrix produces only zeros. Singular-vector
signs are arbitrary, although this report does not expose vectors. Very large
feature dynamic ranges still lose floating-point precision, and component
selection remains a separate, domain-specific decision.

## Related Snippets

<!-- catalog:related:start -->
- [Score Feature Importances Against Bounded Null Runs](score-feature-importances-against-bounded-null-runs.md)
- [Fit and Apply Fixed Quantile Clipping Bounds](fit-and-apply-fixed-quantile-clipping-bounds.md)
- [Calculate a Symmetrically Trimmed Mean](calculate-a-symmetrically-trimmed-mean.md)
<!-- catalog:related:end -->
