---
title: "Compute a Row-Wise Maximum of Rolling Minima"
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
  - detect-a-recent-drop-against-a-disjoint-pandas-baseline-window.md
  - fit-pca-with-numpy-and-report-cumulative-explained-variance.md
  - ../data-processing/fan-out-events-into-bounded-lookback-windows.md
---

# Compute a Row-Wise Maximum of Rolling Minima

## Idea and Problem

Summarize each bounded time row by taking the largest minimum found across all of its contiguous fixed-width windows.

For a window width `w`, each candidate is the minimum of one contiguous block
of `w` periods; the row's result is the maximum candidate. This maximin
summary rewards rows that contain at least one window whose weakest value is
high, without allowing an isolated peak to dominate the result.

## When to Use

Use this operation when rows are comparable bounded sequences, columns are in
time order, and one domain-chosen window has an interpretable duration. The
input must be an exact two-dimensional NumPy array with a native `float32` or
`float64` dtype, finite non-negative values, and dimensions that fit all four
resource limits.

Choose the window before inspecting the result. A width of one returns each
row maximum, while a width equal to the period count returns each row minimum.
Use a richer temporal model when gaps, irregular spacing, seasonality,
multiple window sizes, or the location of the best interval matters.

## Implementation

```python
import numpy as np


_MAX_ROWS = 10_000
_MAX_PERIODS = 10_000
_MAX_ELEMENTS = 1_000_000
_MAX_COMPARISONS = 25_000_000
_SUPPORTED_DTYPES = frozenset(
    {np.dtype("float32"), np.dtype("float64")}
)


def compute_row_wise_maximum_of_rolling_minima(
    values: np.ndarray,
    *,
    window: int,
) -> np.ndarray:
    if type(values) is not np.ndarray:
        raise TypeError("values must be an exact NumPy ndarray")
    if values.ndim != 2:
        raise ValueError("values must be two-dimensional")

    row_count, period_count = values.shape
    if not 1 <= row_count <= _MAX_ROWS:
        raise ValueError("row count is outside the supported range")
    if not 1 <= period_count <= _MAX_PERIODS:
        raise ValueError("period count is outside the supported range")
    if values.size > _MAX_ELEMENTS:
        raise ValueError("matrix size exceeds the supported limit")
    dtype = values.dtype
    if (
        dtype.metadata is not None
        or not dtype.isnative
        or dtype not in _SUPPORTED_DTYPES
    ):
        raise TypeError("values must use an exact native float32 or float64 dtype")
    if type(window) is not int:
        raise TypeError("window must be an exact integer")
    if not 1 <= window <= period_count:
        raise ValueError("window is outside the available period range")

    window_count = period_count - window + 1
    comparisons_per_row = (
        window_count * (window - 1) + window_count - 1
    )
    if row_count * comparisons_per_row > _MAX_COMPARISONS:
        raise ValueError("rolling comparison work exceeds the supported limit")
    if not bool(np.isfinite(values).all()):
        raise ValueError("values must contain only finite numbers")
    if bool((values < 0).any()):
        raise ValueError("values must be non-negative")

    windows = np.lib.stride_tricks.sliding_window_view(
        values,
        window_shape=window,
        axis=1,
    )
    rolling_minima = np.min(windows, axis=-1)
    return np.max(rolling_minima, axis=1)
```

## Example

```python
period_values = np.array(
    [
        [4.0, 1.0, 3.0, 2.0, 5.0],
        [8.0, 6.0, 7.0, 5.0, 9.0],
    ],
    dtype=np.float32,
)
before = period_values.tobytes()

row_scores = compute_row_wise_maximum_of_rolling_minima(
    period_values,
    window=3,
)

assert (
    row_scores.tolist(),
    row_scores.dtype == np.dtype("float32"),
    period_values.tobytes() == before,
) == ([2.0, 6.0], True, True)
```

## Trade-offs and Limitations

The sliding view itself does not copy the matrix, and the reductions allocate
one rolling-minimum value per candidate window plus one result per row. The
direct reduction performs `windows * window - 1` comparisons per row, so wide
windows are more expensive than a deque-based rolling minimum. Row, period,
element, and explicit comparison limits reject excessive work before creating
the view.

The result preserves the input dtype and the implementation never writes to
the input, but concurrent mutation by another thread can still make a read
inconsistent. Exact non-negative finite inputs exclude missing-value and
signed-signal conventions. The statistic discards the winning window's
position and every other temporal detail; it does not normalize values, fit a
model, test several windows, or decide whether the result should become a
feature.

## Related Snippets

<!-- catalog:related:start -->
- [Detect a Recent Drop Against a Disjoint pandas Baseline Window](detect-a-recent-drop-against-a-disjoint-pandas-baseline-window.md)
- [Fit PCA with NumPy and Report Cumulative Explained Variance](fit-pca-with-numpy-and-report-cumulative-explained-variance.md)
- [Fan Out Events into Bounded Lookback Windows](../data-processing/fan-out-events-into-bounded-lookback-windows.md)
<!-- catalog:related:end -->
