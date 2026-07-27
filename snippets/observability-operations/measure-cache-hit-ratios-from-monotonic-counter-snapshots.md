---
title: "Measure Cache Hit Ratios from Monotonic Counter Snapshots"
snippet_type: algorithm
use_cases:
  - observability
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-a-process-cpu-rate-from-two-linux-procfs-samples.md
  - report-partition-offsets-behind-a-fixed-checkpoint.md
  - ../reliability-resilience/cache-values-with-a-monotonic-ttl-and-early-jitter.md
---

# Measure Cache Hit Ratios from Monotonic Counter Snapshots

## Idea and Problem

Derive one cache hit ratio from two immutable cumulative-counter snapshots without coupling the calculation to a cache or metrics system.

Subtracting the earlier counters isolates activity during the observation
interval. Hits and misses are attempted lookups, while skipped operations stay
visible as a separate delta and never enter the ratio denominator. An interval
with no attempted lookups has no meaningful ratio instead of an invented zero.

## When to Use

Use this algorithm when a caller can capture coherent snapshots from the same
set of monotonic counters before and after a bounded interval. Keeping capture
outside the calculation makes recorded observations deterministic to test and
lets the caller choose its own clock, labels, collection schedule, and export
policy.

Treat any decrease as an incomparable reset rather than guessing whether a
counter wrapped or the observed process restarted. Use epoch metadata or a
reset-aware monitoring system when comparisons must span those events.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction

_MAX_COUNTER = (1 << 64) - 1


def _counter(value: object, *, name: str) -> int:
    if type(value) is not int:
        raise TypeError(f"{name} must be an exact integer")
    if not 0 <= value <= _MAX_COUNTER:
        raise ValueError(f"{name} is outside the unsigned 64-bit range")
    return value


@dataclass(frozen=True, slots=True)
class CacheCounterSnapshot:
    hits: int
    misses: int
    skipped: int

    def __post_init__(self) -> None:
        _counter(self.hits, name="hits")
        _counter(self.misses, name="misses")
        _counter(self.skipped, name="skipped")


@dataclass(frozen=True, slots=True)
class CacheHitRatioInterval:
    hit_delta: int
    miss_delta: int
    skipped_delta: int
    attempted_lookups: int
    hit_ratio: Fraction | None


def _validated_snapshot(
    value: object,
    *,
    name: str,
) -> CacheCounterSnapshot:
    if type(value) is not CacheCounterSnapshot:
        raise TypeError(f"{name} must be an exact CacheCounterSnapshot")
    return CacheCounterSnapshot(
        hits=_counter(value.hits, name=f"{name}.hits"),
        misses=_counter(value.misses, name=f"{name}.misses"),
        skipped=_counter(value.skipped, name=f"{name}.skipped"),
    )


def measure_cache_hit_ratio(
    earlier: CacheCounterSnapshot,
    later: CacheCounterSnapshot,
) -> CacheHitRatioInterval:
    first = _validated_snapshot(earlier, name="earlier")
    second = _validated_snapshot(later, name="later")

    hit_delta = second.hits - first.hits
    miss_delta = second.misses - first.misses
    skipped_delta = second.skipped - first.skipped
    if hit_delta < 0 or miss_delta < 0 or skipped_delta < 0:
        raise ValueError("a counter decreased, so the snapshots are incomparable")

    attempted_lookups = hit_delta + miss_delta
    if attempted_lookups > _MAX_COUNTER:
        raise OverflowError("attempted lookup delta exceeds the supported range")
    hit_ratio = Fraction(hit_delta, attempted_lookups) if attempted_lookups else None
    return CacheHitRatioInterval(
        hit_delta=hit_delta,
        miss_delta=miss_delta,
        skipped_delta=skipped_delta,
        attempted_lookups=attempted_lookups,
        hit_ratio=hit_ratio,
    )
```

## Example

```python
earlier = CacheCounterSnapshot(hits=120, misses=30, skipped=10)
later = CacheCounterSnapshot(hits=150, misses=40, skipped=13)

measured = measure_cache_hit_ratio(earlier, later)
skipped_only = measure_cache_hit_ratio(
    later,
    CacheCounterSnapshot(hits=150, misses=40, skipped=18),
)

try:
    measure_cache_hit_ratio(
        later,
        CacheCounterSnapshot(hits=149, misses=41, skipped=18),
    )
except ValueError:
    reset_rejected = True
else:
    reset_rejected = False

assert (measured, skipped_only, reset_rejected) == (
    CacheHitRatioInterval(30, 10, 3, 40, Fraction(3, 4)),
    CacheHitRatioInterval(0, 0, 5, 0, None),
    True,
)
```

## Trade-offs and Limitations

The calculation uses constant time and space, and `Fraction` preserves the
exact integer ratio at the cost of more arithmetic than a float. The combined
attempt count is checked against the same unsigned 64-bit ceiling as each
counter, so a mathematically valid but oversized interval is rejected.

The result describes only the interval between two coherent snapshots. It
does not define a cache, decorator, hook, clock, global collector, label set,
rolling window, or metric sink. A skipped delta is reported for separate
policy decisions but has no effect on the ratio. Capture consistency, reset
epochs, aggregation across instances, and conversion for presentation remain
the caller's responsibility.

## Related Snippets

<!-- catalog:related:start -->
- [Compute a Process CPU Rate from Two Linux procfs Samples](compute-a-process-cpu-rate-from-two-linux-procfs-samples.md)
- [Report Partition Offsets Behind a Fixed Checkpoint](report-partition-offsets-behind-a-fixed-checkpoint.md)
- [Cache Values with a Monotonic TTL and Early Jitter](../reliability-resilience/cache-values-with-a-monotonic-ttl-and-early-jitter.md)
<!-- catalog:related:end -->
