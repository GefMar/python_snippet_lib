---
title: "Split a Half-Open Date Range into Clipped ISO Weeks"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - count-working-dates-in-a-half-open-range-with-explicit-closures.md
  - ../storage-databases/select-snapshot-representatives-by-utc-calendar-buckets.md
---

# Split a Half-Open Date Range into Clipped ISO Weeks

## Idea and Problem

Turn a date range into adjacent ISO-week slices with exact clipped edges for weekly reporting or batch queries.

ISO weeks begin on Monday. The ISO year can differ from the calendar year
near New Year, so label each slice with both components from `date.isocalendar()`.
Jump directly to the next Monday, clipping the jump to the remaining days
before doing date arithmetic. This keeps the final slice within the requested
range even near `date.max`.

## When to Use

Use this when a weekly export, report or query owns a range of calendar dates.
Give the function an inclusive start and an exclusive end. Each result is
nonempty and owns only dates inside that range. Convert timestamped records
to dates in the reporting timezone before applying these slices.

## Implementation

```python
from collections.abc import Iterator
from dataclasses import dataclass
from datetime import date, timedelta


@dataclass(frozen=True, slots=True)
class WeekSlice:
    iso_year: int
    iso_week: int
    start: date
    end: date


def split_iso_weeks(start: date, end: date) -> Iterator[WeekSlice]:
    """Yield the ISO-week partition of the date range [start, end)."""
    if type(start) is not date or type(end) is not date:
        raise TypeError("start and end must be dates without a time component")
    if end < start:
        raise ValueError("end must not precede start")

    cursor = start
    while cursor < end:
        week = cursor.isocalendar()
        days_to_monday = 7 - cursor.weekday()
        stop = cursor + timedelta(days=min(days_to_monday, (end - cursor).days))
        yield WeekSlice(week.year, week.week, cursor, stop)
        cursor = stop
```

## Example

```python
new_year = tuple(split_iso_weeks(date(2020, 12, 31), date(2021, 1, 6)))
last_day = date.max - timedelta(days=1)

assert (
    new_year,
    tuple(split_iso_weeks(date(2021, 1, 4), date(2021, 1, 4))),
    tuple(split_iso_weeks(last_day, date.max)),
) == (
    (
        WeekSlice(2020, 53, date(2020, 12, 31), date(2021, 1, 4)),
        WeekSlice(2021, 1, date(2021, 1, 4), date(2021, 1, 6)),
    ),
    (),
    (WeekSlice(9999, 52, last_day, date.max),),
)
```

## Trade-offs and Limitations

The iterator does work proportional to the number of slices and retains one
slice at a time. Store the results only if needed. Type and order errors surface
when iteration starts. The exclusive end must be representable as a `date`,
so `date.max` can be an endpoint but cannot be included in a slice.

This policy uses ISO 8601 weeks. For business calendars with Sunday starts,
fiscal weeks or custom cutovers, supply that calendar's boundaries and labels.

## Related Snippets

<!-- catalog:related:start -->
- [Count Working Dates in a Half-Open Range with Explicit Closures](count-working-dates-in-a-half-open-range-with-explicit-closures.md)
- [Select Snapshot Representatives by UTC Calendar Buckets](../storage-databases/select-snapshot-representatives-by-utc-calendar-buckets.md)
<!-- catalog:related:end -->
