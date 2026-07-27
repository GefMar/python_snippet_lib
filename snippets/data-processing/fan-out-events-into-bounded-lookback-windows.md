---
title: "Fan Out Events into Bounded Lookback Windows"
snippet_type: algorithm
use_cases:
  - data-transformation
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - measure-time-in-a-state-within-a-half-open-window.md
  - yield-stream-items-with-bounded-neighbor-context.md
  - aggregate-consecutive-values-into-weighted-runs.md
---

# Fan Out Events into Bounded Lookback Windows

## Idea and Problem

Expand bounded timestamped events into explicit memberships for several lookback windows anchored at one timestamp.

An event belongs to a window of `w` seconds exactly when its timestamp is in
the half-open interval `[anchor - w, anchor)`. Events at the lower boundary are
included; events at the anchor or in the future are excluded. Memberships
preserve input event order and then the caller's declared window order.

## When to Use

Use this algorithm when a small in-memory batch must be labelled for multiple
fixed lookback calculations and downstream code benefits from explicit,
immutable memberships. It is useful for preparing inputs to independent
counts, sums, or other aggregation code without embedding those policies in
the fan-out step.

Use a sweep-line, prefix aggregate, or streaming window engine when either
dimension is large. Define watermark and lateness rules before using this idea
with events that arrive continuously or out of order.

## Implementation

```python
from collections.abc import Iterable
from dataclasses import dataclass
from itertools import islice


_MIN_TIMESTAMP = -(1 << 63)
_MAX_TIMESTAMP = (1 << 63) - 1
_MAX_EVENTS = 10_000
_MAX_WINDOWS = 64
_MAX_MEMBERSHIPS = 100_000
_MAX_EVENT_ID_BYTES = 128


def _signed_timestamp(value: object, *, name: str) -> int:
    if type(value) is not int:
        raise TypeError(f"{name} must be an exact integer")
    if not _MIN_TIMESTAMP <= value <= _MAX_TIMESTAMP:
        raise ValueError(f"{name} is outside the signed 64-bit range")
    return value


@dataclass(frozen=True, slots=True)
class LookbackEvent:
    event_id: str
    timestamp: int

    def __post_init__(self) -> None:
        if type(self.event_id) is not str:
            raise TypeError("event_id must be exact text")
        if (
            not self.event_id
            or self.event_id != self.event_id.strip()
            or not self.event_id.isprintable()
        ):
            raise ValueError("event_id must be stripped printable text")
        if len(self.event_id.encode("utf-8")) > _MAX_EVENT_ID_BYTES:
            raise ValueError("event_id exceeds the supported byte limit")
        _signed_timestamp(self.timestamp, name="timestamp")


@dataclass(frozen=True, slots=True)
class LookbackMembership:
    event_index: int
    event_id: str
    event_timestamp: int
    window_seconds: int


def fan_out_lookback_windows(
    events: Iterable[LookbackEvent],
    *,
    anchor: int,
    windows: tuple[int, ...],
    max_memberships: int = _MAX_MEMBERSHIPS,
) -> tuple[LookbackMembership, ...]:
    current_anchor = _signed_timestamp(anchor, name="anchor")
    if type(windows) is not tuple or not windows:
        raise TypeError("windows must be a non-empty tuple")
    if len(windows) > _MAX_WINDOWS:
        raise ValueError("window count exceeds the supported limit")
    for window in windows:
        if type(window) is not int:
            raise TypeError("window sizes must be exact integers")
        if not 1 <= window <= _MAX_TIMESTAMP:
            raise ValueError("window sizes must be positive and bounded")
    if len(set(windows)) != len(windows):
        raise ValueError("window sizes must be unique")

    if type(max_memberships) is not int:
        raise TypeError("max_memberships must be an exact integer")
    if not 0 <= max_memberships <= _MAX_MEMBERSHIPS:
        raise ValueError("max_memberships is outside the supported range")
    if isinstance(events, (str, bytes)):
        raise TypeError("events must be a non-text iterable")
    try:
        materialized = tuple(islice(events, _MAX_EVENTS + 1))
    except TypeError as error:
        raise TypeError("events must be iterable") from error
    if len(materialized) > _MAX_EVENTS:
        raise ValueError("event count exceeds the supported limit")
    if any(type(event) is not LookbackEvent for event in materialized):
        raise TypeError("every event must be a LookbackEvent")

    worst_case = len(materialized) * len(windows)
    if worst_case > max_memberships:
        raise ValueError("worst-case fan-out exceeds max_memberships")

    memberships = tuple(
        LookbackMembership(
            event_index=index,
            event_id=event.event_id,
            event_timestamp=event.timestamp,
            window_seconds=window,
        )
        for index, event in enumerate(materialized)
        for window in windows
        if current_anchor - window <= event.timestamp < current_anchor
    )
    if len(memberships) > max_memberships:
        raise ValueError("actual fan-out exceeds max_memberships")
    return memberships
```

## Example

```python
events = (
    LookbackEvent("old-boundary", 100),
    LookbackEvent("recent-boundary", 700),
    LookbackEvent("at-anchor", 1_000),
    LookbackEvent("future", 1_001),
)

memberships = fan_out_lookback_windows(
    events,
    anchor=1_000,
    windows=(900, 300),
    max_memberships=8,
)

assert tuple(
    (item.event_index, item.event_id, item.window_seconds)
    for item in memberships
) == (
    (0, "old-boundary", 900),
    (1, "recent-boundary", 900),
    (1, "recent-boundary", 300),
)

try:
    fan_out_lookback_windows(
        events[:2],
        anchor=1_000,
        windows=(900, 300),
        max_memberships=3,
    )
except ValueError:
    worst_case_rejected = True
else:
    worst_case_rejected = False

assert worst_case_rejected
```

## Trade-offs and Limitations

The algorithm performs `O(events * windows)` membership tests and stores the
bounded event snapshot plus the immutable output. It rejects the full
multiplicative worst case before constructing memberships, even when the
actual timestamps would produce a smaller result. This conservative rule keeps
the memory contract independent of input values.

Declared window order is preserved rather than sorted. Duplicate events are
allowed and remain distinguishable by `event_index`, but window sizes must be
unique. The helper does not aggregate memberships, parse timestamps, assign
an event-time watermark, handle late arrivals, persist results, or deduplicate
event identifiers.

## Related Snippets

<!-- catalog:related:start -->
- [Measure Time in a State Within a Half-Open Window](measure-time-in-a-state-within-a-half-open-window.md)
- [Yield Stream Items with Bounded Neighbor Context](yield-stream-items-with-bounded-neighbor-context.md)
- [Aggregate Consecutive Values into Weighted Runs](aggregate-consecutive-values-into-weighted-runs.md)
<!-- catalog:related:end -->
