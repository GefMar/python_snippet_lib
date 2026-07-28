---
title: "Join Bounded Strictly Increasing Sequences by the Latest Prior Timestamp"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - join-two-strictly-increasing-streams-by-exact-timestamp.md
  - ../algorithms-data-structures/accept-sparse-observations-that-preserve-strict-position-time-order.md
  - overlay-aligned-time-series-at-the-finest-step.md
---

# Join Bounded Strictly Increasing Sequences by the Latest Prior Timestamp

## Idea and Problem

Join each left record to the latest right record whose timestamp is no greater, after completely validating both bounded input sequences.

Because timestamps strictly increase within each side, one forward-moving
right index is sufficient. A right record becomes the current candidate when
its timestamp is at or before the left timestamp, and it can remain the
candidate for several later left records. A left record before every right
record receives no match.

## When to Use

Use this bounded in-memory as-of join for already sorted snapshots such as
measurements enriched with the most recently effective configuration. Equality
across the two sides is eligible, while equal timestamps within one side are
invalid because they would make the choice ambiguous.

Use an exact join when timestamps must be identical, a tolerance-based join
when stale candidates should expire, or a streaming state machine when either
input cannot be held in memory.

## Implementation

```python
from dataclasses import dataclass

_MIN_TIMESTAMP = -(1 << 63)
_MAX_TIMESTAMP = (1 << 63) - 1
_MAX_RECORDS_PER_SIDE = 10_000


@dataclass(frozen=True, slots=True)
class TimedRecord:
    timestamp: int
    payload: object


@dataclass(frozen=True, slots=True)
class LatestPriorMatch:
    left: TimedRecord
    right: TimedRecord | None


def _validated_records(
    value: object,
    *,
    field: str,
) -> tuple[TimedRecord, ...]:
    if type(value) is not tuple:
        raise TypeError(f"{field} must be an exact tuple")
    if len(value) > _MAX_RECORDS_PER_SIDE:
        raise ValueError(f"{field} exceeds the supported limit")

    previous_timestamp: int | None = None
    for record in value:
        if type(record) is not TimedRecord:
            raise TypeError(f"each {field} item must be an exact TimedRecord")
        if type(record.timestamp) is not int:
            raise TypeError(f"each {field} timestamp must be an exact integer")
        if not _MIN_TIMESTAMP <= record.timestamp <= _MAX_TIMESTAMP:
            raise ValueError(f"each {field} timestamp must be in the signed 64-bit range")
        if previous_timestamp is not None and record.timestamp <= previous_timestamp:
            raise ValueError(f"{field} timestamps must strictly increase")
        previous_timestamp = record.timestamp

    return value


def join_by_latest_prior_timestamp(
    left: tuple[TimedRecord, ...],
    right: tuple[TimedRecord, ...],
) -> tuple[LatestPriorMatch, ...]:
    checked_left = _validated_records(left, field="left")
    checked_right = _validated_records(right, field="right")

    matches: list[LatestPriorMatch] = []
    right_index = 0
    latest_right: TimedRecord | None = None

    for left_record in checked_left:
        while (
            right_index < len(checked_right)
            and checked_right[right_index].timestamp <= left_record.timestamp
        ):
            latest_right = checked_right[right_index]
            right_index += 1
        matches.append(LatestPriorMatch(left_record, latest_right))

    return tuple(matches)
```

## Example

```python
left = tuple(TimedRecord(timestamp, f"left-{timestamp}") for timestamp in (1, 3, 4, 7, 9, 12))
right = tuple(TimedRecord(timestamp, f"right-{timestamp}") for timestamp in (2, 4, 8))

matches = join_by_latest_prior_timestamp(left, right)

try:
    join_by_latest_prior_timestamp(
        (TimedRecord(1, "first"), TimedRecord(1, "duplicate")),
        right,
    )
except ValueError:
    duplicate_rejected = True
else:
    duplicate_rejected = False

try:
    join_by_latest_prior_timestamp(
        left,
        (TimedRecord(8, "later"), TimedRecord(2, "earlier")),
    )
except ValueError:
    descending_input_rejected = True
else:
    descending_input_rejected = False

try:
    join_by_latest_prior_timestamp((TimedRecord(True, "invalid"),), right)
except TypeError:
    bool_timestamp_rejected = True
else:
    bool_timestamp_rejected = False

try:
    join_by_latest_prior_timestamp(left, list(right))  # type: ignore[arg-type]
except TypeError:
    list_input_rejected = True
else:
    list_input_rejected = False

assert (
    matches,
    join_by_latest_prior_timestamp((), right),
    matches[2].right is right[1],
    matches[3].right is right[1],
    duplicate_rejected,
    descending_input_rejected,
    bool_timestamp_rejected,
    list_input_rejected,
) == (
    (
        LatestPriorMatch(left[0], None),
        LatestPriorMatch(left[1], right[0]),
        LatestPriorMatch(left[2], right[1]),
        LatestPriorMatch(left[3], right[1]),
        LatestPriorMatch(left[4], right[2]),
        LatestPriorMatch(left[5], right[2]),
    ),
    (),
    True,
    True,
    True,
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

After validation, the join takes `O(L + R)` time because each input record is
visited at most once. The output uses `O(L)` memory, and each bounded input is
retained by its caller. No partial output is constructed unless both sides
pass type, size, range, and ordering validation.

The result retains references to the original records and their payloads;
freezing the records does not deep-freeze mutable payloads. The function has
no tolerance, interpolation, grouping, parsing, deduplication, persistence, or
streaming behavior. Right records after the final left timestamp do not appear
in the output.

## Related Snippets

<!-- catalog:related:start -->
- [Join Two Strictly Increasing Streams by Exact Timestamp](join-two-strictly-increasing-streams-by-exact-timestamp.md)
- [Accept Sparse Observations That Preserve Strict Position-Time Order](../algorithms-data-structures/accept-sparse-observations-that-preserve-strict-position-time-order.md)
- [Overlay Aligned Time Series at the Finest Step](overlay-aligned-time-series-at-the-finest-step.md)
<!-- catalog:related:end -->
