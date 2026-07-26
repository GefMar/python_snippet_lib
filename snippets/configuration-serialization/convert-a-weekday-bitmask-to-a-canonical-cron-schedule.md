---
title: "Convert a Weekday Bitmask to a Canonical Cron Schedule"
snippet_type: recipe
use_cases:
  - configuration
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../reliability-resilience/assign-stable-schedule-slots-with-a-digest.md
  - parse-compact-durations-into-timedelta.md
---

# Convert a Weekday Bitmask to a Canonical Cron Schedule

## Idea and Problem

Translate a checkbox-style weekday bitmask and strict local wall time into one deliberately limited five-field cron representation.

The contract fixes bit 0 as Monday through bit 6 as Sunday. It emits numeric
Vixie-style day-of-week values with Sunday as `0`, sorts explicit values, and
uses `*` only when all seven days are selected, making supported schedules
round-trip without alternate spellings.

## When to Use

Use this adapter when an interface owns a weekday selection plus `HH:MM`, while
the configured scheduler accepts the exact five-field subset documented here.
Keep the original timezone beside the schedule and let the scheduler define
daylight-saving behavior. Use a full cron library when ranges, names, steps,
seconds, macros, or multiple cron dialects are required.

## Implementation

```python
import re


ALL_WEEKDAYS = (1 << 7) - 1
_TIME = re.compile(r"(?:[01][0-9]|2[0-3]):[0-5][0-9]")


def weekday_mask_to_cron(day_mask: int, wall_time: str) -> str:
    if isinstance(day_mask, bool) or not isinstance(day_mask, int):
        raise TypeError("day_mask must be an integer")
    if day_mask <= 0 or day_mask & ~ALL_WEEKDAYS:
        raise ValueError("day_mask must contain only Monday-to-Sunday bits")
    if not isinstance(wall_time, str):
        raise TypeError("wall_time must be text")
    if _TIME.fullmatch(wall_time) is None:
        raise ValueError("wall_time must use strict 24-hour HH:MM syntax")

    hour_text, minute_text = wall_time.split(":")
    if day_mask == ALL_WEEKDAYS:
        day_field = "*"
    else:
        cron_days = sorted(
            0 if weekday == 6 else weekday + 1
            for weekday in range(7)
            if day_mask & (1 << weekday)
        )
        day_field = ",".join(map(str, cron_days))
    return f"{int(minute_text)} {int(hour_text)} * * {day_field}"


def _canonical_field_integer(text: str, *, maximum: int, name: str) -> int:
    if not text.isascii() or not text.isdigit():
        raise ValueError(f"{name} must be an unsigned decimal integer")
    value = int(text)
    if text != str(value) or value > maximum:
        raise ValueError(f"{name} is outside its canonical range")
    return value


def cron_to_weekday_mask(expression: str) -> tuple[int, str]:
    if not isinstance(expression, str):
        raise TypeError("cron expression must be text")
    fields = expression.split(" ")
    if len(fields) != 5 or fields[2:4] != ["*", "*"]:
        raise ValueError("expected the supported canonical five-field cron subset")

    minute = _canonical_field_integer(fields[0], maximum=59, name="minute")
    hour = _canonical_field_integer(fields[1], maximum=23, name="hour")
    day_field = fields[4]

    if day_field == "*":
        day_mask = ALL_WEEKDAYS
    else:
        tokens = day_field.split(",")
        cron_days = [
            _canonical_field_integer(token, maximum=6, name="weekday")
            for token in tokens
        ]
        if not cron_days or cron_days != sorted(set(cron_days)):
            raise ValueError("weekday values must be unique and sorted")
        if len(cron_days) == 7:
            raise ValueError("all weekdays must use the wildcard")

        day_mask = 0
        for cron_day in cron_days:
            weekday = 6 if cron_day == 0 else cron_day - 1
            day_mask |= 1 << weekday

    wall_time = f"{hour:02d}:{minute:02d}"
    if weekday_mask_to_cron(day_mask, wall_time) != expression:
        raise ValueError("cron expression is not canonical")
    return day_mask, wall_time
```

## Example

```python
mixed_mask = (1 << 0) | (1 << 2) | (1 << 6)
schedules = (
    weekday_mask_to_cron(1 << 0, "09:30"),
    weekday_mask_to_cron(1 << 6, "00:00"),
    weekday_mask_to_cron(ALL_WEEKDAYS, "23:59"),
    weekday_mask_to_cron(mixed_mask, "09:30"),
)
round_trip = cron_to_weekday_mask(schedules[-1])

invalid_cron = (
    "30 09 * * 1",
    "30 9 * * 1,1",
    "30 9 * * 1,0",
    "30 9 * * 0,1,2,3,4,5,6",
    "30 9 * * 7",
    "30 9 * * 1-5",
)
rejected = []
for expression in invalid_cron:
    try:
        cron_to_weekday_mask(expression)
    except ValueError:
        rejected.append(expression)

assert (schedules, round_trip, tuple(rejected)) == (
    (
        "30 9 * * 1",
        "0 0 * * 0",
        "59 23 * * *",
        "30 9 * * 0,1,3",
    ),
    (mixed_mask, "09:30"),
    invalid_cron,
)
```

## Trade-offs and Limitations

Cron weekday numbering and accepted syntax vary across schedulers, so verify
that the target uses this exact Sunday-`0` five-field subset. The adapter does
not model seconds, ranges, steps, weekday names, macros, timezones, holidays,
or daylight-saving transitions. A bitmask also carries no locale or display
order; those remain user-interface concerns.

## Related Snippets

<!-- catalog:related:start -->
- [Assign Stable Schedule Slots with a Digest](../reliability-resilience/assign-stable-schedule-slots-with-a-digest.md)
- [Parse Compact Durations into timedelta](parse-compact-durations-into-timedelta.md)
<!-- catalog:related:end -->
