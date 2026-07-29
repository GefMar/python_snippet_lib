---
title: "Reorder Bounded Out-of-Order Timestamped Records Under a Declared Lateness Bound"
snippet_type: pattern
use_cases:
  - data-transformation
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - fan-out-events-into-bounded-lookback-windows.md
  - partition-strictly-increasing-integer-timestamps-into-idle-gap-sessions.md
  - join-two-strictly-increasing-streams-by-exact-timestamp.md
---

# Reorder Bounded Out-of-Order Timestamped Records Under a Declared Lateness Bound

## Idea and Problem

Reorder a finite sequence by timestamp while separating arrivals that violate an explicit maximum-disorder assumption.

After every arrival, subtract `allowed_lateness` from the greatest timestamp
seen so far. This derived frontier is an open release boundary: buffered
records strictly below it can be emitted, while records equal to it remain
pending. A newly arriving record strictly below the frontier is late and is
never inserted into the buffer.

The heap key `(timestamp, arrival_index)` makes every release stable for equal
timestamps. A final drain closes the finite input and emits all remaining
admitted records in the same order.

## When to Use

Use this pattern when a complete bounded input represents one arrival stream
and its producer contract guarantees that any on-time timestamp trails the
greatest observed timestamp by at most a declared amount. The returned indexes
can reorder separate immutable records without copying their payloads.

This local frontier is not an authoritative event-time watermark. Use a stream
processor when watermarks must coordinate partitions, survive restarts, bound a
long-lived buffer, trigger windows, retract results, or incorporate explicit
producer progress.

## Implementation

```python
from dataclasses import dataclass
from heapq import heappop, heappush

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_TIMESTAMP_COUNT = 100_000


@dataclass(frozen=True, slots=True)
class TimestampReorderResult:
    emitted_indexes: tuple[int, ...]
    late_indexes: tuple[int, ...]


def reorder_bounded_timestamps(
    timestamps: tuple[int, ...],
    *,
    allowed_lateness: int,
) -> TimestampReorderResult:
    """Return stable emission order and indexes rejected as late."""
    if type(timestamps) is not tuple:
        raise TypeError("timestamps must be an exact tuple")
    if len(timestamps) > _MAX_TIMESTAMP_COUNT:
        raise ValueError("timestamps exceeds the supported count")
    if type(allowed_lateness) is not int:
        raise TypeError("allowed_lateness must be an exact integer")
    if not 0 <= allowed_lateness <= _MAX_INT64:
        raise ValueError("allowed_lateness is outside the supported range")

    for index, timestamp in enumerate(timestamps):
        if type(timestamp) is not int:
            raise TypeError(f"timestamps[{index}] must be an exact integer")
        if not _MIN_INT64 <= timestamp <= _MAX_INT64:
            raise ValueError(f"timestamps[{index}] is outside the signed 64-bit range")

    pending: list[tuple[int, int]] = []
    emitted_indexes: list[int] = []
    late_indexes: list[int] = []
    max_seen: int | None = None

    for arrival_index, timestamp in enumerate(timestamps):
        max_seen = timestamp if max_seen is None else max(max_seen, timestamp)
        frontier = max_seen - allowed_lateness

        if timestamp < frontier:
            late_indexes.append(arrival_index)
        else:
            heappush(pending, (timestamp, arrival_index))

        while pending and pending[0][0] < frontier:
            _, ready_index = heappop(pending)
            emitted_indexes.append(ready_index)

    while pending:
        _, ready_index = heappop(pending)
        emitted_indexes.append(ready_index)

    return TimestampReorderResult(
        emitted_indexes=tuple(emitted_indexes),
        late_indexes=tuple(late_indexes),
    )
```

## Example

```python
ordinary = reorder_bounded_timestamps(
    (10, 8, 9, 12, 7, 12),
    allowed_lateness=2,
)
zero_lateness = reorder_bounded_timestamps(
    (3, 1, 3, 2),
    allowed_lateness=0,
)
wide_boundary = reorder_bounded_timestamps(
    (_MIN_INT64, _MAX_INT64),
    allowed_lateness=_MAX_INT64,
)
empty = reorder_bounded_timestamps((), allowed_lateness=0)

assert (ordinary, zero_lateness, wide_boundary, empty) == (
    TimestampReorderResult((1, 2, 0, 3, 5), (4,)),
    TimestampReorderResult((0, 2), (1, 3)),
    TimestampReorderResult((0, 1), ()),
    TimestampReorderResult((), ()),
)
```

## Trade-offs and Limitations

For `n` arrivals, the function performs `O(n log n)` heap work and stores
`O(n)` indexes in the heap and result. The count limit is also the only buffer
cap: equal timestamps or an overly generous lateness allowance can keep every
admitted record pending until the final drain. Validation is completed before
classification, so malformed input produces no partial result.

Strict comparisons are part of the contract. An arrival exactly on the
frontier is admitted, and a pending record exactly on the frontier is not
released until the frontier advances or input ends. Changing either comparison
would create a different lateness policy.

The result contains indexes rather than record objects and discards no payload
itself. It does not derive lateness from wall-clock time, verify the disorder
assumption, limit memory independently of input size, merge partitions, persist
state, or revise a result when a late record appears.

## Related Snippets

<!-- catalog:related:start -->
- [Fan Out Events into Bounded Lookback Windows](fan-out-events-into-bounded-lookback-windows.md)
- [Partition Strictly Increasing Integer Timestamps into Idle-Gap Sessions](partition-strictly-increasing-integer-timestamps-into-idle-gap-sessions.md)
- [Join Two Strictly Increasing Streams by Exact Timestamp](join-two-strictly-increasing-streams-by-exact-timestamp.md)
<!-- catalog:related:end -->
