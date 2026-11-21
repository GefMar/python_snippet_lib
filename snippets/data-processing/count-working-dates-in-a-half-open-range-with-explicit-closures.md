---
title: "Count Working Dates in a Half-Open Range with Explicit Closures"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - shift-an-exact-date-by-whole-calendar-months-with-explicit-day-overflow.md
  - ../configuration-serialization/convert-a-weekday-bitmask-to-a-canonical-cron-schedule.md
---

# Count Working Dates in a Half-Open Range with Explicit Closures

## Idea and Problem

Count the dates on a recurring weekly schedule inside a reporting range, subtracting explicitly closed dates without visiting every date.

Every complete week contains each weekday once. Count full weeks in bulk,
inspect at most six remaining weekdays, then subtract distinct closures that
would otherwise be counted. Adjacent half-open date ranges can share an
endpoint without counting it twice.

## When to Use

Use this for report denominators, staffing calendars, or due-date policies
defined in whole local dates. The caller supplies working weekdays using
`date.weekday()` numbering (Monday is `0`) and all relevant closure dates.
Convert timestamps to business-local dates before calling the function.

## Implementation

```python
from collections.abc import Iterable
from datetime import date


def count_working_dates(
    start: date,
    end: date,
    working_weekdays: Iterable[int],
    closures: Iterable[date] = (),
) -> int:
    """Count scheduled dates in [start, end) after applying closures."""
    if type(start) is not date or type(end) is not date:
        raise TypeError("start and end must be dates without a time component")
    if end < start:
        raise ValueError("end must not precede start")

    weekdays: set[int] = set()
    for weekday in working_weekdays:
        if type(weekday) is not int or not 0 <= weekday <= 6:
            raise ValueError("working weekdays must be integers from 0 to 6")
        weekdays.add(weekday)

    whole_weeks, extra_days = divmod((end - start).days, 7)
    first_weekday = start.weekday()
    scheduled = whole_weeks * len(weekdays) + sum(
        (first_weekday + offset) % 7 in weekdays for offset in range(extra_days)
    )

    closed_working_dates: set[date] = set()
    for day in closures:
        if type(day) is not date:
            raise TypeError("closures must contain dates without a time component")
        if start <= day < end and day.weekday() in weekdays:
            closed_working_dates.add(day)
    return scheduled - len(closed_working_dates)
```

## Example

```python
start = date(2024, 2, 26)
boundary = date(2024, 3, 4)
weekdays = (0, 1, 2, 3, 4)
closures = (
    date(2024, 2, 29),
    date(2024, 2, 29),
    date(2024, 3, 2),
    boundary,
)

assert (
    count_working_dates(start, boundary, weekdays, closures),
    count_working_dates(boundary, date(2024, 3, 6), weekdays, closures),
    count_working_dates(boundary, boundary, weekdays, closures),
) == (4, 1, 0)
```

## Trade-offs and Limitations

Runtime depends on the number of supplied weekdays and closures, regardless of
the date span. Memory grows with the number of distinct closures that affect
the result. Inputs must be finite, and the weekly schedule must be constant
throughout the range.

This calendar models closures of normally working dates. It cannot represent
exceptional openings, partial days, or changes to the weekly schedule. Use a
maintained business calendar when holidays or observed dates need to be
computed from jurisdiction-specific rules.

## Related Snippets

<!-- catalog:related:start -->
- [Shift an Exact Date by Whole Calendar Months with Explicit Day Overflow](shift-an-exact-date-by-whole-calendar-months-with-explicit-day-overflow.md)
- [Convert a Weekday Bitmask to a Canonical Cron Schedule](../configuration-serialization/convert-a-weekday-bitmask-to-a-canonical-cron-schedule.md)
<!-- catalog:related:end -->
