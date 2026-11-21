---
title: "Parse and Render a Canonical ISO Week Date"
snippet_type: recipe
use_cases:
  - parsing
  - serialization
tested_python:
  - "3.14"
dependencies: []
related:
  - ../data-processing/split-a-half-open-date-range-into-clipped-iso-weeks.md
  - ../data-processing/convert-aware-datetimes-to-exact-unix-microseconds.md
---

# Parse and Render a Canonical ISO Week Date

## Idea and Problem

Parse and render an ISO week date without confusing its week-year with its Gregorian year.

Check the exact lexical form first, then use the date type to validate the week and weekday. When rendering, read the week-year, week, and weekday from the same date.

## When to Use

Use this for interchange fields that require the single extended ISO week-date spelling with a four-digit year. It is not a general parser for ISO date variants.

## Implementation

```python
import re
from datetime import date


WEEK_DATE = re.compile(r"([0-9]{4})-W([0-9]{2})-([1-7])")


def parse_week_date(value: str) -> date:
    match = WEEK_DATE.fullmatch(value)
    if match is None:
        raise ValueError("expected an extended ISO week date")
    year, week, weekday = map(int, match.groups())
    return date.fromisocalendar(year, week, weekday)


def render_week_date(value: date) -> str:
    year, week, weekday = value.isocalendar()
    return f"{year:04d}-W{week:02d}-{weekday}"
```

## Example

```python
rejected = []
for value in ("2021-W53-5", "2020-W53-8", "2020W535"):
    try:
        parse_week_date(value)
    except ValueError:
        rejected.append(value)

assert (
    parse_week_date("2020-W53-5"),
    render_week_date(date(2021, 1, 1)),
    render_week_date(date(2018, 12, 31)),
    rejected,
) == (date(2021, 1, 1), "2020-W53-5", "2019-W01-1", ["2021-W53-5", "2020-W53-8", "2020W535"])
```

## Trade-offs and Limitations

The supported dates are limited by the Python date type. The result is a day rather than a local time or time zone. If several input spellings are acceptable, the broader standard fromisoformat parser may be simpler.

## Related Snippets

<!-- catalog:related:start -->
- [Split a Half-Open Date Range into Clipped ISO Weeks](../data-processing/split-a-half-open-date-range-into-clipped-iso-weeks.md)
- [Convert Aware Datetimes to Exact Unix Microseconds](../data-processing/convert-aware-datetimes-to-exact-unix-microseconds.md)
<!-- catalog:related:end -->
