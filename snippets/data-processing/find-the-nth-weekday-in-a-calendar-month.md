---
title: "Find the Nth Weekday in a Calendar Month"
snippet_type: recipe
use_cases:
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - shift-an-exact-date-by-whole-calendar-months-with-explicit-day-overflow.md
  - split-a-half-open-date-range-into-clipped-iso-weeks.md
---

# Find the Nth Weekday in a Calendar Month

## Idea and Problem

Compute one occurrence of a weekday in a calendar month without accidentally continuing into the next month.

Start with the weekday offset from day one, add whole weeks for later occurrences, and check that the resulting day still belongs to the month.

## When to Use

Use this for Gregorian calendar rules such as the third Tuesday or fifth Thursday. The weekday uses zero for Monday and six for Sunday. Occurrences are numbered from one.

## Implementation

```python
from calendar import monthrange
from datetime import date


def nth_weekday(year: int, month: int, weekday: int, occurrence: int) -> date | None:
    first = date(year, month, 1)
    if not 0 <= weekday <= 6 or not 1 <= occurrence <= 5:
        raise ValueError("weekday or occurrence is out of range")

    offset = (weekday - first.weekday()) % 7
    day = 1 + offset + 7 * (occurrence - 1)
    _, last_day = monthrange(year, month)
    return first.replace(day=day) if day <= last_day else None
```

## Example

```python
assert (
    nth_weekday(2024, 2, 3, 5),
    nth_weekday(2025, 2, 3, 5),
    nth_weekday(2025, 2, 1, 3),
) == (date(2024, 2, 29), None, date(2025, 2, 18))
```

## Trade-offs and Limitations

This returns a date, not a timezone-aware execution time. It does not skip holidays, move missing occurrences forward, or implement a recurrence calendar. Years must fit the supported range of date.

## Related Snippets

<!-- catalog:related:start -->
- [Shift an Exact Date by Whole Calendar Months with Explicit Day Overflow](shift-an-exact-date-by-whole-calendar-months-with-explicit-day-overflow.md)
- [Split a Half-Open Date Range into Clipped ISO Weeks](split-a-half-open-date-range-into-clipped-iso-weeks.md)
<!-- catalog:related:end -->
