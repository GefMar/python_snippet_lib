---
title: "Evaluate a Bounded Gaussian Kernel Density on an Explicit Grid"
snippet_type: recipe
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - calculate-a-symmetrically-trimmed-mean.md
  - fit-and-apply-fixed-quantile-clipping-bounds.md
  - compute-a-bounded-numerically-stable-log-sum-exp-for-finite-floats.md
---

# Evaluate a Bounded Gaussian Kernel Density on an Explicit Grid

## Idea and Problem

Evaluate a one-dimensional Gaussian kernel density estimate at bounded caller-supplied grid points with an explicit bandwidth.

`statistics.kde` turns observations into a callable density estimate. Keeping
the normal kernel fixed and requiring the bandwidth from the caller makes the
statistical assumption visible. Separate sample, grid, magnitude, and product
limits bound the full-support work before any density values are computed.

## When to Use

Use this helper for small exploratory summaries, deterministic plots, or tests
where observations and evaluation points are finite floats and a bandwidth has
already been chosen deliberately. Grid order and duplicate points are
preserved, so the result can be aligned directly with caller-owned labels or
plot coordinates.

Kernel density estimation smooths observations; it does not recover the true
distribution. Use a statistical library with a documented modeling workflow
when bandwidth selection, sample weights, multivariate data, boundary
correction, confidence intervals, or vectorized large-data evaluation matter.

## Implementation

```python
import math
import statistics
from dataclasses import dataclass

_MAX_SAMPLE_COUNT = 512
_MAX_GRID_COUNT = 512
_MAX_EVALUATIONS = 65_536
_MAX_MAGNITUDE = 1e12
_MIN_BANDWIDTH = 1e-12
_MAX_BANDWIDTH = 1e12


@dataclass(frozen=True, slots=True)
class DensityPoint:
    x: float
    density: float


def _validate_float_tuple(
    values: tuple[float, ...],
    *,
    name: str,
    maximum_count: int,
) -> None:
    if type(values) is not tuple:
        raise TypeError(f"{name} must be an exact tuple")
    if not 1 <= len(values) <= maximum_count:
        raise ValueError(f"{name} count is outside the supported range")
    for value in values:
        if type(value) is not float:
            raise TypeError(f"{name} must contain exact floats")
        if not math.isfinite(value):
            raise ValueError(f"{name} must contain finite values")
        if abs(value) > _MAX_MAGNITUDE:
            raise ValueError(f"{name} value exceeds the supported magnitude")


def evaluate_gaussian_kde(
    sample: tuple[float, ...],
    grid: tuple[float, ...],
    bandwidth: float,
) -> tuple[DensityPoint, ...]:
    _validate_float_tuple(
        sample,
        name="sample",
        maximum_count=_MAX_SAMPLE_COUNT,
    )
    _validate_float_tuple(
        grid,
        name="grid",
        maximum_count=_MAX_GRID_COUNT,
    )
    if type(bandwidth) is not float:
        raise TypeError("bandwidth must be an exact float")
    if not math.isfinite(bandwidth):
        raise ValueError("bandwidth must be finite")
    if not _MIN_BANDWIDTH <= bandwidth <= _MAX_BANDWIDTH:
        raise ValueError("bandwidth is outside the supported range")
    if len(sample) * len(grid) > _MAX_EVALUATIONS:
        raise ValueError("sample and grid exceed the supported evaluation count")

    estimate = statistics.kde(
        sample,
        h=bandwidth,
        kernel="normal",
        cumulative=False,
    )
    points: list[DensityPoint] = []
    for x in grid:
        density = estimate(x)
        if not math.isfinite(density) or density < 0.0:
            raise ArithmeticError("kernel density produced an invalid value")
        points.append(DensityPoint(x=x, density=density))
    return tuple(points)
```

## Example

```python
bandwidth = 2.0
points = evaluate_gaussian_kde(
    sample=(0.0,),
    grid=(-bandwidth, 0.0, bandwidth, 0.0),
    bandwidth=bandwidth,
)

expected_center = 1.0 / (bandwidth * math.sqrt(2.0 * math.pi))
expected_one_bandwidth = expected_center * math.exp(-0.5)

assert tuple(point.x for point in points) == (-2.0, 0.0, 2.0, 0.0)
assert math.isclose(points[1].density, expected_center, rel_tol=1e-15)
assert math.isclose(points[3].density, expected_center, rel_tol=1e-15)
assert math.isclose(points[0].density, expected_one_bandwidth, rel_tol=1e-15)
assert math.isclose(points[2].density, expected_one_bandwidth, rel_tol=1e-15)

boundary = evaluate_gaussian_kde(
    sample=(0.0,) * 256,
    grid=(0.0,) * 256,
    bandwidth=1.0,
)
assert len(boundary) == 256

try:
    evaluate_gaussian_kde(
        sample=(0.0,) * 257,
        grid=(0.0,) * 256,
        bandwidth=1.0,
    )
except ValueError:
    pass
else:
    raise AssertionError("sample-grid product must be bounded")

assert boundary[0].density > 0.0
```

## Trade-offs and Limitations

For `S` observations and `G` grid points, the fixed full-support normal kernel
uses `O(S * G)` time and `O(S + G)` retained state, including the admitted
sample reference and returned grid-aligned records. The product limit is the
direct cap on density contributions.

The magnitude and bandwidth bounds keep the largest normalized distance in a
range where squaring it remains finite. Very distant observations can still
underflow to zero contribution, which is valid for this estimate. The helper
rejects non-finite or negative derived densities but does not quantify
floating-point error.

The bandwidth controls smoothing and is not selected or validated against the
scientific purpose of the sample. The function fixes a Gaussian kernel, returns
a PDF estimate rather than a CDF or sampler, and does not provide weights,
multivariate support, boundary correction, uncertainty estimates, or a claim
that the estimate is the data-generating distribution.

## Related Snippets

<!-- catalog:related:start -->
- [Calculate a Symmetrically Trimmed Mean](calculate-a-symmetrically-trimmed-mean.md)
- [Fit and Apply Fixed Quantile Clipping Bounds](fit-and-apply-fixed-quantile-clipping-bounds.md)
- [Compute a Bounded Numerically Stable Log-Sum-Exp for Finite Floats](compute-a-bounded-numerically-stable-log-sum-exp-for-finite-floats.md)
<!-- catalog:related:end -->
