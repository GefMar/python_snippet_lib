---
title: "Split a Half-Open UTC Range Across Ordered Storage Tiers"
snippet_type: algorithm
use_cases:
  - data-transformation
  - persistence
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - select-snapshot-representatives-by-utc-calendar-buckets.md
  - ../algorithms-data-structures/find-a-point-in-disjoint-half-open-intervals.md
  - ../data-processing/measure-time-in-a-state-within-a-half-open-window.md
---

# Split a Half-Open UTC Range Across Ordered Storage Tiers

## Idea and Problem

Intersect one half-open time range with ordered storage-tier cutovers so every instant is routed exactly once without building a query string.

The first tier covers the unbounded past and every later tier begins at an
explicit timezone-aware instant. Inputs and cutovers are normalized to UTC;
the result is an immutable chronological sequence whose adjacent segments
cover `[start, end)` without gaps, overlap, or empty boundary fragments.

## When to Use

Use this algorithm when historical and recent data live behind different
storage adapters and one request may cross their cutovers. Keep the tier policy
small, explicit, and sourced from trusted configuration. Each adapter must use
its returned segment bounds with half-open semantics. Use a database-native union or
partition planner when the storage engine already owns this routing policy.

## Implementation

```python
import re
from collections.abc import Iterable
from dataclasses import dataclass
from datetime import UTC, datetime


_TIER_NAME = re.compile(r"[a-z][a-z0-9_-]{0,31}", re.ASCII)


@dataclass(frozen=True, slots=True)
class StorageTier:
    name: str
    starts_at: datetime | None


@dataclass(frozen=True, slots=True)
class TierRange:
    tier: str
    start: datetime
    end: datetime


def _utc_instant(value: datetime, *, name: str) -> datetime:
    if not isinstance(value, datetime):
        raise TypeError(f"{name} must be a datetime")
    if value.tzinfo is None or value.utcoffset() is None:
        raise ValueError(f"{name} must be timezone-aware")
    return value.astimezone(UTC)


def split_range_across_storage_tiers(
    start: datetime,
    end: datetime,
    tiers: Iterable[StorageTier],
) -> tuple[TierRange, ...]:
    range_start = _utc_instant(start, name="start")
    range_end = _utc_instant(end, name="end")
    if range_start >= range_end:
        raise ValueError("start must precede end")

    normalized: list[tuple[str, datetime | None]] = []
    names: set[str] = set()
    previous_cutover: datetime | None = None
    for index, tier in enumerate(tiers):
        if index >= 32:
            raise ValueError("tiers exceed the supported count")
        if not isinstance(tier, StorageTier):
            raise TypeError("tiers must contain StorageTier values")
        if _TIER_NAME.fullmatch(tier.name) is None:
            raise ValueError("tier name is invalid")
        if tier.name in names:
            raise ValueError("tier names must be unique")
        names.add(tier.name)

        if index == 0:
            if tier.starts_at is not None:
                raise ValueError("the first tier must cover the unbounded past")
            cutover = None
        else:
            if tier.starts_at is None:
                raise ValueError("only the first tier may omit starts_at")
            cutover = _utc_instant(tier.starts_at, name="tier starts_at")
            if previous_cutover is not None and cutover <= previous_cutover:
                raise ValueError("tier cutovers must be strictly increasing")
            previous_cutover = cutover
        normalized.append((tier.name, cutover))

    if not normalized:
        raise ValueError("at least one tier is required")

    result: list[TierRange] = []
    for index, (tier_name, tier_start) in enumerate(normalized):
        next_start = (
            normalized[index + 1][1]
            if index + 1 < len(normalized)
            else None
        )
        segment_start = (
            range_start
            if tier_start is None
            else max(range_start, tier_start)
        )
        segment_end = (
            range_end
            if next_start is None
            else min(range_end, next_start)
        )
        if segment_start < segment_end:
            result.append(TierRange(tier_name, segment_start, segment_end))

    return tuple(result)
```

## Example

```python
tiers = (
    StorageTier("archive", None),
    StorageTier("recent", datetime(2026, 7, 20, tzinfo=UTC)),
    StorageTier("current", datetime(2026, 7, 26, tzinfo=UTC)),
)
segments = split_range_across_storage_tiers(
    datetime(2026, 7, 19, 12, tzinfo=UTC),
    datetime(2026, 7, 27, tzinfo=UTC),
    tiers,
)

assert segments == (
    TierRange(
        "archive",
        datetime(2026, 7, 19, 12, tzinfo=UTC),
        datetime(2026, 7, 20, tzinfo=UTC),
    ),
    TierRange(
        "recent",
        datetime(2026, 7, 20, tzinfo=UTC),
        datetime(2026, 7, 26, tzinfo=UTC),
    ),
    TierRange(
        "current",
        datetime(2026, 7, 26, tzinfo=UTC),
        datetime(2026, 7, 27, tzinfo=UTC),
    ),
)
```

## Trade-offs and Limitations

This is a pure range plan, not a query builder. It does not align bounds to
physical partition keys, verify tier availability or retention, account for
late-arriving data, escape identifiers, or reconcile duplicate records across
stores. UTC normalization deliberately ignores local calendar and daylight
saving boundaries. The policy must contain at most 32 tiers and the first one
must cover the past, while the final tier implicitly has no upper bound.
Changing cutovers between planning and execution can still produce inconsistent
reads, so callers should use one immutable policy snapshot for the whole query.

## Related Snippets

<!-- catalog:related:start -->
- [Select Snapshot Representatives by UTC Calendar Buckets](select-snapshot-representatives-by-utc-calendar-buckets.md)
- [Find a Point in Disjoint Half-Open Intervals](../algorithms-data-structures/find-a-point-in-disjoint-half-open-intervals.md)
- [Measure Time in a State Within a Half-Open Window](../data-processing/measure-time-in-a-state-within-a-half-open-window.md)
<!-- catalog:related:end -->
