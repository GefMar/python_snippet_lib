---
title: "Parse a Closed RFC 3339 Timestamp Subset into an Aware Datetime"
snippet_type: recipe
use_cases:
  - configuration
  - interoperability
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - parse-compact-durations-into-timedelta.md
  - render-fixed-date-placeholders-from-an-explicit-anchor.md
  - ../storage-databases/select-expired-backup-names-with-strict-utc-timestamps.md
---

# Parse a Closed RFC 3339 Timestamp Subset into an Aware Datetime

## Idea and Problem

Parse one bounded timestamp with an explicit UTC designator or numeric offset into an aware datetime under a deliberately closed RFC 3339 subset.

The grammar is `YYYY-MM-DDTHH:MM:SS`, followed by an optional one-to-six-digit
fraction and either `Z` or a `+HH:MM` or `-HH:MM` offset. Explicit range checks
reject year zero, leap-second spellings, end-of-day `24:00`, out-of-range
offsets, and the `-00:00` convention for an unknown local offset.

## When to Use

Use this recipe at a configuration or protocol boundary whose producers can
emit this exact ASCII profile and whose consumers need an aware fixed-offset
value. The parser requires uppercase `T` and `Z`, seconds, and an explicit
offset; it accepts no surrounding whitespace or alternate separators.

Use a standards-complete parser when the entire RFC 3339 grammar, including
leap seconds or unknown-offset semantics, is required. Use an explicit time
zone database and policy when a region name, daylight-saving history, or local
wall-time disambiguation matters.

## Implementation

```python
import re
from datetime import UTC, datetime, timedelta, timezone

_MAX_TIMESTAMP_LENGTH = 32
_RFC3339_SUBSET = re.compile(
    r"(?P<year>[0-9]{4})-(?P<month>[0-9]{2})-(?P<day>[0-9]{2})"
    r"T(?P<hour>[0-9]{2}):(?P<minute>[0-9]{2}):(?P<second>[0-9]{2})"
    r"(?:\.(?P<fraction>[0-9]{1,6}))?"
    r"(?P<zone>Z|(?P<sign>[+-])(?P<offset_hour>[0-9]{2}):"
    r"(?P<offset_minute>[0-9]{2}))",
    re.ASCII,
)


def parse_rfc3339_timestamp(text: str) -> datetime:
    if type(text) is not str:
        raise TypeError("timestamp must be an exact string")
    if not 20 <= len(text) <= _MAX_TIMESTAMP_LENGTH:
        raise ValueError("timestamp length is outside the supported range")
    if not text.isascii():
        raise ValueError("timestamp must contain only ASCII")

    match = _RFC3339_SUBSET.fullmatch(text)
    if match is None:
        raise ValueError("timestamp does not match the supported RFC 3339 grammar")

    year = int(match.group("year"))
    month = int(match.group("month"))
    day = int(match.group("day"))
    hour = int(match.group("hour"))
    minute = int(match.group("minute"))
    second = int(match.group("second"))
    if year == 0:
        raise ValueError("year zero is not supported")
    if hour > 23 or minute > 59 or second > 59:
        raise ValueError("timestamp contains an out-of-range clock value")

    fraction = match.group("fraction")
    microsecond = int(fraction.ljust(6, "0")) if fraction is not None else 0

    if match.group("zone") == "Z":
        zone = UTC
    else:
        offset_hour = int(match.group("offset_hour"))
        offset_minute = int(match.group("offset_minute"))
        if offset_hour > 23 or offset_minute > 59:
            raise ValueError("timestamp contains an out-of-range UTC offset")
        sign = match.group("sign")
        if sign == "-" and offset_hour == 0 and offset_minute == 0:
            raise ValueError("the unknown-offset convention -00:00 is not supported")
        offset = timedelta(hours=offset_hour, minutes=offset_minute)
        if sign == "-":
            offset = -offset
        zone = timezone(offset)

    try:
        return datetime(
            year,
            month,
            day,
            hour,
            minute,
            second,
            microsecond,
            tzinfo=zone,
        )
    except ValueError:
        raise ValueError("timestamp contains an invalid calendar date") from None
```

## Example

```python
parsed = (
    parse_rfc3339_timestamp("1985-04-12T23:20:50.52Z"),
    parse_rfc3339_timestamp("1996-12-19T16:39:57-08:00"),
    parse_rfc3339_timestamp("2024-02-29T00:00:00+23:59"),
)

invalid_values = (
    "0000-01-01T00:00:00Z",
    "2023-02-29T00:00:00Z",
    "2024-01-01T23:59:60Z",
    "2024-01-01T24:00:00Z",
    "2024-01-01T00:00:00+24:00",
    "2024-01-01T00:00:00+23:60",
    "2024-01-01T00:00:00-00:00",
    "2024-01-01T00:00:00.1234567Z",
    "2024-01-01t00:00:00z",
    "2024-01-01T00:00:00",
)
rejected = []
for value in invalid_values:
    try:
        parse_rfc3339_timestamp(value)
    except ValueError:
        rejected.append(value)

assert (
    parsed[0],
    parsed[1].utcoffset(),
    parsed[2].utcoffset(),
    tuple(rejected),
) == (
    datetime(1985, 4, 12, 23, 20, 50, 520_000, tzinfo=UTC),
    timedelta(hours=-8),
    timedelta(hours=23, minutes=59),
    invalid_values,
)
```

## Trade-offs and Limitations

The parser implements only the documented closed subset. It excludes basic
date forms, lowercase designators, whitespace, missing seconds, fractions
longer than microseconds, leap seconds, and `-00:00`. It creates only UTC or a
fixed-offset `timezone`; it neither loads a named time zone nor infers daylight
saving behavior. Converting the result back to text can lose the original
choice of `Z` versus `+00:00` and the original number of fractional digits.
Syntax and calendar validation do not establish that a timestamp is timely,
trusted, unique, or authorized for an application action.

## Related Snippets

<!-- catalog:related:start -->
- [Parse Compact Durations into timedelta](parse-compact-durations-into-timedelta.md)
- [Render Fixed Date Placeholders from an Explicit Anchor](render-fixed-date-placeholders-from-an-explicit-anchor.md)
- [Select Expired Backup Names with Strict UTC Timestamps](../storage-databases/select-expired-backup-names-with-strict-utc-timestamps.md)
<!-- catalog:related:end -->
