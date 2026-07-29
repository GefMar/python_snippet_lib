---
title: "Shift an Exact Date by Whole Calendar Months with Explicit Day Overflow"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../configuration-serialization/render-fixed-date-placeholders-from-an-explicit-anchor.md
  - ../configuration-serialization/parse-compact-durations-into-timedelta.md
  - ../storage-databases/select-snapshot-representatives-by-utc-calendar-buckets.md
---

# Shift an Exact Date by Whole Calendar Months with Explicit Day Overflow

## Idea and Problem

Shift a calendar date by an exact number of whole months while making an invalid target day an explicit policy decision.

Months do not have a fixed elapsed duration. Moving January 31 by one month,
for example, identifies February before deciding what to do with day 31. This
algorithm either clamps that day to the target month's last valid day or
returns `None`; it never silently substitutes elapsed-day arithmetic.

## When to Use

Use this operation when a domain rule is expressed in whole calendar months
and the caller can choose one of two closed day-overflow policies. It suits
bounded billing anchors, review dates, and other date-only transformations in
which preserving the original day is preferred whenever that day exists.

Choose the policy at the domain boundary. `CLAMP` is useful when every
representable target month needs a date, while `RETURN_NONE` makes an
unrepresentable day visible to a caller that must decide separately. Use a
calendar or recurrence library when business-day, time-zone, or repeated-event
semantics are part of the contract.

## Implementation

```python
from calendar import monthrange
from datetime import date
from enum import Enum

_MIN_MONTH_DELTA = -1_200
_MAX_MONTH_DELTA = 1_200
_MONTHS_PER_YEAR = 12
_MAX_MONTH_INDEX = 9_999 * _MONTHS_PER_YEAR - 1


class DayOverflowPolicy(Enum):
    CLAMP = "clamp"
    RETURN_NONE = "return-none"


def shift_date_by_calendar_months(
    value: date,
    *,
    months: int,
    overflow_policy: DayOverflowPolicy,
) -> date | None:
    """Shift one exact date under a closed target-day overflow policy."""
    if type(value) is not date:
        raise TypeError("value must be an exact date")
    if type(months) is not int:
        raise TypeError("months must be an exact non-boolean integer")
    if not _MIN_MONTH_DELTA <= months <= _MAX_MONTH_DELTA:
        raise ValueError("months is outside the supported range")
    if type(overflow_policy) is not DayOverflowPolicy:
        raise TypeError("overflow_policy must be an exact DayOverflowPolicy")

    source_month_index = (value.year - 1) * _MONTHS_PER_YEAR + value.month - 1
    target_month_index = source_month_index + months
    if not 0 <= target_month_index <= _MAX_MONTH_INDEX:
        raise ValueError("target month is outside the representable date range")

    target_year_index, target_month_index_in_year = divmod(
        target_month_index,
        _MONTHS_PER_YEAR,
    )
    target_year = target_year_index + 1
    target_month = target_month_index_in_year + 1
    target_last_day = monthrange(target_year, target_month)[1]

    if value.day <= target_last_day:
        target_day = value.day
    elif overflow_policy is DayOverflowPolicy.RETURN_NONE:
        return None
    else:
        target_day = target_last_day

    return date(target_year, target_month, target_day)
```

## Example

```python
def oracle_shift_date(
    value: date,
    months: int,
    overflow_policy: DayOverflowPolicy,
) -> date | None:
    year = value.year
    month = value.month
    direction = 1 if months >= 0 else -1

    for _ in range(abs(months)):
        month += direction
        if month == 13:
            year += 1
            month = 1
        elif month == 0:
            year -= 1
            month = 12
        if not 1 <= year <= 9_999:
            raise ValueError("target year is not representable")

    for target_day in range(value.day, 0, -1):
        try:
            candidate = date(year, month, target_day)
        except ValueError:
            continue
        if target_day != value.day and (overflow_policy is DayOverflowPolicy.RETURN_NONE):
            return None
        return candidate
    raise AssertionError("every calendar month has a valid day")


def datetime_value() -> date:
    from datetime import datetime

    return datetime(2024, 1, 31)


cases = (
    (date(2024, 1, 31), 1),
    (date(2023, 1, 31), 1),
    (date(2024, 3, 31), -1),
    (date(2024, 2, 29), 12),
    (date(2024, 12, 31), -12),
    (date(1, 12, 31), -11),
    (date(9999, 1, 31), 11),
    (date(101, 1, 31), -1200),
    (date(9899, 12, 31), 1200),
)
for source, month_delta in cases:
    for policy in DayOverflowPolicy:
        assert shift_date_by_calendar_months(
            source,
            months=month_delta,
            overflow_policy=policy,
        ) == oracle_shift_date(source, month_delta, policy)

rejected_year_edges = 0
for source, month_delta in (
    (date(1, 1, 1), -1),
    (date(9999, 12, 31), 1),
    (date(100, 1, 1), -1200),
    (date(9900, 1, 1), 1200),
):
    try:
        shift_date_by_calendar_months(
            source,
            months=month_delta,
            overflow_policy=DayOverflowPolicy.CLAMP,
        )
    except ValueError:
        rejected_year_edges += 1

try:
    shift_date_by_calendar_months(
        datetime_value(),
        months=1,
        overflow_policy=DayOverflowPolicy.CLAMP,
    )
except TypeError:
    datetime_rejected = True
else:
    datetime_rejected = False

direct = shift_date_by_calendar_months(
    date(2024, 1, 31),
    months=2,
    overflow_policy=DayOverflowPolicy.CLAMP,
)
after_one_month = shift_date_by_calendar_months(
    date(2024, 1, 31),
    months=1,
    overflow_policy=DayOverflowPolicy.CLAMP,
)
assert after_one_month is not None
repeated = shift_date_by_calendar_months(
    after_one_month,
    months=1,
    overflow_policy=DayOverflowPolicy.CLAMP,
)

assert (direct, repeated, rejected_year_edges, datetime_rejected) == (
    date(2024, 3, 31),
    date(2024, 3, 29),
    4,
    True,
)
```

## Trade-offs and Limitations

The month-index calculation, month-length lookup, and date construction use
`O(1)` time and memory. The input delta is deliberately bounded to 100 years in
either direction, and a target before year 1 or after year 9999 is rejected
rather than wrapped or saturated.

Clamping loses the original day when the target month is shorter. Consequently,
repeated clamped shifts are not associative: clamping January 31 to February 29
and then shifting again produces March 29, while a direct two-month shift
produces March 31. `RETURN_NONE` avoids that adjustment but requires the caller
to handle the missing result.

The function handles dates only. It does not define elapsed durations, times,
time zones, daylight-saving transitions, business calendars, fractional
months, holiday adjustment, or recurrence scheduling.

## Related Snippets

<!-- catalog:related:start -->
- [Render Fixed Date Placeholders from an Explicit Anchor](../configuration-serialization/render-fixed-date-placeholders-from-an-explicit-anchor.md)
- [Parse Compact Durations into timedelta](../configuration-serialization/parse-compact-durations-into-timedelta.md)
- [Select Snapshot Representatives by UTC Calendar Buckets](../storage-databases/select-snapshot-representatives-by-utc-calendar-buckets.md)
<!-- catalog:related:end -->
