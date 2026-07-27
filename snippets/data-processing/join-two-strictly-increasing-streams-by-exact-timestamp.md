---
title: "Join Two Strictly Increasing Streams by Exact Timestamp"
snippet_type: pattern
use_cases:
  - data-transformation
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../algorithms-data-structures/accept-sparse-observations-that-preserve-strict-position-time-order.md
  - yield-stream-items-with-bounded-neighbor-context.md
  - group-items-by-an-exact-compatibility-signature-and-report-unmatched-inputs.md
---

# Join Two Strictly Increasing Streams by Exact Timestamp

## Idea and Problem

Incrementally join two streams whose timestamps are strictly increasing within each side, while bounding the number of records waiting for an exact counterpart.

Each push returns immutable matches and records that have just become
impossible to match. Advancing one side past a pending timestamp proves that
the older record is unmatched. A newly received record is also immediately
unmatched when the other side has already advanced past it. No callback runs
inside the state transition.

## When to Use

Use this pattern when two independently delivered, already ordered streams use
the same exact integer timestamp as a correlation key. It fits a single-owner
consumer that can process every returned batch before making the next push and
can choose an explicit pending-record limit for temporary skew.

Reorder upstream data first when either side can move backward. Use a
windowed or tolerance-based join when nearby timestamps should match, and use a
durable state store when pending records must survive a process restart.

## Implementation

```python
from collections import deque
from dataclasses import dataclass
from typing import Literal


_MIN_TIMESTAMP = -(1 << 63)
_MAX_TIMESTAMP = (1 << 63) - 1
_MAX_PENDING = 10_000


@dataclass(frozen=True, slots=True)
class TimestampMatch:
    timestamp: int
    left: object
    right: object


@dataclass(frozen=True, slots=True)
class UnmatchedTimestamp:
    side: Literal["left", "right"]
    timestamp: int
    value: object


@dataclass(frozen=True, slots=True)
class TimestampJoinBatch:
    matches: tuple[TimestampMatch, ...] = ()
    unmatched: tuple[UnmatchedTimestamp, ...] = ()


@dataclass(frozen=True, slots=True)
class _Pending:
    timestamp: int
    value: object


def _timestamp(value: object) -> int:
    if type(value) is not int:
        raise TypeError("timestamp must be an exact integer")
    if not _MIN_TIMESTAMP <= value <= _MAX_TIMESTAMP:
        raise ValueError("timestamp is outside the signed 64-bit range")
    return value


class IncreasingTimestampJoiner:
    def __init__(self, *, max_pending: int = 1_000) -> None:
        if type(max_pending) is not int:
            raise TypeError("max_pending must be an exact integer")
        if not 1 <= max_pending <= _MAX_PENDING:
            raise ValueError("max_pending is outside the supported range")

        self._max_pending = max_pending
        self._left: deque[_Pending] = deque()
        self._right: deque[_Pending] = deque()
        self._last_left: int | None = None
        self._last_right: int | None = None
        self._closed = False

    @property
    def pending_count(self) -> int:
        return len(self._left) + len(self._right)

    def push_left(self, timestamp: int, value: object) -> TimestampJoinBatch:
        return self._push("left", timestamp, value)

    def push_right(self, timestamp: int, value: object) -> TimestampJoinBatch:
        return self._push("right", timestamp, value)

    def _push(
        self,
        side: Literal["left", "right"],
        timestamp: int,
        value: object,
    ) -> TimestampJoinBatch:
        if self._closed:
            raise RuntimeError("the joiner is closed")
        current_timestamp = _timestamp(timestamp)

        if side == "left":
            own = self._left
            opposite = self._right
            own_last = self._last_left
            opposite_last = self._last_right
            opposite_side: Literal["left", "right"] = "right"
        else:
            own = self._right
            opposite = self._left
            own_last = self._last_right
            opposite_last = self._last_left
            opposite_side = "left"

        if own_last is not None and current_timestamp <= own_last:
            raise ValueError("timestamps on each side must strictly increase")

        expired_count = 0
        exact_match = False
        for pending in opposite:
            if pending.timestamp < current_timestamp:
                expired_count += 1
                continue
            exact_match = pending.timestamp == current_timestamp
            break

        already_passed = (
            not exact_match
            and opposite_last is not None
            and opposite_last >= current_timestamp
        )
        pending_after_push = (
            self.pending_count
            - expired_count
            - int(exact_match)
            + int(not exact_match and not already_passed)
        )
        if pending_after_push > self._max_pending:
            raise ValueError("pending records would exceed max_pending")

        unmatched = []
        for _ in range(expired_count):
            expired = opposite.popleft()
            unmatched.append(
                UnmatchedTimestamp(
                    opposite_side,
                    expired.timestamp,
                    expired.value,
                )
            )

        matches = []
        if exact_match:
            matched = opposite.popleft()
            if side == "left":
                matches.append(
                    TimestampMatch(current_timestamp, value, matched.value)
                )
            else:
                matches.append(
                    TimestampMatch(current_timestamp, matched.value, value)
                )
        elif already_passed:
            unmatched.append(UnmatchedTimestamp(side, current_timestamp, value))
        else:
            own.append(_Pending(current_timestamp, value))

        if side == "left":
            self._last_left = current_timestamp
        else:
            self._last_right = current_timestamp
        return TimestampJoinBatch(tuple(matches), tuple(unmatched))

    def finish(self) -> TimestampJoinBatch:
        if self._closed:
            raise RuntimeError("the joiner is closed")

        remaining = [
            *(UnmatchedTimestamp("left", item.timestamp, item.value)
              for item in self._left),
            *(UnmatchedTimestamp("right", item.timestamp, item.value)
              for item in self._right),
        ]
        remaining.sort(key=lambda item: (item.timestamp, item.side))
        self._left.clear()
        self._right.clear()
        self._closed = True
        return TimestampJoinBatch(unmatched=tuple(remaining))
```

## Example

```python
joiner = IncreasingTimestampJoiner(max_pending=2)

assert joiner.push_left(10, "left-10") == TimestampJoinBatch()
assert joiner.push_right(8, "right-8") == TimestampJoinBatch(
    unmatched=(UnmatchedTimestamp("right", 8, "right-8"),)
)
assert joiner.push_right(10, "right-10") == TimestampJoinBatch(
    matches=(TimestampMatch(10, "left-10", "right-10"),)
)
assert joiner.push_left(12, "left-12") == TimestampJoinBatch()
assert joiner.push_right(15, "right-15") == TimestampJoinBatch(
    unmatched=(UnmatchedTimestamp("left", 12, "left-12"),)
)
assert joiner.finish() == TimestampJoinBatch(
    unmatched=(UnmatchedTimestamp("right", 15, "right-15"),)
)

rollback = IncreasingTimestampJoiner(max_pending=1)
rollback.push_left(20, "kept")
try:
    rollback.push_left(21, "too-many")
except ValueError:
    capacity_rejected = True
else:
    capacity_rejected = False

assert capacity_rejected
assert rollback.push_right(20, "counterpart") == TimestampJoinBatch(
    matches=(TimestampMatch(20, "kept", "counterpart"),)
)
```

## Trade-offs and Limitations

Each pending record is appended and removed once. A push may emit a prefix of
the opposite queue, so its immediate cost is `O(emitted)`; total queue work is
linear across the stream. The pending-record count is checked before any
queue, watermark, or last-seen timestamp is changed, making a capacity failure
safe to retry after the caller resolves pressure.

Payloads are retained by reference and are not copied or validated. The
joiner is mutable, single-owner, and not thread-safe. It provides no parsing,
deduplication, approximate matching, timeout, persistence, callback delivery,
or recovery protocol. `finish()` deterministically flushes both sides and
permanently closes the instance.

## Related Snippets

<!-- catalog:related:start -->
- [Accept Sparse Observations That Preserve Strict Position-Time Order](../algorithms-data-structures/accept-sparse-observations-that-preserve-strict-position-time-order.md)
- [Yield Stream Items with Bounded Neighbor Context](yield-stream-items-with-bounded-neighbor-context.md)
- [Group Items by an Exact Compatibility Signature and Report Unmatched Inputs](group-items-by-an-exact-compatibility-signature-and-report-unmatched-inputs.md)
<!-- catalog:related:end -->
