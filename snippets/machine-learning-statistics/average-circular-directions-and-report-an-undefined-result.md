---
title: "Average Circular Directions and Report an Undefined Result"
snippet_type: recipe
use_cases:
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../algorithms-data-structures/measure-the-unsigned-angle-between-two-planar-vectors.md
  - ../data-processing/normalize-a-finite-binary64-value-into-a-canonical-half-open-period.md
---

# Average Circular Directions and Report an Undefined Result

## Idea and Problem

Average directions without introducing a false midpoint where angles wrap around zero.

Convert each direction to a unit vector, sum the components, and convert the resulting direction back to degrees. Detect cancellation before interpreting a nearly zero vector as an angle.

## When to Use

Use this for equally weighted bearings in the half-open interval from zero to 360 degrees. Decide whether the mean direction is useful separately from how tightly the sample is concentrated.

## Implementation

```python
from collections.abc import Iterable
from math import atan2, cos, degrees, fsum, hypot, radians, sin


def average_direction(angles: Iterable[float]) -> float | None:
    values = tuple(angles)
    if not values:
        return None
    if any(not 0.0 <= angle < 360.0 for angle in values):
        raise ValueError("every direction must be in the interval [0, 360)")

    x = fsum(cos(radians(angle)) for angle in values)
    y = fsum(sin(radians(angle)) for angle in values)
    if hypot(x, y) <= 1e-12 * len(values):
        return None
    result = degrees(atan2(y, x)) % 360.0
    return 0.0 if result == 360.0 else result
```

## Example

```python
from math import isclose

around_zero = average_direction((350.0, 10.0))
quarter = average_direction((0.0, 90.0))
assert (
    around_zero is not None and min(around_zero, 360.0 - around_zero) < 1e-12,
    quarter is not None and isclose(quarter, 45.0),
    average_direction((0.0, 180.0)),
    average_direction(()),
) == (True, True, None, None)
```

## Trade-offs and Limitations

This stores a finite input in memory and gives no confidence interval or multimodality diagnostic. The small resultant cutoff is an explicit numerical policy, not a universal significance threshold. Averaging orientations where opposite directions should be considered equivalent requires a different calculation.

## Related Snippets

<!-- catalog:related:start -->
- [Measure the Unsigned Angle Between Two Planar Vectors](../algorithms-data-structures/measure-the-unsigned-angle-between-two-planar-vectors.md)
- [Normalize a Finite Binary64 Value into a Canonical Half-Open Period](../data-processing/normalize-a-finite-binary64-value-into-a-canonical-half-open-period.md)
<!-- catalog:related:end -->
