---
title: "Schedule the Next Review from Outcome and Bounded Coverage"
snippet_type: algorithm
use_cases:
  - automation
  - retry-recovery
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - assign-stable-schedule-slots-with-a-digest.md
  - hold-a-switch-active-through-a-monotonic-cooldown.md
  - ../observability-operations/classify-required-health-stamps-by-freshness.md
---

# Schedule the Next Review from Outcome and Bounded Coverage

## Idea and Problem

Choose a bounded whole-day review interval from an explicit outcome and the fraction of a planned scan that actually completed.

A fixed interval ignores how much evidence the previous review collected. This
helper gives each outcome its own caller-defined interval window and uses a
documented square-root curve to interpolate within that window. The result is
a date only: storage, status changes, queueing, and retries remain outside the
policy function.

## When to Use

Use this policy when reviews occur on calendar dates, completed coverage is a
useful scheduling signal, and the application can define suitable minimum and
maximum delays for every outcome. A coverage of zero chooses the minimum delay;
complete coverage chooses the maximum. Intermediate coverage lengthens the
delay quickly at first and then tapers because the square-root curve is
concave.

Treat the curve and windows as operational policy, not statistical confidence
or a risk score. Use a measured risk model when the schedule must represent a
probability of failure, and use a time-zone-aware `datetime` policy when the
exact execution instant matters.

## Implementation

```python
import math
from dataclasses import dataclass
from datetime import date, timedelta
from enum import StrEnum


_MAX_PLANNED_ITEMS = 1_000_000
_MAX_INTERVAL_DAYS = 3_650


class ReviewOutcome(StrEnum):
    CLEAR = "clear"
    WARNING = "warning"


@dataclass(frozen=True, slots=True)
class DayWindow:
    minimum: int
    maximum: int


@dataclass(frozen=True, slots=True)
class ReviewPolicy:
    clear: DayWindow
    warning: DayWindow


def _validate_window(name: str, window: object) -> DayWindow:
    if not isinstance(window, DayWindow):
        raise TypeError(f"{name} must be a DayWindow")
    for field_name, value in (
        ("minimum", window.minimum),
        ("maximum", window.maximum),
    ):
        if isinstance(value, bool) or not isinstance(value, int):
            raise TypeError(f"{name}.{field_name} must be an integer")
    if not 1 <= window.minimum <= window.maximum <= _MAX_INTERVAL_DAYS:
        raise ValueError(f"{name} is outside the supported interval range")
    return window


def schedule_next_review(
    observed_on: date,
    outcome: ReviewOutcome,
    *,
    checked: int,
    planned: int,
    policy: ReviewPolicy,
) -> date:
    if type(observed_on) is not date:
        raise TypeError("observed_on must be a date, not a datetime")
    if not isinstance(outcome, ReviewOutcome):
        raise TypeError("outcome must be a ReviewOutcome")
    for name, value in (("checked", checked), ("planned", planned)):
        if isinstance(value, bool) or not isinstance(value, int):
            raise TypeError(f"{name} must be an integer")
    if not 1 <= planned <= _MAX_PLANNED_ITEMS:
        raise ValueError("planned is outside the supported range")
    if not 0 <= checked <= planned:
        raise ValueError("checked must be between zero and planned")
    if not isinstance(policy, ReviewPolicy):
        raise TypeError("policy must be a ReviewPolicy")

    clear = _validate_window("clear", policy.clear)
    warning = _validate_window("warning", policy.warning)
    window = clear if outcome is ReviewOutcome.CLEAR else warning

    coverage = checked / planned
    unrounded_days = (
        window.minimum
        + (window.maximum - window.minimum) * math.sqrt(coverage)
    )
    delay_days = math.ceil(unrounded_days)
    try:
        return observed_on + timedelta(days=delay_days)
    except OverflowError as error:
        raise ValueError("the next review date is outside the supported range") from error
```

## Example

```python
from datetime import date


policy = ReviewPolicy(
    clear=DayWindow(minimum=30, maximum=90),
    warning=DayWindow(minimum=2, maximum=14),
)

partial_warning = schedule_next_review(
    date(2026, 1, 10),
    ReviewOutcome.WARNING,
    checked=25,
    planned=100,
    policy=policy,
)
complete_clear = schedule_next_review(
    date(2026, 1, 10),
    ReviewOutcome.CLEAR,
    checked=100,
    planned=100,
    policy=policy,
)

assert (partial_warning, complete_clear) == (
    date(2026, 1, 18),
    date(2026, 4, 10),
)
```

## Trade-offs and Limitations

Square-root interpolation is easy to explain but still arbitrary; changing the
curve or window changes operational load and should be reviewed as a policy
change. Ceiling fractional days biases the result toward a later date, while a
different rounding rule would produce different boundary behavior. Coverage
also says nothing about sampling quality, severity, or whether checked items
were representative.

The helper supports two outcomes deliberately. Extend the immutable policy
model when a domain has more stable outcome classes, rather than accepting
free-form labels or hidden defaults. It does not persist the date, deduplicate
scheduled work, add jitter, or prevent many records from selecting the same
day.

## Related Snippets

<!-- catalog:related:start -->
- [Assign Stable Schedule Slots with a Digest](assign-stable-schedule-slots-with-a-digest.md)
- [Hold a Switch Active Through a Monotonic Cooldown](hold-a-switch-active-through-a-monotonic-cooldown.md)
- [Classify Required Health Stamps by Freshness](../observability-operations/classify-required-health-stamps-by-freshness.md)
<!-- catalog:related:end -->
