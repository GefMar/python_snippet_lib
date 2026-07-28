---
title: "Build Bounded Digest Summaries Across Explicit Lookback Horizons"
snippet_type: algorithm
use_cases:
  - data-transformation
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - fan-out-events-into-bounded-lookback-windows.md
  - group-items-by-an-exact-compatibility-signature-and-report-unmatched-inputs.md
  - aggregate-consecutive-values-into-weighted-runs.md
---

# Build Bounded Digest Summaries Across Explicit Lookback Horizons

## Idea and Problem

Reduce a finite event snapshot into grouped digests for several explicitly anchored lookback horizons.

For a horizon `h`, an event contributes exactly when its timestamp is in
`[anchor - h, anchor)`. Each populated horizon-and-group pair produces one
immutable summary with a count, timestamp bounds, and a small deterministic
sample of event IDs. Summaries follow declared horizon order and then group-key
order; samples follow timestamp and then event-ID order.

## When to Use

Use this reducer for bounded in-memory reporting where the caller already owns
the event snapshot, anchor, and horizon policy. It combines window membership
and grouped aggregation, so downstream code receives digests rather than the
membership-only fan-out emitted by a labelling helper.

Use a streaming window engine or prefix-aggregate design when the input or
horizon count is large. Define watermark and lateness semantics separately for
continuously arriving events; this function deliberately has no clock reads,
scheduling, persistence, or delivery behavior.

## Implementation

```python
from dataclasses import dataclass, field

_MIN_SIGNED_INTEGER = -(1 << 63)
_MAX_SIGNED_INTEGER = (1 << 63) - 1
_MAX_EVENTS = 10_000
_MAX_HORIZONS = 64
_MAX_EVENT_ID_BYTES = 128
_MAX_GROUP_KEY_BYTES = 64
_MAX_TOTAL_TEXT_BYTES = 512 * 1_024
_MAX_CONTRIBUTIONS = 100_000
_MAX_SUMMARIES = 2_048
_MAX_SAMPLE_IDS_PER_SUMMARY = 8
_MAX_TOTAL_SAMPLE_IDS = 8_192


@dataclass(frozen=True, slots=True)
class DigestEvent:
    event_id: str
    timestamp: int
    group_key: str


@dataclass(frozen=True, slots=True)
class DigestSummary:
    horizon: int
    group_key: str
    count: int
    earliest_timestamp: int
    latest_timestamp: int
    representative_event_ids: tuple[str, ...]


@dataclass(slots=True)
class _Accumulator:
    count: int = 0
    earliest: int | None = None
    latest: int | None = None
    representatives: list[str] = field(default_factory=list)


def _signed_integer(value: object, *, name: str) -> int:
    if type(value) is not int:
        raise TypeError(f"{name} must be an exact integer")
    if not _MIN_SIGNED_INTEGER <= value <= _MAX_SIGNED_INTEGER:
        raise ValueError(f"{name} is outside the signed 64-bit range")
    return value


def _bounded_limit(value: object, *, name: str, maximum: int) -> int:
    if type(value) is not int:
        raise TypeError(f"{name} must be an exact integer")
    if not 0 <= value <= maximum:
        raise ValueError(f"{name} is outside the supported range")
    return value


def _bounded_text(value: object, *, name: str, maximum_bytes: int) -> tuple[str, int]:
    if type(value) is not str:
        raise TypeError(f"{name} must be exact text")
    if not value or value != value.strip() or not value.isprintable():
        raise ValueError(f"{name} must be stripped, non-empty printable text")
    try:
        size = len(value.encode("utf-8"))
    except UnicodeEncodeError as error:
        raise ValueError(f"{name} is not valid UTF-8 text") from error
    if size > maximum_bytes:
        raise ValueError(f"{name} exceeds its UTF-8 byte limit")
    return value, size


def build_digest_summaries(
    events: tuple[DigestEvent, ...],
    *,
    anchor: int,
    horizons: tuple[int, ...],
    sample_size: int = 3,
    max_contributions: int = _MAX_CONTRIBUTIONS,
    max_summaries: int = _MAX_SUMMARIES,
    max_sample_ids: int = _MAX_TOTAL_SAMPLE_IDS,
) -> tuple[DigestSummary, ...]:
    current_anchor = _signed_integer(anchor, name="anchor")
    if type(events) is not tuple:
        raise TypeError("events must be an exact tuple")
    if len(events) > _MAX_EVENTS:
        raise ValueError("event count exceeds the supported limit")
    if type(horizons) is not tuple or not horizons:
        raise TypeError("horizons must be a non-empty exact tuple")
    if len(horizons) > _MAX_HORIZONS:
        raise ValueError("horizon count exceeds the supported limit")

    validated_horizons: list[int] = []
    seen_horizons: set[int] = set()
    for horizon in horizons:
        size = _signed_integer(horizon, name="horizon")
        if size <= 0:
            raise ValueError("horizons must be positive")
        if size in seen_horizons:
            raise ValueError("horizons must be unique")
        seen_horizons.add(size)
        validated_horizons.append(size)

    requested_sample_size = _bounded_limit(
        sample_size,
        name="sample_size",
        maximum=_MAX_SAMPLE_IDS_PER_SUMMARY,
    )
    contribution_limit = _bounded_limit(
        max_contributions,
        name="max_contributions",
        maximum=_MAX_CONTRIBUTIONS,
    )
    summary_limit = _bounded_limit(
        max_summaries,
        name="max_summaries",
        maximum=_MAX_SUMMARIES,
    )
    sample_limit = _bounded_limit(
        max_sample_ids,
        name="max_sample_ids",
        maximum=_MAX_TOTAL_SAMPLE_IDS,
    )

    event_ids: set[str] = set()
    group_keys: set[str] = set()
    total_text_bytes = 0
    for event in events:
        if type(event) is not DigestEvent:
            raise TypeError("events must contain exact DigestEvent values")
        event_id, event_id_bytes = _bounded_text(
            event.event_id,
            name="event_id",
            maximum_bytes=_MAX_EVENT_ID_BYTES,
        )
        group_key, group_key_bytes = _bounded_text(
            event.group_key,
            name="group_key",
            maximum_bytes=_MAX_GROUP_KEY_BYTES,
        )
        _signed_integer(event.timestamp, name="event timestamp")
        if event_id in event_ids:
            raise ValueError("event IDs must be unique")
        event_ids.add(event_id)
        group_keys.add(group_key)
        total_text_bytes += event_id_bytes + group_key_bytes
        if total_text_bytes > _MAX_TOTAL_TEXT_BYTES:
            raise ValueError("events exceed the aggregate UTF-8 text limit")

    worst_contributions = len(events) * len(validated_horizons)
    if worst_contributions > contribution_limit:
        raise ValueError("worst-case event contributions exceed max_contributions")
    worst_summaries = len(group_keys) * len(validated_horizons)
    if worst_summaries > summary_limit:
        raise ValueError("worst-case summaries exceed max_summaries")
    if worst_summaries * requested_sample_size > sample_limit:
        raise ValueError("worst-case representative IDs exceed max_sample_ids")

    buckets: dict[tuple[int, str], _Accumulator] = {}
    ordered_events = sorted(events, key=lambda event: (event.timestamp, event.event_id))
    for event in ordered_events:
        for horizon_index, horizon in enumerate(validated_horizons):
            if not current_anchor - horizon <= event.timestamp < current_anchor:
                continue
            accumulator = buckets.setdefault(
                (horizon_index, event.group_key),
                _Accumulator(),
            )
            accumulator.count += 1
            if accumulator.earliest is None:
                accumulator.earliest = event.timestamp
            accumulator.latest = event.timestamp
            if len(accumulator.representatives) < requested_sample_size:
                accumulator.representatives.append(event.event_id)

    summaries: list[DigestSummary] = []
    for horizon_index, horizon in enumerate(validated_horizons):
        keys = sorted(
            group_key
            for bucket_index, group_key in buckets
            if bucket_index == horizon_index
        )
        for group_key in keys:
            accumulator = buckets[(horizon_index, group_key)]
            if accumulator.earliest is None or accumulator.latest is None:
                raise AssertionError("a populated digest must have timestamp bounds")
            summaries.append(
                DigestSummary(
                    horizon=horizon,
                    group_key=group_key,
                    count=accumulator.count,
                    earliest_timestamp=accumulator.earliest,
                    latest_timestamp=accumulator.latest,
                    representative_event_ids=tuple(accumulator.representatives),
                )
            )
    return tuple(summaries)
```

## Example

```python
events = (
    DigestEvent("evt-old", 100, "blue"),
    DigestEvent("evt-b", 750, "amber"),
    DigestEvent("evt-a", 750, "amber"),
    DigestEvent("evt-late", 999, "blue"),
    DigestEvent("evt-anchor", 1_000, "amber"),
)

summaries = build_digest_summaries(
    events,
    anchor=1_000,
    horizons=(300, 900),
    sample_size=2,
    max_contributions=10,
    max_summaries=4,
    max_sample_ids=8,
)

assert tuple(
    (
        summary.horizon,
        summary.group_key,
        summary.count,
        summary.earliest_timestamp,
        summary.latest_timestamp,
        summary.representative_event_ids,
    )
    for summary in summaries
) == (
    (300, "amber", 2, 750, 750, ("evt-a", "evt-b")),
    (300, "blue", 1, 999, 999, ("evt-late",)),
    (900, "amber", 2, 750, 750, ("evt-a", "evt-b")),
    (900, "blue", 2, 100, 999, ("evt-old", "evt-late")),
)

try:
    build_digest_summaries((*events, events[0]), anchor=1_000, horizons=(300,))
except ValueError:
    duplicate_id_rejected = True
else:
    duplicate_id_rejected = False

assert duplicate_id_rejected
```

## Trade-offs and Limitations

Validation is completed before aggregation. The conservative event-by-horizon,
horizon-by-distinct-group, and summary-by-sample products may reject a batch
whose timestamps would yield a much smaller actual result. This makes output
memory independent of favourable input values. Runtime is
`O(events log events + events * horizons)` and storage is bounded by the input,
actual contributions, summaries, and samples.

Samples are deterministic diagnostics, not random or statistically
representative samples. Text uses exact Unicode code points without
normalization, and group keys use Python's stable lexicographic order. The
reducer does not parse timestamps, deduplicate by payload, read a clock, manage
TTL or locks, destructively drain input, persist results, schedule work, or
deliver notifications.

## Related Snippets

<!-- catalog:related:start -->
- [Fan Out Events into Bounded Lookback Windows](fan-out-events-into-bounded-lookback-windows.md)
- [Group Items by an Exact Compatibility Signature and Report Unmatched Inputs](group-items-by-an-exact-compatibility-signature-and-report-unmatched-inputs.md)
- [Aggregate Consecutive Values into Weighted Runs](aggregate-consecutive-values-into-weighted-runs.md)
<!-- catalog:related:end -->
