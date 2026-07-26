---
title: "Choose Nice Linear Axis Bounds and Tick Step"
snippet_type: algorithm
use_cases:
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - map-points-between-rectangular-coordinate-spaces.md
---

# Choose Nice Linear Axis Bounds and Tick Step

## Idea and Problem

Enclose a finite numeric range with linear-axis bounds and a readable tick step based on 1, 2, or 5 times a power of ten.

The requested interval count determines a raw step. Rounding that step upward
to the next `1/2/5 x 10^n` value prevents crowded ticks, while floor and ceiling
normally expand the endpoints to step multiples. A flat range receives
symmetric padding before the same calculation. The function rejects ranges
whose calculated step is too small to move an endpoint at its float magnitude.

## When to Use

Use this algorithm for simple linear charts when approximate tick density is
more important than a fixed number of ticks. Inputs must be finite floats whose
span and intermediate arithmetic remain representable. Keep label precision,
units, time axes, logarithmic scales, and domain-specific zero baselines in a
separate formatting or chart policy.

## Implementation

```python
import math
from dataclasses import dataclass
from numbers import Real


@dataclass(frozen=True, slots=True)
class LinearAxis:
    lower: float
    upper: float
    step: float


def _finite_axis_value(value: Real, *, name: str) -> float:
    if isinstance(value, bool) or not isinstance(value, Real):
        raise TypeError(f"{name} must be a real number")
    try:
        converted = float(value)
    except OverflowError as error:
        raise ValueError(f"{name} must fit in a finite float") from error
    if not math.isfinite(converted):
        raise ValueError(f"{name} must be finite")
    return converted


def _nice_step_at_least(raw_step: float) -> float:
    if not math.isfinite(raw_step) or raw_step <= 0.0:
        raise ValueError("range does not have a representable positive step")

    exponent = math.floor(math.log10(raw_step))
    magnitude = 10.0**exponent
    if magnitude == 0.0 or not math.isfinite(magnitude):
        raise ValueError("range scale is outside supported float decades")
    fraction = raw_step / magnitude
    if fraction <= 1.0:
        factor = 1.0
    elif fraction <= 2.0:
        factor = 2.0
    elif fraction <= 5.0:
        factor = 5.0
    else:
        factor = 10.0
    step = factor * magnitude
    if not math.isfinite(step) or step <= 0.0:
        raise ValueError("calculated step is not a finite positive float")
    return step


def choose_linear_axis(
    minimum: Real,
    maximum: Real,
    *,
    target_intervals: int = 5,
) -> LinearAxis:
    if isinstance(target_intervals, bool) or not isinstance(target_intervals, int):
        raise TypeError("target_intervals must be an integer")
    if target_intervals <= 0:
        raise ValueError("target_intervals must be positive")

    lower_input = _finite_axis_value(minimum, name="minimum")
    upper_input = _finite_axis_value(maximum, name="maximum")
    if lower_input > upper_input:
        raise ValueError("minimum must not exceed maximum")

    if lower_input == upper_input:
        padding = max(abs(lower_input) * 0.05, 1.0)
        working_lower = lower_input - padding
        working_upper = upper_input + padding
    else:
        working_lower = lower_input
        working_upper = upper_input

    span = working_upper - working_lower
    if not math.isfinite(span) or span <= 0.0:
        raise ValueError("range span is not a finite positive float")
    step = _nice_step_at_least(span / target_intervals)

    lower = math.floor(working_lower / step) * step
    upper = math.ceil(working_upper / step) * step
    if lower > working_lower:
        adjusted_lower = lower - step
        if adjusted_lower >= lower:
            raise ValueError("step cannot move the lower bound at this float scale")
        lower = adjusted_lower
    if upper < working_upper:
        adjusted_upper = upper + step
        if adjusted_upper <= upper:
            raise ValueError("step cannot move the upper bound at this float scale")
        upper = adjusted_upper
    if not math.isfinite(lower) or not math.isfinite(upper):
        raise ValueError("axis bounds are outside the finite float range")
    if lower > working_lower or upper < working_upper:
        raise ValueError("calculated float bounds do not enclose the requested range")
    if lower == 0.0:
        lower = 0.0
    if upper == 0.0:
        upper = 0.0
    return LinearAxis(lower=lower, upper=upper, step=step)
```

## Example

```python
positive = choose_linear_axis(93, 117, target_intervals=5)
negative = choose_linear_axis(-117, -93, target_intervals=5)
mixed = choose_linear_axis(-3, 7, target_intervals=5)
flat = choose_linear_axis(-10, -10, target_intervals=5)
small = choose_linear_axis(1.2e-6, 1.8e-6, target_intervals=5)

try:
    choose_linear_axis(2, 1)
except ValueError:
    reversed_range_rejected = True
else:
    reversed_range_rejected = False

try:
    choose_linear_axis(math.nextafter(1.0, -math.inf), 1.0)
except ValueError:
    unresolved_tick_rejected = True
else:
    unresolved_tick_rejected = False

assert (
    positive,
    negative,
    mixed,
    flat,
    small.lower <= 1.2e-6 <= 1.8e-6 <= small.upper,
    small.step > 0,
    reversed_range_rejected,
    unresolved_tick_rejected,
) == (
    LinearAxis(90.0, 120.0, 5.0),
    LinearAxis(-120.0, -90.0, 5.0),
    LinearAxis(-4.0, 8.0, 2.0),
    LinearAxis(-11.0, -9.0, 0.5),
    True,
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

The interval count is a target, not a guarantee, because both endpoints expand
to step multiples. Binary float arithmetic can leave tiny representation noise
for decimal-looking ticks; format labels separately. Extremely wide, tiny,
near-limit, or high-offset narrow ranges are rejected when the span, step, or
bounds cannot be represented distinctly and safely. The helper is only for
linear numeric axes and does not choose a semantic baseline or prevent
misleading visual scales.

## Related Snippets

<!-- catalog:related:start -->
- [Map Points Between Rectangular Coordinate Spaces](map-points-between-rectangular-coordinate-spaces.md)
<!-- catalog:related:end -->
