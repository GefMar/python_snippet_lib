---
title: "Encode Cyclic Positions with Sine and Cosine"
snippet_type: algorithm
use_cases:
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../algorithms-data-structures/map-points-between-rectangular-coordinate-spaces.md
  - ../python-language/model-a-quantity-with-one-canonical-unit.md
---

# Encode Cyclic Positions with Sine and Cosine

## Idea and Problem

Map a position on a known cycle to sine and cosine coordinates so the cycle has no artificial numeric discontinuity.

On a 24-position clock, positions 23 and 0 are neighbors even though their
ordinary numeric difference is large. Wrapping the position and projecting it
onto the unit circle preserves that adjacency in two numeric features.

## When to Use

Use this encoding when a model or distance calculation receives a genuinely
periodic scalar with a fixed positive period and does not otherwise learn the
wraparound relationship. Define the position and period in the same units.
Keep calendar, timezone, missing-value, and feature-selection policy outside
the helper.

## Implementation

```python
import math


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


def encode_cyclic_position(
    position: int | float,
    *,
    period: int | float,
) -> tuple[float, float]:
    position_value = _finite_number(position, name="position")
    period_value = _finite_number(period, name="period")
    if period_value <= 0:
        raise ValueError("period must be positive")

    wrapped = position_value % period_value
    angle = math.tau * (wrapped / period_value)
    return math.sin(angle), math.cos(angle)
```

## Example

```python
cardinal = tuple(
    tuple(round(component, 12) for component in encode_cyclic_position(hour, period=24))
    for hour in (0, 6, 12, 18)
)
wrapped = encode_cyclic_position(-1, period=24)
previous_hour = encode_cyclic_position(23, period=24)
midnight = encode_cyclic_position(0, period=24)
noon = encode_cyclic_position(12, period=24)
large_finite = encode_cyclic_position(9e307, period=1e308)

unit_circle = math.isclose(
    wrapped[0] ** 2 + wrapped[1] ** 2,
    1.0,
    rel_tol=1e-12,
)

assert (
    cardinal,
    all(
        math.isclose(left, right, abs_tol=1e-12)
        for left, right in zip(wrapped, previous_hour)
    ),
    math.dist(previous_hour, midnight) < math.dist(noon, midnight),
    unit_circle,
    all(math.isfinite(component) for component in large_finite),
) == (
    ((0.0, 1.0), (1.0, 0.0), (0.0, -1.0), (-1.0, -0.0)),
    True,
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

The result uses floating-point trigonometry and should be compared with a
tolerance. One pair represents only the fundamental cycle; more complex
periodic shapes may need higher harmonics or a different model. The transform
does not prove that a feature is predictive. Calendar months are unequal in
elapsed time, daylight-saving rules can change local-hour meaning, and callers
must normalize those domain semantics before choosing a position and period.
Very large floats may also lose the low-order information needed for accurate
modulo reduction.

## Related Snippets

<!-- catalog:related:start -->
- [Map Points Between Rectangular Coordinate Spaces](../algorithms-data-structures/map-points-between-rectangular-coordinate-spaces.md)
- [Model a Quantity with One Canonical Unit](../python-language/model-a-quantity-with-one-canonical-unit.md)
<!-- catalog:related:end -->
