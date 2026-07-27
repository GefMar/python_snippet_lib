---
title: "Classify a Finite Value with Warning and Critical Limits"
snippet_type: recipe
use_cases:
  - observability
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - count-values-in-fixed-upper-bound-bins.md
  - ../data-processing/check-a-value-against-an-asymmetric-tolerance-band.md
---

# Classify a Finite Value with Warning and Critical Limits

## Idea and Problem

Classify one finite float as ok, warning, or critical using an explicit direction and strictly ordered inclusive limits.

Checking the critical condition first makes values exactly on the critical
limit unambiguous. Higher-is-worse policies require the warning limit below the
critical limit; lower-is-worse policies require the reverse ordering.

## When to Use

Use this recipe when one already measured value has two fixed policy limits and
larger or smaller values consistently represent greater severity. Supply limits
in the same unit as the observation and choose the direction at the call site
instead of encoding it through reordered comparisons.

Use a stateful policy when transitions require hysteresis, consecutive samples,
cooldowns, or missing-data handling. Validate and convert textual configuration
before calling this numeric boundary.

## Implementation

```python
import math
from enum import StrEnum


class Direction(StrEnum):
    HIGHER_IS_WORSE = "higher_is_worse"
    LOWER_IS_WORSE = "lower_is_worse"


class Severity(StrEnum):
    OK = "ok"
    WARNING = "warning"
    CRITICAL = "critical"


def _exact_finite_float(value: object, *, name: str) -> float:
    if type(value) is not float:
        raise TypeError(f"{name} must be an exact float")
    if not math.isfinite(value):
        raise ValueError(f"{name} must be finite")
    return value


def classify_by_limits(
    value: float,
    *,
    warning_limit: float,
    critical_limit: float,
    direction: Direction,
) -> Severity:
    observed = _exact_finite_float(value, name="value")
    warning = _exact_finite_float(warning_limit, name="warning_limit")
    critical = _exact_finite_float(critical_limit, name="critical_limit")
    if type(direction) is not Direction:
        raise TypeError("direction must be an exact Direction")

    if direction is Direction.HIGHER_IS_WORSE:
        if warning >= critical:
            raise ValueError("warning_limit must be below critical_limit")
        if observed >= critical:
            return Severity.CRITICAL
        if observed >= warning:
            return Severity.WARNING
    else:
        if critical >= warning:
            raise ValueError("critical_limit must be below warning_limit")
        if observed <= critical:
            return Severity.CRITICAL
        if observed <= warning:
            return Severity.WARNING
    return Severity.OK
```

## Example

```python
higher = tuple(
    classify_by_limits(
        value,
        warning_limit=7.5,
        critical_limit=9.0,
        direction=Direction.HIGHER_IS_WORSE,
    )
    for value in (7.49, 7.5, 9.0)
)
lower = tuple(
    classify_by_limits(
        value,
        warning_limit=4.0,
        critical_limit=2.5,
        direction=Direction.LOWER_IS_WORSE,
    )
    for value in (4.01, 4.0, 2.5)
)

try:
    classify_by_limits(
        float("nan"),
        warning_limit=1.0,
        critical_limit=2.0,
        direction=Direction.HIGHER_IS_WORSE,
    )
except ValueError:
    non_finite_rejected = True
else:
    non_finite_rejected = False

try:
    classify_by_limits(
        1.0,
        warning_limit=2.0,
        critical_limit=2.0,
        direction=Direction.HIGHER_IS_WORSE,
    )
except ValueError:
    unordered_limits_rejected = True
else:
    unordered_limits_rejected = False

try:
    classify_by_limits(
        1,
        warning_limit=2.0,
        critical_limit=3.0,
        direction=Direction.HIGHER_IS_WORSE,
    )
except TypeError:
    integer_rejected = True
else:
    integer_rejected = False

assert (
    higher,
    lower,
    non_finite_rejected,
    unordered_limits_rejected,
    integer_rejected,
) == (
    (Severity.OK, Severity.WARNING, Severity.CRITICAL),
    (Severity.OK, Severity.WARNING, Severity.CRITICAL),
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

Classification takes constant time and returns an immutable enum without
mutating caller data. Strict float inputs prevent implicit boolean, integer,
decimal, or unit conversion, but callers must perform intentional conversion
before this boundary. Floating-point rounding still matters for computed values
very close to a limit.

The recipe evaluates one sample independently. It does not fetch configuration,
attach labels, emit alerts, debounce repeated states, model trends, or define a
policy for absent measurements. Those behaviors require separate state and
delivery layers.

## Related Snippets

<!-- catalog:related:start -->
- [Count Values in Fixed Upper-Bound Bins](count-values-in-fixed-upper-bound-bins.md)
- [Check a Value Against an Asymmetric Tolerance Band](../data-processing/check-a-value-against-an-asymmetric-tolerance-band.md)
<!-- catalog:related:end -->
