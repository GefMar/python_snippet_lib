---
title: "Convert Aware Datetimes to Exact Unix Microseconds"
snippet_type: recipe
use_cases:
  - data-transformation
  - serialization
tested_python:
  - "3.14"
dependencies: []
related:
  - ../configuration-serialization/parse-a-closed-rfc-3339-timestamp-subset-into-an-aware-datetime.md
  - ../configuration-serialization/classify-a-naive-local-datetime-against-an-explicit-zoneinfo-transition.md
---

# Convert Aware Datetimes to Exact Unix Microseconds

## Idea and Problem

Store a timezone-aware date and time as an exact signed count of microseconds since the Unix epoch.

A floating-point timestamp can lose microseconds for dates far from 1970.
Convert to UTC, subtract the epoch, and divide the resulting `timedelta`
by one microsecond with integer floor division. A `timedelta` already has
microsecond resolution, so every such division is exact, even before the
epoch.

## When to Use

Use this when storing or comparing Python datetimes in a system that expects
signed Unix microseconds. Supply an aware standard-library `datetime` that
has a representable UTC value. Resolve any ambiguity in the source local
time before conversion.

## Implementation

```python
from datetime import UTC, datetime, timedelta

_UNIX_EPOCH = datetime(1970, 1, 1, tzinfo=UTC)
_MICROSECOND = timedelta(microseconds=1)


def to_unix_microseconds(value: datetime) -> int:
    if type(value) is not datetime:
        raise TypeError("value must be a standard datetime")
    if value.utcoffset() is None:
        raise ValueError("value must be timezone-aware")

    return (value.astimezone(UTC) - _UNIX_EPOCH) // _MICROSECOND


def from_unix_microseconds(count: int) -> datetime:
    if type(count) is not int:
        raise TypeError("count must be an integer number of microseconds")
    return _UNIX_EPOCH + timedelta(microseconds=count)
```

## Example

```python
from datetime import timezone

early = datetime(1969, 12, 31, 23, 59, 59, 999999, tzinfo=UTC)
local = datetime(1970, 1, 1, 2, 30, 0, 7, tzinfo=timezone(timedelta(hours=2, minutes=30)))
later = datetime(2600, 7, 2, 4, 8, 10, 123456, tzinfo=UTC)

assert (
    to_unix_microseconds(early),
    to_unix_microseconds(local),
    from_unix_microseconds(-1),
    from_unix_microseconds(to_unix_microseconds(later)),
) == (-1, 7, early, later)
```

## Trade-offs and Limitations

The inverse produces a UTC datetime. Store the original zone or offset
separately when it matters for display or later local-time operations.
An offset conversion or decoded count outside the supported datetime range
raises `OverflowError`.

This representation inherits Python datetime's microsecond resolution and
86,400-second days. Choose a different time model when the input needs leap
seconds or finer precision.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Closed RFC 3339 Timestamp Subset into an Aware Datetime](../configuration-serialization/parse-a-closed-rfc-3339-timestamp-subset-into-an-aware-datetime.md)
- [Classify a Naive Local Datetime Against an Explicit ZoneInfo Transition](../configuration-serialization/classify-a-naive-local-datetime-against-an-explicit-zoneinfo-transition.md)
<!-- catalog:related:end -->
