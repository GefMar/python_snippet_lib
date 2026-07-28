---
title: "Partition Strictly Increasing Integer Timestamps into Idle-Gap Sessions"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - join-bounded-strictly-increasing-sequences-by-the-latest-prior-timestamp.md
  - measure-time-in-a-state-within-a-half-open-window.md
  - ../algorithms-data-structures/accept-sparse-observations-that-preserve-strict-position-time-order.md
---

# Partition Strictly Increasing Integer Timestamps into Idle-Gap Sessions

## Idea and Problem

Partition a bounded strictly increasing timestamp tuple whenever one adjacent idle gap exceeds an explicit threshold.

The result contains immutable half-open index ranges into the original tuple.
A gap exactly equal to the threshold stays in the current session; only a
strictly greater gap starts the next range.

## When to Use

Use this algorithm for a complete in-memory event snapshot whose timestamps
already share one caller-defined integer unit and timeline. Half-open index
ranges avoid copying event payloads and let the caller slice aligned records
with the same session boundaries.

Normalize timestamps and decide the idle threshold before calling. Use a
stateful sessionizer when events arrive incrementally, and define lateness and
watermark behavior separately when observations can arrive out of order.

## Implementation

```python
from dataclasses import dataclass

_MIN_TIMESTAMP = -(1 << 63)
_MAX_TIMESTAMP = (1 << 63) - 1
_MAX_TIMESTAMPS = 10_000
_MAX_IDLE_GAP = (1 << 63) - 1


@dataclass(frozen=True, slots=True)
class SessionRange:
    start: int
    stop: int


def partition_idle_gap_sessions(
    timestamps: tuple[int, ...],
    *,
    max_idle_gap: int,
) -> tuple[SessionRange, ...]:
    """Return half-open index ranges separated by gaps above the threshold."""
    if type(timestamps) is not tuple:
        raise TypeError("timestamps must be an exact tuple")
    if len(timestamps) > _MAX_TIMESTAMPS:
        raise ValueError(f"timestamps must contain at most {_MAX_TIMESTAMPS} items")
    if type(max_idle_gap) is not int:
        raise TypeError("max_idle_gap must be an exact integer")
    if not 0 <= max_idle_gap <= _MAX_IDLE_GAP:
        raise ValueError("max_idle_gap is outside the supported range")

    ranges: list[SessionRange] = []
    session_start = 0
    previous: int | None = None

    for index, timestamp in enumerate(timestamps):
        if type(timestamp) is not int:
            raise TypeError("timestamps must contain exact integers")
        if not _MIN_TIMESTAMP <= timestamp <= _MAX_TIMESTAMP:
            raise ValueError("timestamps must be in the signed 64-bit range")
        if previous is not None:
            if timestamp <= previous:
                raise ValueError("timestamps must strictly increase")
            if timestamp - previous > max_idle_gap:
                ranges.append(SessionRange(session_start, index))
                session_start = index
        previous = timestamp

    if timestamps:
        ranges.append(SessionRange(session_start, len(timestamps)))
    return tuple(ranges)
```

## Example

```python
timestamps = (10, 12, 15, 21, 28, 29)
ranges = partition_idle_gap_sessions(timestamps, max_idle_gap=6)
sessions = tuple(timestamps[item.start : item.stop] for item in ranges)

try:
    partition_idle_gap_sessions((1, 1), max_idle_gap=0)
except ValueError:
    duplicate_rejected = True
else:
    duplicate_rejected = False

try:
    partition_idle_gap_sessions((1, True), max_idle_gap=1)
except TypeError:
    bool_timestamp_rejected = True
else:
    bool_timestamp_rejected = False

assert (
    ranges,
    sessions,
    partition_idle_gap_sessions((), max_idle_gap=0),
    duplicate_rejected,
    bool_timestamp_rejected,
) == (
    (SessionRange(0, 4), SessionRange(4, 6)),
    ((10, 12, 15, 21), (28, 29)),
    (),
    True,
    True,
)
```

## Trade-offs and Limitations

Validation and partitioning take `O(n)` time. The result uses `O(s)` memory
for `s` sessions and retains no reference to the timestamp tuple. Consumers
must keep the original tuple, or an aligned record tuple, if they need to use
the returned index ranges.

Timestamp units and epoch semantics belong entirely to the caller. The
function accepts no duplicate or out-of-order timestamps and performs no
parsing, unit conversion, clock normalization, duration inference, grouping by
another key, lateness handling, or incremental merging. Python integer
subtraction cannot overflow, but admitted timestamps and the threshold remain
explicitly bounded.

## Related Snippets

<!-- catalog:related:start -->
- [Join Bounded Strictly Increasing Sequences by the Latest Prior Timestamp](join-bounded-strictly-increasing-sequences-by-the-latest-prior-timestamp.md)
- [Measure Time in a State Within a Half-Open Window](measure-time-in-a-state-within-a-half-open-window.md)
- [Accept Sparse Observations That Preserve Strict Position-Time Order](../algorithms-data-structures/accept-sparse-observations-that-preserve-strict-position-time-order.md)
<!-- catalog:related:end -->
