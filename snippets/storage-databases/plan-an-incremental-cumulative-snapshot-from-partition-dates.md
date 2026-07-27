---
title: "Plan an Incremental Cumulative Snapshot from Partition Dates"
snippet_type: algorithm
use_cases:
  - automation
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - select-snapshot-representatives-by-utc-calendar-buckets.md
  - split-a-half-open-utc-range-across-ordered-storage-tiers.md
  - check-whether-a-generated-file-is-older-than-its-inputs.md
---

# Plan an Incremental Cumulative Snapshot from Partition Dates

## Idea and Problem

Build an immutable plan for advancing a cumulative snapshot from exact daily partition dates without reading or writing storage.

The latest daily date is the target, and the latest existing cumulative date is
the optional base. The plan contains every available daily date strictly after
that base in sorted order. Empty daily input and a base already at the target
need no work, while a cumulative snapshot beyond the latest daily partition is
rejected as inconsistent state.

## When to Use

Use this algorithm after a trusted discovery layer has converted partition
names into finite `datetime.date` values. It is suitable when an executor can
start from the latest cumulative snapshot, apply later daily partitions in
date order, and publish a new snapshot for the returned target.

Missing calendar days are accepted because this function knows availability,
not the producer's completeness policy. Add a separate continuity check when
every day is required, and keep storage reads, writes, locking, and atomic
publication in the execution layer.

## Implementation

```python
from collections.abc import Iterable
from dataclasses import dataclass
from datetime import date, datetime
from itertools import islice


_MAX_PARTITION_DATES = 10_000


@dataclass(frozen=True, slots=True)
class CumulativeSnapshotPlan:
    target_date: date
    base_date: date | None
    daily_dates: tuple[date, ...]


def _bounded_unique_dates(
    values: Iterable[date],
    *,
    name: str,
) -> tuple[date, ...]:
    if isinstance(values, (str, bytes)):
        raise TypeError(f"{name} must be a non-text iterable")
    try:
        snapshot = tuple(islice(values, _MAX_PARTITION_DATES + 1))
    except TypeError as error:
        raise TypeError(f"{name} must be an iterable") from error
    if len(snapshot) > _MAX_PARTITION_DATES:
        raise ValueError(f"{name} exceeds the supported item limit")
    if any(type(value) is not date for value in snapshot):
        raise TypeError(f"{name} must contain exact date values")
    if len(set(snapshot)) != len(snapshot):
        raise ValueError(f"{name} must not contain duplicate dates")
    return tuple(sorted(snapshot))


def plan_incremental_cumulative_snapshot(
    daily_dates: Iterable[date],
    cumulative_dates: Iterable[date],
) -> CumulativeSnapshotPlan | None:
    daily = _bounded_unique_dates(daily_dates, name="daily_dates")
    cumulative = _bounded_unique_dates(
        cumulative_dates,
        name="cumulative_dates",
    )
    if not daily:
        return None

    target = daily[-1]
    base = cumulative[-1] if cumulative else None
    if base is not None and base > target:
        raise ValueError("the latest cumulative date is later than the target")
    if base == target:
        return None

    selected_daily = tuple(
        partition_date
        for partition_date in daily
        if base is None or partition_date > base
    )
    return CumulativeSnapshotPlan(
        target_date=target,
        base_date=base,
        daily_dates=selected_daily,
    )
```

## Example

```python
plan = plan_incremental_cumulative_snapshot(
    (
        date(2026, 7, 27),
        date(2026, 7, 21),
        date(2026, 7, 25),
        date(2026, 7, 23),
    ),
    (date(2026, 7, 20), date(2026, 7, 24)),
)

no_daily = plan_incremental_cumulative_snapshot(
    (),
    (date(2026, 7, 24),),
)
already_current = plan_incremental_cumulative_snapshot(
    (date(2026, 7, 26), date(2026, 7, 27)),
    (date(2026, 7, 27),),
)

rejections = []
for daily, cumulative in (
    ((date(2026, 7, 27),), (date(2026, 7, 28),)),
    ((date(2026, 7, 27), date(2026, 7, 27)), ()),
    ((datetime(2026, 7, 27, 12, 0),), ()),
):
    try:
        plan_incremental_cumulative_snapshot(daily, cumulative)
    except (TypeError, ValueError):
        rejections.append(True)

assert (
    plan,
    no_daily,
    already_current,
    len(rejections),
) == (
    CumulativeSnapshotPlan(
        target_date=date(2026, 7, 27),
        base_date=date(2026, 7, 24),
        daily_dates=(date(2026, 7, 25), date(2026, 7, 27)),
    ),
    None,
    None,
    3,
)
```

## Trade-offs and Limitations

Both inputs are independently materialized, capped at 10,000 dates, checked
for duplicates, and sorted, so the function uses linear memory and
`O(n log n)` time. It accepts only exact `date` instances and deliberately
rejects `datetime`, whose time and timezone components would otherwise be
discarded by date-only comparisons.

The latest cumulative date is assumed to be a usable base; older cumulative
dates are retained by the caller but do not appear in the plan. Gaps between
the base and target are accepted, and dates missing from discovery cannot be
distinguished from intentionally absent partitions. The function does not
validate contents, dependencies, schemas, retention, or whether a prior
snapshot was published completely.

This is a pure planner. It never opens partitions, combines records, reserves a
target name, writes output, or removes older snapshots. An executor must define
ordering semantics within each partition, failure recovery, concurrency
control, idempotency, and atomic publication before treating the target as
available.

## Related Snippets

<!-- catalog:related:start -->
- [Select Snapshot Representatives by UTC Calendar Buckets](select-snapshot-representatives-by-utc-calendar-buckets.md)
- [Split a Half-Open UTC Range Across Ordered Storage Tiers](split-a-half-open-utc-range-across-ordered-storage-tiers.md)
- [Check Whether a Generated File Is Older Than Its Inputs](check-whether-a-generated-file-is-older-than-its-inputs.md)
<!-- catalog:related:end -->
