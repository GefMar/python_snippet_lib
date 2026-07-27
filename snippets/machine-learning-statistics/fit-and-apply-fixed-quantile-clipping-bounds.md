---
title: "Fit and Apply Fixed Quantile Clipping Bounds"
snippet_type: recipe
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - flag-groupwise-numeric-outliers-with-iqr-fences.md
  - calculate-a-symmetrically-trimmed-mean.md
  - ../data-processing/check-a-value-against-an-asymmetric-tolerance-band.md
---

# Fit and Apply Fixed Quantile Clipping Bounds

## Idea and Problem

Fit immutable quantile bounds once from training values, then clip future values to those same bounds without mutating either sequence or refitting.

The fitted quantiles use an inclusive linear rule. For a sorted training
sequence of length `n`, quantile `q` is evaluated at position `q * (n - 1)` and
linearly interpolated between adjacent values. Quantiles zero and one therefore
return the minimum and maximum. A singleton or an all-equal sequence produces
equal lower and upper bounds.

## When to Use

Use this recipe when a bounded, finite training sample defines a deliberate
clipping policy that must remain fixed while processing later bounded batches.
Choose lower and upper quantiles in the inclusive range from zero to one before
examining future values. The immutable bounds record the quantiles and training
size so the fitted policy remains inspectable.

Applying the bounds returns a new tuple in the original future-value order.
It neither changes the supplied values nor recalculates bounds, which makes the
training-versus-application boundary explicit.

## Implementation

```python
import math
from collections.abc import Iterable
from dataclasses import dataclass
from itertools import islice
from numbers import Real


_MAX_VALUES = 10_000


def _bounded_finite_values(
    values: Iterable[Real],
    *,
    name: str,
    require_non_empty: bool,
) -> tuple[float, ...]:
    if isinstance(values, (str, bytes)):
        raise TypeError(f"{name} must be a non-text iterable")
    snapshot = tuple(islice(values, _MAX_VALUES + 1))
    if len(snapshot) > _MAX_VALUES:
        raise ValueError(f"{name} contains too many values")
    if require_non_empty and not snapshot:
        raise ValueError(f"{name} must not be empty")

    normalized: list[float] = []
    for value in snapshot:
        if isinstance(value, bool) or not isinstance(value, Real):
            raise TypeError(f"{name} must contain real numbers")
        try:
            sample = float(value)
        except OverflowError as error:
            raise ValueError(f"{name} values must fit in finite floats") from error
        if not math.isfinite(sample):
            raise ValueError(f"{name} values must be finite")
        normalized.append(sample)
    return tuple(normalized)


def _quantile(value: Real, *, name: str) -> float:
    if isinstance(value, bool) or not isinstance(value, Real):
        raise TypeError(f"{name} must be a real number")
    try:
        result = float(value)
    except OverflowError as error:
        raise ValueError(f"{name} must be between zero and one") from error
    if not math.isfinite(result) or not 0.0 <= result <= 1.0:
        raise ValueError(f"{name} must be between zero and one")
    return result


def _inclusive_linear_quantile(sorted_values: list[float], q: float) -> float:
    position = q * (len(sorted_values) - 1)
    lower_index = math.floor(position)
    fraction = position - lower_index
    upper_index = min(lower_index + 1, len(sorted_values) - 1)
    return (
        (1.0 - fraction) * sorted_values[lower_index]
        + fraction * sorted_values[upper_index]
    )


@dataclass(frozen=True, slots=True)
class QuantileClippingBounds:
    lower_quantile: float
    upper_quantile: float
    lower: float
    upper: float
    training_size: int

    def __post_init__(self) -> None:
        if not all(
            type(value) is float and math.isfinite(value)
            for value in (
                self.lower_quantile,
                self.upper_quantile,
                self.lower,
                self.upper,
            )
        ):
            raise TypeError("quantiles and bounds must be finite floats")
        if not 0.0 <= self.lower_quantile <= self.upper_quantile <= 1.0:
            raise ValueError("quantiles must be ordered within zero and one")
        if self.lower > self.upper:
            raise ValueError("lower bound must not exceed upper bound")
        if type(self.training_size) is not int or self.training_size <= 0:
            raise ValueError("training_size must be a positive integer")


def fit_quantile_clipping_bounds(
    training_values: Iterable[Real],
    *,
    lower_quantile: Real,
    upper_quantile: Real,
) -> QuantileClippingBounds:
    lower_q = _quantile(lower_quantile, name="lower_quantile")
    upper_q = _quantile(upper_quantile, name="upper_quantile")
    if lower_q > upper_q:
        raise ValueError("lower_quantile must not exceed upper_quantile")

    ordered = sorted(
        _bounded_finite_values(
            training_values,
            name="training_values",
            require_non_empty=True,
        )
    )
    return QuantileClippingBounds(
        lower_quantile=lower_q,
        upper_quantile=upper_q,
        lower=_inclusive_linear_quantile(ordered, lower_q),
        upper=_inclusive_linear_quantile(ordered, upper_q),
        training_size=len(ordered),
    )


def apply_quantile_clipping_bounds(
    values: Iterable[Real],
    bounds: QuantileClippingBounds,
) -> tuple[float, ...]:
    if type(bounds) is not QuantileClippingBounds:
        raise TypeError("bounds must be QuantileClippingBounds")
    snapshot = _bounded_finite_values(
        values,
        name="values",
        require_non_empty=False,
    )
    return tuple(min(max(value, bounds.lower), bounds.upper) for value in snapshot)
```

## Example

```python
training = [0.0, 10.0, 20.0, 30.0]
future = [-5.0, 12.0, 40.0, 7.5]
bounds = fit_quantile_clipping_bounds(
    training,
    lower_quantile=0.25,
    upper_quantile=0.75,
)
clipped = apply_quantile_clipping_bounds(future, bounds)

singleton = fit_quantile_clipping_bounds(
    [4.0], lower_quantile=0.1, upper_quantile=0.9
)
all_equal = fit_quantile_clipping_bounds(
    [3.0, 3.0, 3.0], lower_quantile=0.2, upper_quantile=0.8
)
endpoints = fit_quantile_clipping_bounds(
    [9.0, 1.0, 5.0], lower_quantile=0.0, upper_quantile=1.0
)

assert (
    bounds,
    clipped,
    training,
    future,
    (singleton.lower, singleton.upper),
    (all_equal.lower, all_equal.upper),
    (endpoints.lower, endpoints.upper),
) == (
    QuantileClippingBounds(0.25, 0.75, 7.5, 22.5, 4),
    (7.5, 12.0, 22.5, 7.5),
    [0.0, 10.0, 20.0, 30.0],
    [-5.0, 12.0, 40.0, 7.5],
    (4.0, 4.0),
    (3.0, 3.0),
    (1.0, 9.0),
)
```

## Trade-offs and Limitations

Fitting materializes and sorts the training values in `O(n log n)` time and
`O(n)` memory; applying uses `O(n)` time and output memory. Float conversion can
lose precision. Quantile estimates from small samples are unstable, and fixed
bounds can become stale under distribution drift.

Clipping changes the data distribution and can flatten meaningful extremes.
It may hide rare events precisely when they matter, so retain raw data and
monitor how often each bound is reached. This recipe differs from a trimmed
mean, which discards tails to calculate one summary, and from groupwise IQR
fences, which diagnose observations relative to each group's spread. Do not use
fixed clipping as a substitute for domain-aware anomaly handling.

## Related Snippets

<!-- catalog:related:start -->
- [Flag Groupwise Numeric Outliers with IQR Fences](flag-groupwise-numeric-outliers-with-iqr-fences.md)
- [Calculate a Symmetrically Trimmed Mean](calculate-a-symmetrically-trimmed-mean.md)
- [Check a Value Against an Asymmetric Tolerance Band](../data-processing/check-a-value-against-an-asymmetric-tolerance-band.md)
<!-- catalog:related:end -->
