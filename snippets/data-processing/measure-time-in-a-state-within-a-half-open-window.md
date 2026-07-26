---
title: "Measure Time in a State Within a Half-Open Window"
snippet_type: algorithm
use_cases:
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - aggregate-consecutive-values-into-weighted-runs.md
  - ../algorithms-data-structures/find-a-point-in-disjoint-half-open-intervals.md
---

# Measure Time in a State Within a Half-Open Window

## Idea and Problem

Measure how long an append-only state history occupies one target state inside an exact half-open reporting window.

Each normalized event changes the current state from its timestamp onward. An
explicit initial state removes guesswork before the first event, while clipping
transitions to `[window_start, window_end)` avoids double-counting adjacent
reporting windows.

## When to Use

Use this algorithm for finite, complete, non-decreasing state histories behind
SLA, utilization, or lifecycle reports. Normalize domain operations into states
before calling it, and choose an initial state that is valid immediately before
the history. Do not use it to infer missing transitions or reconstruct a state
from an incomplete event feed.

## Implementation

```python
from collections.abc import Iterable
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone


@dataclass(frozen=True, slots=True)
class StateChange:
    at: datetime
    state: str


@dataclass(frozen=True, slots=True)
class StateWindowMeasure:
    duration: timedelta
    state_at_end: str


def _is_aware(value: datetime) -> bool:
    return value.utcoffset() is not None


def _timeline_value(value: datetime) -> datetime:
    return value.astimezone(timezone.utc) if _is_aware(value) else value


def measure_state_time(
    events: Iterable[StateChange],
    *,
    window_start: datetime,
    window_end: datetime,
    initial_state: str,
    target_state: str,
) -> StateWindowMeasure:
    if not isinstance(window_start, datetime) or not isinstance(window_end, datetime):
        raise TypeError("window boundaries must be datetime values")
    if _is_aware(window_start) != _is_aware(window_end):
        raise TypeError("window boundaries must have compatible awareness")
    timeline_start = _timeline_value(window_start)
    timeline_end = _timeline_value(window_end)
    if timeline_start >= timeline_end:
        raise ValueError("window_start must be earlier than window_end")
    for name, state in (("initial_state", initial_state), ("target_state", target_state)):
        if not isinstance(state, str) or not state:
            raise ValueError(f"{name} must be non-empty text")

    window_is_aware = _is_aware(window_start)
    current_state = initial_state
    cursor = timeline_start
    duration = timedelta()
    previous_at: datetime | None = None

    for event in events:
        if not isinstance(event, StateChange):
            raise TypeError("events must contain StateChange values")
        if not isinstance(event.at, datetime):
            raise TypeError("event timestamps must be datetime values")
        if not isinstance(event.state, str) or not event.state:
            raise ValueError("event states must be non-empty text")
        if _is_aware(event.at) != window_is_aware:
            raise TypeError("event timestamps must match window awareness")
        event_at = _timeline_value(event.at)
        if previous_at is not None and event_at < previous_at:
            raise ValueError("events must be ordered by non-decreasing timestamp")
        previous_at = event_at

        if event_at < timeline_start:
            current_state = event.state
            continue
        if event_at > timeline_end:
            continue

        if current_state == target_state and cursor < event_at:
            duration += event_at - cursor
        cursor = event_at
        current_state = event.state

    if cursor < timeline_end and current_state == target_state:
        duration += timeline_end - cursor

    return StateWindowMeasure(duration=duration, state_at_end=current_state)
```

## Example

```python
from datetime import tzinfo


base = datetime(2026, 1, 1, tzinfo=timezone.utc)
events = [
    StateChange(base + timedelta(hours=8), "active"),
    StateChange(base + timedelta(hours=10), "idle"),
    StateChange(base + timedelta(hours=11), "active"),
    StateChange(base + timedelta(hours=13), "idle"),
    StateChange(base + timedelta(hours=13), "closed"),
    StateChange(base + timedelta(hours=14), "active"),
]
measured = measure_state_time(
    events,
    window_start=base + timedelta(hours=9),
    window_end=base + timedelta(hours=13),
    initial_state="idle",
    target_state="active",
)

boundary = measure_state_time(
    [
        StateChange(base, "active"),
        StateChange(base + timedelta(hours=1), "idle"),
    ],
    window_start=base,
    window_end=base + timedelta(hours=1),
    initial_state="idle",
    target_state="active",
)


class SpringForward(tzinfo):
    def utcoffset(self, value: datetime | None) -> timedelta:
        if value is None:
            return timedelta(hours=1)
        return timedelta(hours=1 if value.hour < 3 else 2)

    def dst(self, value: datetime | None) -> timedelta:
        return self.utcoffset(value) - timedelta(hours=1)


civil_zone = SpringForward()
spring_elapsed = measure_state_time(
    [],
    window_start=datetime(2026, 3, 29, 1, 30, tzinfo=civil_zone),
    window_end=datetime(2026, 3, 29, 3, 30, tzinfo=civil_zone),
    initial_state="active",
    target_state="active",
)

try:
    measure_state_time(
        [StateChange(base + timedelta(minutes=2), "a"), StateChange(base, "b")],
        window_start=base,
        window_end=base + timedelta(hours=1),
        initial_state="idle",
        target_state="active",
    )
except ValueError:
    unordered_rejected = True
else:
    unordered_rejected = False

assert (
    measured.duration,
    measured.state_at_end,
    boundary.duration,
    boundary.state_at_end,
    spring_elapsed.duration,
    unordered_rejected,
) == (
    timedelta(hours=3),
    "closed",
    timedelta(hours=1),
    "idle",
    timedelta(hours=1),
    True,
)
```

## Trade-offs and Limitations

The function consumes the complete finite iterable to validate ordering, even
after the reporting endpoint. Events sharing a timestamp are applied in input
order, and events exactly at `window_end` affect `state_at_end` but add no time
to the half-open window. Naive and aware datetimes cannot be mixed. Aware values
are normalized to UTC before ordering and subtraction so daylight-saving
offset changes use elapsed time; naive values remain on one caller-defined
timeline. The result is not calendar-day arithmetic, and no missing or
contradictory events are repaired.

## Related Snippets

<!-- catalog:related:start -->
- [Aggregate Consecutive Values into Weighted Runs](aggregate-consecutive-values-into-weighted-runs.md)
- [Find a Point in Disjoint Half-Open Intervals](../algorithms-data-structures/find-a-point-in-disjoint-half-open-intervals.md)
<!-- catalog:related:end -->
