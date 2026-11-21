---
title: "Find the Earliest Common Slot in Multiple Availability Calendars"
snippet_type: algorithm
use_cases:
  - automation
  - data-transformation
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - coalesce-bounded-half-open-integer-intervals-under-an-explicit-adjacency-policy.md
  - ../data-processing/join-non-overlapping-half-open-validity-intervals-by-exact-overlap.md
---

# Find the Earliest Common Slot in Multiple Availability Calendars

## Idea and Problem

Find the first slot of a requested duration that fits every availability calendar.

Keep one current window from each calendar. Their first possible common slot
starts at the latest of their start times and must finish by the earliest end
time. If the slot is too long, the earliest-ending window can be discarded.
Later windows in the other calendars can only move the candidate start
forward. A min-heap identifies which calendar to advance.

## When to Use

Use this to find a meeting or maintenance window across several independent
calendars on one integer timeline. Supply at least one calendar, with its
half-open windows sorted and separated by positive gaps. Coalesce touching or
overlapping availability first. Normalize time zones and calendar rules
before calling the function.

## Implementation

```python
from collections.abc import Iterable, Iterator
from heapq import heapify, heapreplace


def _checked_windows(windows: Iterable[tuple[int, int]]) -> Iterator[tuple[int, int]]:
    previous_end: int | None = None
    for start, end in windows:
        if type(start) is not int or type(end) is not int:
            raise TypeError("window boundaries must be integers")
        if start >= end or (previous_end is not None and start <= previous_end):
            raise ValueError("windows must be nonempty, ordered and separated by gaps")
        previous_end = end
        yield start, end


def earliest_common_slot(
    calendars: Iterable[Iterable[tuple[int, int]]], duration: int
) -> tuple[int, int] | None:
    if type(duration) is not int or duration <= 0:
        raise ValueError("duration must be a positive integer")

    readers: list[Iterator[tuple[int, int]]] = []
    ends: list[tuple[int, int]] = []
    latest_start: int | None = None
    for index, calendar in enumerate(calendars):
        reader = _checked_windows(calendar)
        first = next(reader, None)
        if first is None:
            return None
        start, end = first
        latest_start = start if latest_start is None else max(latest_start, start)
        readers.append(reader)
        ends.append((end, index))
    if latest_start is None:
        raise ValueError("at least one calendar is required")

    heapify(ends)
    while True:
        earliest_end, calendar_index = ends[0]
        slot_end = latest_start + duration
        if slot_end <= earliest_end:
            return latest_start, slot_end

        window = next(readers[calendar_index], None)
        if window is None:
            return None
        start, end = window
        latest_start = max(latest_start, start)
        heapreplace(ends, (end, calendar_index))
```

## Example

```python
calendars = (
    ((0, 5), (7, 15)),
    ((1, 4), (5, 13)),
    ((2, 3), (8, 12)),
)

assert (
    earliest_common_slot(calendars, duration=3),
    earliest_common_slot(calendars, duration=4),
    earliest_common_slot(calendars, duration=5),
    earliest_common_slot((calendars[0], ()), duration=3),
) == ((8, 11), (8, 12), None, None)
```

## Trade-offs and Limitations

The heap holds one end time per calendar and each advance costs logarithmic
time in the number of calendars. The search reads only the needed prefixes.
Checks apply to the windows it consumes, so validate the full inputs first
when every window must be checked.

The duration is measured on the caller's shared integer timeline. Discrete
start restrictions, recurring availability and required breaks between
meetings need additional scheduling policy.

## Related Snippets

<!-- catalog:related:start -->
- [Coalesce Bounded Half-Open Integer Intervals Under an Explicit Adjacency Policy](coalesce-bounded-half-open-integer-intervals-under-an-explicit-adjacency-policy.md)
- [Join Non-Overlapping Half-Open Validity Intervals by Exact Overlap](../data-processing/join-non-overlapping-half-open-validity-intervals-by-exact-overlap.md)
<!-- catalog:related:end -->
