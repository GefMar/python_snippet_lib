---
title: "Select Snapshot Representatives by UTC Calendar Buckets"
snippet_type: algorithm
use_cases:
  - automation
  - persistence
tested_python:
  - "3.14"
dependencies: []
related:
  - store-bytes-by-their-content-digest.md
  - ../data-processing/measure-time-in-a-state-within-a-half-open-window.md
---

# Select Snapshot Representatives by UTC Calendar Buckets

## Idea and Problem

Select the newest snapshot from each recent populated UTC day, ISO week, and month without performing deletion as part of the policy.

Each retention tier chooses one deterministic representative per bucket and
the final result is their union. Keeping selection pure makes it possible to
review or log the proposed result before a separate component removes data.

## When to Use

Use this algorithm when snapshots have unique stable identifiers and
timezone-aware creation timestamps, and retention is defined by UTC calendar
buckets rather than elapsed durations. Limits count the newest buckets present
in the input, not every calendar period relative to the current clock. Use an
age-based policy when an empty week or month must still consume retention time.

## Implementation

```python
from collections.abc import Callable, Iterable
from dataclasses import dataclass
from datetime import UTC, datetime


@dataclass(frozen=True, slots=True)
class Snapshot:
    snapshot_id: str
    created_at: datetime


def _retention_limit(value: int, *, name: str) -> int:
    if isinstance(value, bool) or not isinstance(value, int):
        raise TypeError(f"{name} must be an integer")
    if value < 0:
        raise ValueError(f"{name} must not be negative")
    return value


def select_snapshot_representatives(
    snapshots: Iterable[Snapshot],
    *,
    daily: int,
    weekly: int,
    monthly: int,
) -> frozenset[str]:
    limits = {
        "daily": _retention_limit(daily, name="daily"),
        "weekly": _retention_limit(weekly, name="weekly"),
        "monthly": _retention_limit(monthly, name="monthly"),
    }

    normalized: list[tuple[datetime, str]] = []
    identifiers: set[str] = set()
    for snapshot in snapshots:
        if not isinstance(snapshot, Snapshot):
            raise TypeError("snapshots must contain Snapshot values")
        if not isinstance(snapshot.snapshot_id, str):
            raise TypeError("snapshot_id must be text")
        if not snapshot.snapshot_id:
            raise ValueError("snapshot_id must not be empty")
        if snapshot.snapshot_id in identifiers:
            raise ValueError("snapshot_id values must be unique")
        identifiers.add(snapshot.snapshot_id)

        timestamp = snapshot.created_at
        if not isinstance(timestamp, datetime):
            raise TypeError("created_at must be a datetime")
        if timestamp.tzinfo is None or timestamp.utcoffset() is None:
            raise ValueError("created_at must be timezone-aware")
        normalized.append((timestamp.astimezone(UTC), snapshot.snapshot_id))

    newest_first = sorted(normalized, key=lambda item: (item[0], item[1]), reverse=True)
    retained: set[str] = set()

    def select_buckets(
        limit: int,
        bucket_for: Callable[[datetime], object],
    ) -> None:
        represented: set[object] = set()
        for timestamp, snapshot_id in newest_first:
            bucket = bucket_for(timestamp)
            if bucket in represented:
                continue
            if len(represented) == limit:
                break
            represented.add(bucket)
            retained.add(snapshot_id)

    select_buckets(limits["daily"], lambda timestamp: timestamp.date())
    select_buckets(
        limits["weekly"],
        lambda timestamp: (
            timestamp.isocalendar().year,
            timestamp.isocalendar().week,
        ),
    )
    select_buckets(limits["monthly"], lambda timestamp: (timestamp.year, timestamp.month))
    return frozenset(retained)
```

## Example

```python
from datetime import UTC, datetime


snapshots = [
    Snapshot("alpha", datetime(2026, 7, 25, 12, tzinfo=UTC)),
    Snapshot("beta", datetime(2026, 7, 25, 8, tzinfo=UTC)),
    Snapshot("gamma", datetime(2026, 7, 24, 12, tzinfo=UTC)),
    Snapshot("delta", datetime(2026, 7, 17, 12, tzinfo=UTC)),
    Snapshot("epsilon", datetime(2026, 6, 30, 12, tzinfo=UTC)),
    Snapshot("zeta", datetime(2026, 5, 31, 12, tzinfo=UTC)),
]

retained = select_snapshot_representatives(
    reversed(snapshots),
    daily=2,
    weekly=2,
    monthly=2,
)

try:
    select_snapshot_representatives(
        [Snapshot(123, datetime(2026, 7, 25, tzinfo=UTC))],
        daily=1,
        weekly=0,
        monthly=0,
    )
except TypeError:
    invalid_identifier_rejected = True
else:
    invalid_identifier_rejected = False

assert (retained, invalid_identifier_rejected) == (
    frozenset({"alpha", "gamma", "delta", "epsilon"}),
    True,
)
```

## Trade-offs and Limitations

The function materializes and sorts all snapshot metadata, using `O(n)` memory
and `O(n log n)` time. UTC buckets may not match a business calendar in another
timezone, and ISO weeks can belong to a different ISO year than their calendar
date. One snapshot can represent several tiers, so the union may contain fewer
items than the sum of the limits. Equal timestamps choose the
lexicographically greatest identifier. This selects identifiers only: callers
must add dry-run review, authorization, locking, deletion retries, and audit
records around any destructive operation.

## Related Snippets

<!-- catalog:related:start -->
- [Store Bytes by Their Content Digest](store-bytes-by-their-content-digest.md)
- [Measure Time in a State Within a Half-Open Window](../data-processing/measure-time-in-a-state-within-a-half-open-window.md)
<!-- catalog:related:end -->
