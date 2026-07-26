---
title: "Check a Value Against an Asymmetric Tolerance Band"
snippet_type: algorithm
use_cases:
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../machine-learning-statistics/calculate-a-symmetrically-trimmed-mean.md
  - ../observability-operations/count-values-in-fixed-upper-bound-bins.md
---

# Check a Value Against an Asymmetric Tolerance Band

## Idea and Problem

Validate one finite observation against independently sized lower and upper margins around a finite reference value.

Each margin is the larger of an absolute tolerance and a relative tolerance
scaled by the reference magnitude. Separate downward and upward ratios express
asymmetric policy, while inclusive boundaries make the exact acceptance rule
explicit for positive, negative, and zero references.

## When to Use

Use this check when a caller already owns a deterministic point-to-point
tolerance policy and needs a small validation boundary. Express ratios as
fractions, so `0.1` means ten percent, and choose an absolute tolerance when a
zero or near-zero reference still needs a nonzero band. Use statistical anomaly
detection when variability must be learned from a series rather than configured.

## Implementation

```python
import math
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class NumericBand:
    lower: float
    upper: float

    def __post_init__(self) -> None:
        lower = _finite_number(self.lower, name="lower")
        upper = _finite_number(self.upper, name="upper")
        if lower > upper:
            raise ValueError("lower must not exceed upper")
        object.__setattr__(self, "lower", lower)
        object.__setattr__(self, "upper", upper)

    def contains(self, value: int | float) -> bool:
        observed = _finite_number(value, name="value")
        return self.lower <= observed <= self.upper


def _finite_number(value: int | float, *, name: str) -> float:
    if isinstance(value, bool) or not isinstance(value, (int, float)):
        raise TypeError(f"{name} must be an integer or float")
    try:
        converted = float(value)
    except OverflowError as error:
        raise ValueError(f"{name} must be representable as a finite float") from error
    if not math.isfinite(converted):
        raise ValueError(f"{name} must be finite")
    return converted


def asymmetric_tolerance_band(
    reference: int | float,
    *,
    downward_ratio: int | float,
    upward_ratio: int | float,
    absolute_tolerance: int | float = 0.0,
) -> NumericBand:
    reference_value = _finite_number(reference, name="reference")
    downward = _finite_number(downward_ratio, name="downward_ratio")
    upward = _finite_number(upward_ratio, name="upward_ratio")
    absolute = _finite_number(absolute_tolerance, name="absolute_tolerance")
    if downward < 0 or upward < 0 or absolute < 0:
        raise ValueError("tolerances must be non-negative")

    scale = abs(reference_value)
    lower_margin = max(absolute, scale * downward)
    upper_margin = max(absolute, scale * upward)
    lower = reference_value - lower_margin
    upper = reference_value + upper_margin
    if not math.isfinite(lower) or not math.isfinite(upper):
        raise OverflowError("tolerance band is not representable as finite floats")
    return NumericBand(lower=lower, upper=upper)


def is_within_asymmetric_tolerance(
    value: int | float,
    *,
    reference: int | float,
    downward_ratio: int | float,
    upward_ratio: int | float,
    absolute_tolerance: int | float = 0.0,
) -> bool:
    band = asymmetric_tolerance_band(
        reference,
        downward_ratio=downward_ratio,
        upward_ratio=upward_ratio,
        absolute_tolerance=absolute_tolerance,
    )
    return band.contains(value)
```

## Example

```python
positive = asymmetric_tolerance_band(
    100,
    downward_ratio=0.1,
    upward_ratio=0.2,
    absolute_tolerance=2,
)
negative = asymmetric_tolerance_band(
    -100,
    downward_ratio=0.1,
    upward_ratio=0.2,
)
near_zero = asymmetric_tolerance_band(
    0,
    downward_ratio=0.1,
    upward_ratio=0.2,
    absolute_tolerance=0.5,
)

try:
    asymmetric_tolerance_band(
        1,
        downward_ratio=float("nan"),
        upward_ratio=0,
    )
except ValueError:
    non_finite_rejected = True
else:
    non_finite_rejected = False

assert (
    positive,
    positive.contains(90),
    positive.contains(120),
    positive.contains(121),
    negative,
    near_zero,
    near_zero.contains(-0.5),
    non_finite_rejected,
) == (
    NumericBand(90.0, 120.0),
    True,
    True,
    False,
    NumericBand(-110.0, -80.0),
    NumericBand(-0.5, 0.5),
    True,
    True,
)
```

## Trade-offs and Limitations

Relative margins use the reference magnitude, and the absolute tolerance is a
floor via `max`, not an additional amount. Ratios above one are allowed because
some domains intentionally permit changes larger than the baseline. This is a
single-point deterministic rule, not a confidence interval, trend detector, or
missing-reference policy. Floating-point rounding can affect values extremely
close to a boundary; use `Decimal` or integer minor units when exact business
arithmetic is required. Configuration lookup and per-key overrides should stay
outside this numeric primitive.

## Related Snippets

<!-- catalog:related:start -->
- [Calculate a Symmetrically Trimmed Mean](../machine-learning-statistics/calculate-a-symmetrically-trimmed-mean.md)
- [Count Values in Fixed Upper-Bound Bins](../observability-operations/count-values-in-fixed-upper-bound-bins.md)
<!-- catalog:related:end -->
