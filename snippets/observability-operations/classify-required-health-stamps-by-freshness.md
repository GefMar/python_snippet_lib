---
title: "Classify Required Health Stamps by Freshness"
snippet_type: recipe
use_cases:
  - observability
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - resolve-the-latest-status-with-an-explicit-mapping.md
  - ../data-processing/measure-time-in-a-state-within-a-half-open-window.md
---

# Classify Required Health Stamps by Freshness

## Idea and Problem

Classify every required source as fresh, stale, or missing from its latest successful timestamp instead of running expensive checks in the read path.

The evaluator receives a fixed aware `now`, a positive freshness window, and
an ordered set of required source names. It validates every stamp, keeps the
latest success for each source, and returns one result per requirement in the
declared order. A stamp exactly on the lower time boundary is still fresh;
future timestamps are rejected rather than treated as healthy. Comparisons
normalize aware timestamps to UTC so daylight-saving folds are ordered by
their actual instants instead of repeated wall-clock values.

## When to Use

Use this pure read-side policy when checks run separately and publish
last-success stamps to storage. Choose a freshness window longer than the
normal check interval plus expected scheduling delay. Keep failed-check details
elsewhere: not updating a success stamp eventually makes it stale, but does
not explain why the latest check failed.

## Implementation

```python
from collections.abc import Iterable, Sequence
from dataclasses import dataclass
from datetime import UTC, datetime, timedelta
from enum import StrEnum


class Freshness(StrEnum):
    FRESH = "fresh"
    STALE = "stale"
    MISSING = "missing"


@dataclass(frozen=True, slots=True)
class HealthStamp:
    source: str
    succeeded_at: datetime


@dataclass(frozen=True, slots=True)
class HealthResult:
    source: str
    freshness: Freshness
    last_success: datetime | None


def _require_aware(value: datetime, *, name: str) -> None:
    if not isinstance(value, datetime):
        raise TypeError(f"{name} must be a datetime")
    if value.tzinfo is None or value.utcoffset() is None:
        raise ValueError(f"{name} must be timezone-aware")


def _as_utc(value: datetime, *, name: str) -> datetime:
    _require_aware(value, name=name)
    try:
        return value.astimezone(UTC)
    except OverflowError as error:
        raise ValueError(f"{name} cannot be represented in UTC") from error


def classify_health_stamps(
    required_sources: Sequence[str],
    stamps: Iterable[HealthStamp],
    *,
    now: datetime,
    max_age: timedelta,
) -> tuple[HealthResult, ...]:
    if isinstance(required_sources, (str, bytes)) or not isinstance(
        required_sources,
        Sequence,
    ):
        raise TypeError("required_sources must be a sequence of text values")
    ordered_sources = tuple(required_sources)
    if not ordered_sources:
        raise ValueError("at least one required source is needed")
    if any(
        not isinstance(source, str) or not source.strip()
        for source in ordered_sources
    ):
        raise ValueError("required sources must be non-empty text")
    if len(set(ordered_sources)) != len(ordered_sources):
        raise ValueError("required sources must be unique")
    current_utc = _as_utc(now, name="now")
    if not isinstance(max_age, timedelta):
        raise TypeError("max_age must be a timedelta")
    if max_age <= timedelta(0):
        raise ValueError("max_age must be positive")

    required = set(ordered_sources)
    latest: dict[str, tuple[datetime, datetime]] = {}
    for stamp in stamps:
        if not isinstance(stamp, HealthStamp):
            raise TypeError("stamps must contain HealthStamp values")
        if not isinstance(stamp.source, str) or not stamp.source.strip():
            raise ValueError("stamp sources must be non-empty text")
        if stamp.source not in required:
            raise ValueError("encountered an unexpected health source")
        stamp_utc = _as_utc(stamp.succeeded_at, name="stamp.succeeded_at")
        if stamp_utc > current_utc:
            raise ValueError("health stamps must not be in the future")
        previous = latest.get(stamp.source)
        if previous is None or stamp_utc > previous[0]:
            latest[stamp.source] = (stamp_utc, stamp.succeeded_at)

    return tuple(
        HealthResult(
            source=source,
            freshness=(
                Freshness.MISSING
                if source not in latest
                else Freshness.FRESH
                if current_utc - latest[source][0] <= max_age
                else Freshness.STALE
            ),
            last_success=(latest[source][1] if source in latest else None),
        )
        for source in ordered_sources
    )
```

## Example

```python
from datetime import UTC, timezone


current_time = datetime(2026, 7, 1, 12, 0, tzinfo=UTC)
results = classify_health_stamps(
    ("search", "cache", "queue"),
    (
        HealthStamp("search", datetime(2026, 7, 1, 11, 40, tzinfo=UTC)),
        HealthStamp("cache", datetime(2026, 7, 1, 12, 54, tzinfo=timezone(timedelta(hours=1)))),
        HealthStamp("search", datetime(2026, 7, 1, 11, 55, tzinfo=UTC)),
    ),
    now=current_time,
    max_age=timedelta(minutes=5),
)

try:
    classify_health_stamps(
        ("search",),
        (HealthStamp("search", current_time + timedelta(seconds=1)),),
        now=current_time,
        max_age=timedelta(minutes=5),
    )
except ValueError:
    future_rejected = True
else:
    future_rejected = False

assert (
    tuple((result.source, result.freshness) for result in results),
    results[0].last_success,
    future_rejected,
) == (
    (
        ("search", Freshness.FRESH),
        ("cache", Freshness.STALE),
        ("queue", Freshness.MISSING),
    ),
    datetime(2026, 7, 1, 11, 55, tzinfo=UTC),
    True,
)
```

## Trade-offs and Limitations

This policy stores only the latest timestamp per required source, using `O(r)`
memory for `r` requirements. Rejecting future stamps makes clock skew visible
but can turn a small producer-clock error into a failed read; synchronize
clocks or define a separate skew policy when needed. Last-success freshness is
not proof that a dependency is healthy now, and a long window delays failure
detection. The function does not run checks, aggregate the results into one
boolean, persist stamps, report failure causes, or coordinate concurrent
writers.

## Related Snippets

<!-- catalog:related:start -->
- [Resolve the Latest Status with an Explicit Mapping](resolve-the-latest-status-with-an-explicit-mapping.md)
- [Measure Time in a State Within a Half-Open Window](../data-processing/measure-time-in-a-state-within-a-half-open-window.md)
<!-- catalog:related:end -->
