---
title: "Measure Elapsed Time Across a Repeated Local Hour"
snippet_type: recipe
use_cases:
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../configuration-serialization/classify-a-naive-local-datetime-against-an-explicit-zoneinfo-transition.md
  - convert-aware-datetimes-to-exact-unix-microseconds.md
---

# Measure Elapsed Time Across a Repeated Local Hour

## Idea and Problem

Measure elapsed time between aware timestamps even when both display the same local clock reading.

Convert both instants to UTC before subtracting. Direct subtraction of datetimes sharing one time-zone object uses their local fields, so two occurrences of the same wall time can produce an unexpectedly zero difference.

## When to Use

Use this for past observations that already identify the instant, including an explicit fold value during a repeated local hour. For an in-process stopwatch, prefer a monotonic clock.

## Implementation

```python
from datetime import UTC, datetime, timedelta


def elapsed_between(start: datetime, finish: datetime) -> timedelta:
    if start.utcoffset() is None or finish.utcoffset() is None:
        raise ValueError("both timestamps must be aware")
    return finish.astimezone(UTC) - start.astimezone(UTC)
```

## Example

```python
from zoneinfo import ZoneInfo

zone = ZoneInfo("America/New_York")
first = datetime(2024, 11, 3, 1, 30, tzinfo=zone, fold=0)
second = first.replace(fold=1)
assert (
    second - first,
    elapsed_between(first, second),
    elapsed_between(second, first),
) == (timedelta(0), timedelta(hours=1), -timedelta(hours=1))
```

## Trade-offs and Limitations

The example needs an installed time-zone database. This assumes the caller has already resolved ambiguous or nonexistent local times; it does not validate a time zone attached to a naive clock reading. Clock adjustments can still affect timestamps recorded from the wall clock.

## Related Snippets

<!-- catalog:related:start -->
- [Classify a Naive Local Datetime Against an Explicit ZoneInfo Transition](../configuration-serialization/classify-a-naive-local-datetime-against-an-explicit-zoneinfo-transition.md)
- [Convert Aware Datetimes to Exact Unix Microseconds](convert-aware-datetimes-to-exact-unix-microseconds.md)
<!-- catalog:related:end -->
