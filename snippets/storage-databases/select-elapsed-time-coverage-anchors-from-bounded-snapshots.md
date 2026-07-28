---
title: "Select Elapsed-Time Coverage Anchors from Bounded Snapshots"
snippet_type: algorithm
use_cases:
  - automation
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - plan-an-incremental-cumulative-snapshot-from-partition-dates.md
  - decide-whether-to-restore-a-versioned-snapshot.md
  - ../data-processing/measure-time-in-a-state-within-a-half-open-window.md
---

# Select Elapsed-Time Coverage Anchors from Bounded Snapshots

## Idea and Problem

Choose a deterministic subset of timestamped snapshots so every omitted item is covered by a selected anchor that precedes it in the defined order and is at most the inclusive elapsed-time horizon later.

The first item in timestamp-descending, identifier-ascending order is always selected. Each
later item is compared only with the most recently selected preceding anchor: it is skipped when
their timestamp gap is at most the horizon and selected when the gap is greater. Equal
timestamps are therefore handled deterministically without depending on input order.

## When to Use

Use this algorithm when a bounded metadata snapshot is already available as immutable records
with unique identifiers and exact UTC timestamps. It is useful for making a reviewable,
repeatable spacing decision based only on elapsed time supplied by the caller.

The horizon is a duration, not a named period. The function has no implicit notion of the
current moment and does not change the supplied records or perform external actions.

## Implementation

```python
from dataclasses import dataclass
from datetime import UTC, datetime, timedelta

_MAX_SNAPSHOTS = 25_000
_MAX_IDENTIFIER_BYTES = 96
_MAX_TOTAL_IDENTIFIER_BYTES = 2 * 1_024 * 1_024
_MAX_HORIZON = timedelta(days=3_660)


@dataclass(frozen=True, slots=True)
class CoverageAnchor:
    anchor_id: str
    captured_at: datetime


@dataclass(frozen=True, slots=True)
class CoverageSelection:
    selected: tuple[CoverageAnchor, ...]
    skipped: tuple[CoverageAnchor, ...]


def _identifier_size(value: object, field: str) -> int:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if not value:
        raise ValueError(f"{field} must not be empty")
    try:
        size = len(value.encode("utf-8"))
    except UnicodeEncodeError as error:
        raise ValueError(f"{field} must be valid UTF-8 text") from error
    if size > _MAX_IDENTIFIER_BYTES:
        raise ValueError(f"{field} exceeds the identifier byte limit")
    return size


def select_coverage_anchors(
    snapshots: tuple[CoverageAnchor, ...],
    horizon: timedelta,
) -> CoverageSelection:
    if type(snapshots) is not tuple:
        raise TypeError("snapshots must be an exact tuple")
    if len(snapshots) > _MAX_SNAPSHOTS:
        raise ValueError("snapshots exceeds the item-count limit")
    if type(horizon) is not timedelta:
        raise TypeError("horizon must be an exact timedelta")
    if not timedelta(0) < horizon <= _MAX_HORIZON:
        raise ValueError("horizon must be positive and within the supported bound")

    identifiers: set[str] = set()
    identifier_bytes = 0
    for position, snapshot in enumerate(snapshots):
        field = f"snapshots[{position}]"
        if type(snapshot) is not CoverageAnchor:
            raise TypeError(f"{field} must be an exact CoverageAnchor")
        identifier_bytes += _identifier_size(snapshot.anchor_id, f"{field}.anchor_id")
        if identifier_bytes > _MAX_TOTAL_IDENTIFIER_BYTES:
            raise ValueError("snapshots exceeds the aggregate identifier byte limit")
        if snapshot.anchor_id in identifiers:
            raise ValueError("anchor_id values must be unique")
        identifiers.add(snapshot.anchor_id)
        if type(snapshot.captured_at) is not datetime:
            raise TypeError(f"{field}.captured_at must be an exact datetime")
        if snapshot.captured_at.tzinfo is not UTC:
            raise ValueError(f"{field}.captured_at must use the exact UTC timezone")

    newest_first = sorted(snapshots, key=lambda snapshot: snapshot.anchor_id)
    newest_first.sort(key=lambda snapshot: snapshot.captured_at, reverse=True)

    selected: list[CoverageAnchor] = []
    skipped: list[CoverageAnchor] = []
    for snapshot in newest_first:
        if selected and selected[-1].captured_at - snapshot.captured_at <= horizon:
            skipped.append(snapshot)
        else:
            selected.append(snapshot)

    return CoverageSelection(selected=tuple(selected), skipped=tuple(skipped))
```

## Example

```python
horizon = timedelta(minutes=12)
snapshots = (
    CoverageAnchor("kelp", datetime(2028, 4, 9, 11, 31, tzinfo=UTC)),
    CoverageAnchor("nacre", datetime(2028, 4, 9, 12, 0, tzinfo=UTC)),
    CoverageAnchor("quartz", datetime(2028, 4, 9, 11, 20, tzinfo=UTC)),
    CoverageAnchor("cedar", datetime(2028, 4, 9, 11, 53, tzinfo=UTC)),
    CoverageAnchor("frost", datetime(2028, 4, 9, 11, 40, tzinfo=UTC)),
    CoverageAnchor("agate", datetime(2028, 4, 9, 12, 0, tzinfo=UTC)),
)

result = select_coverage_anchors(snapshots, horizon)

assert tuple(item.anchor_id for item in result.selected) == ("agate", "frost", "quartz")
assert tuple(item.anchor_id for item in result.skipped) == ("nacre", "cedar", "kelp")
assert all(
    any(
        anchor.captured_at >= item.captured_at and anchor.captured_at - item.captured_at <= horizon
        for anchor in result.selected
    )
    for item in result.skipped
)
assert all(
    result.selected[position].captured_at - result.selected[position + 1].captured_at > horizon
    for position in range(len(result.selected) - 1)
)
```

## Trade-offs and Limitations

Validation is `O(n)`, followed by `O(n log n)` sorting and `O(n)` result materialization. The
two stable sorts produce newest-first order with identifiers ascending at equal timestamps.
Identifiers are preserved exactly, so their ordering follows Python string ordering. The
supported horizon, record count, per-identifier bytes, and aggregate identifier bytes are all
intentionally finite.

This greedy rule guarantees elapsed-time coverage relative to the most recently selected
preceding anchor, which is newer or has the same timestamp and a lower identifier. Adjacent
selected anchors have timestamp gaps greater than the horizon. The rule does not optimize a
different global objective, infer missing snapshots, consult an external time source, or apply
the classification anywhere.

## Related Snippets

<!-- catalog:related:start -->
- [Plan an Incremental Cumulative Snapshot from Partition Dates](plan-an-incremental-cumulative-snapshot-from-partition-dates.md)
- [Decide Whether to Restore a Versioned Snapshot](decide-whether-to-restore-a-versioned-snapshot.md)
- [Measure Time in a State Within a Half-Open Window](../data-processing/measure-time-in-a-state-within-a-half-open-window.md)
<!-- catalog:related:end -->
