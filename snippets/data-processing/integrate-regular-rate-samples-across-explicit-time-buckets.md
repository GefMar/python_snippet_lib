---
title: "Integrate Regular Rate Samples Across Explicit Time Buckets"
snippet_type: algorithm
use_cases:
  - data-transformation
  - observability
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - overlay-aligned-time-series-at-the-finest-step.md
  - measure-time-in-a-state-within-a-half-open-window.md
  - ../observability-operations/count-values-in-fixed-upper-bound-bins.md
---

# Integrate Regular Rate Samples Across Explicit Time Buckets

## Idea and Problem

Integrate regular piecewise-constant rates into explicit half-open time buckets without treating missing intervals as zero.

Each rate covers one regular interval `[start, start + step)`. For every
intersection with an output bucket, the algorithm adds
`rate * overlap_seconds` and records how many seconds were actually observed.
A missing rate contributes neither a count nor observed time, while a numeric
zero produces a present zero count.

## When to Use

Use this algorithm when finite non-negative rate samples already share one
integer-second timeline and reporting buckets are supplied as an explicit
increasing boundary tuple. It handles buckets that cut through sample intervals
and reports partial coverage without inventing values for gaps.

Do not use it for per-sample counts, irregular observations, interpolation,
calendar arithmetic, or implicit resampling. Normalize timestamps, units, and
rate semantics before calling it.

## Implementation

```python
import math
from dataclasses import dataclass


_MIN_TIMELINE_VALUE = -(1 << 63)
_MAX_TIMELINE_VALUE = (1 << 63) - 1
_MAX_RATE_SAMPLES = 100_000
_MAX_BUCKETS = 100_000
_MAX_OVERLAPS = 200_000


def _timeline_integer(value: object, *, name: str) -> int:
    if type(value) is not int:
        raise TypeError(f"{name} must be an integer")
    if not _MIN_TIMELINE_VALUE <= value <= _MAX_TIMELINE_VALUE:
        raise ValueError(f"{name} is outside the supported timeline range")
    return value


@dataclass(frozen=True, slots=True)
class RegularRateSeries:
    start: int
    step: int
    rates: tuple[float | None, ...]

    def __post_init__(self) -> None:
        _timeline_integer(self.start, name="start")
        if type(self.step) is not int:
            raise TypeError("step must be an integer")
        if not 1 <= self.step <= _MAX_TIMELINE_VALUE:
            raise ValueError("step is outside the supported range")
        if type(self.rates) is not tuple:
            raise TypeError("rates must be a tuple")
        if not 1 <= len(self.rates) <= _MAX_RATE_SAMPLES:
            raise ValueError("rate count is outside the supported range")
        for rate in self.rates:
            if rate is None:
                continue
            if type(rate) is not float:
                raise TypeError("rates must contain floats or None")
            if not math.isfinite(rate):
                raise ValueError("rates must be finite")
            if rate < 0.0:
                raise ValueError("rates must be non-negative")

        stop = self.start + self.step * len(self.rates)
        if not _MIN_TIMELINE_VALUE <= stop <= _MAX_TIMELINE_VALUE:
            raise ValueError("series endpoint is outside the supported timeline range")

    @property
    def stop(self) -> int:
        return self.start + self.step * len(self.rates)


@dataclass(frozen=True, slots=True)
class IntegratedRateBucket:
    start: int
    end: int
    count: float | None
    observed_seconds: int


def integrate_regular_rates(
    series: RegularRateSeries,
    bucket_boundaries: tuple[int, ...],
) -> tuple[IntegratedRateBucket, ...]:
    if type(series) is not RegularRateSeries:
        raise TypeError("series must be a RegularRateSeries")
    if type(bucket_boundaries) is not tuple:
        raise TypeError("bucket_boundaries must be a tuple")
    if not 2 <= len(bucket_boundaries) <= _MAX_BUCKETS + 1:
        raise ValueError("bucket count is outside the supported range")

    for boundary in bucket_boundaries:
        _timeline_integer(boundary, name="bucket boundary")
    for left, right in zip(bucket_boundaries, bucket_boundaries[1:]):
        if right <= left:
            raise ValueError("bucket boundaries must be strictly increasing")
        if right - left > _MAX_TIMELINE_VALUE:
            raise ValueError("bucket duration exceeds the supported range")

    bucket_count = len(bucket_boundaries) - 1
    totals = [0.0] * bucket_count
    observed = [0] * bucket_count
    sample_index = 0
    bucket_index = 0
    overlap_count = 0

    while sample_index < len(series.rates) and bucket_index < bucket_count:
        sample_start = series.start + sample_index * series.step
        sample_end = sample_start + series.step
        bucket_start = bucket_boundaries[bucket_index]
        bucket_end = bucket_boundaries[bucket_index + 1]
        overlap_start = max(sample_start, bucket_start)
        overlap_end = min(sample_end, bucket_end)

        if overlap_start < overlap_end:
            overlap_count += 1
            if overlap_count > _MAX_OVERLAPS:
                raise ValueError("sample-bucket overlaps exceed the supported limit")

            rate = series.rates[sample_index]
            if rate is not None:
                overlap_seconds = overlap_end - overlap_start
                contribution = rate * overlap_seconds
                if not math.isfinite(contribution):
                    raise OverflowError("an integrated contribution is not finite")
                updated_total = totals[bucket_index] + contribution
                if not math.isfinite(updated_total):
                    raise OverflowError("an integrated bucket total is not finite")
                totals[bucket_index] = updated_total
                observed[bucket_index] += overlap_seconds

        if sample_end <= bucket_end:
            sample_index += 1
        if bucket_end <= sample_end:
            bucket_index += 1

    return tuple(
        IntegratedRateBucket(
            start=bucket_boundaries[index],
            end=bucket_boundaries[index + 1],
            count=totals[index] if observed[index] else None,
            observed_seconds=observed[index],
        )
        for index in range(bucket_count)
    )
```

## Example

```python
series = RegularRateSeries(
    start=0,
    step=4,
    rates=(2.0, None, 0.0, 1.5),
)
buckets = integrate_regular_rates(series, (0, 3, 6, 10, 16, 18))

try:
    integrate_regular_rates(
        RegularRateSeries(start=0, step=2, rates=(1e308,)),
        (0, 2),
    )
except OverflowError:
    overflow_rejected = True
else:
    overflow_rejected = False

assert buckets == (
    IntegratedRateBucket(0, 3, 6.0, 3),
    IntegratedRateBucket(3, 6, 2.0, 1),
    IntegratedRateBucket(6, 10, 0.0, 2),
    IntegratedRateBucket(10, 16, 6.0, 6),
    IntegratedRateBucket(16, 18, None, 0),
)
assert sum(bucket.count or 0.0 for bucket in buckets) == 14.0
assert overflow_rejected
```

## Trade-offs and Limitations

The two-pointer traversal costs `O(samples + buckets + overlaps)` time and
stores one frozen result per bucket. Input and overlap caps bound work before a
large accidental expansion. A bucket can contain a valid partial count; use
`observed_seconds` to distinguish that case from complete coverage.

Binary floating-point multiplication and addition are not exact decimal
arithmetic, and any negative or non-finite input, non-finite contribution, or
non-finite accumulated total is rejected. `None` means missing and remains
different from a measured `0.0`.
The function performs no clock access, storage, timestamp parsing,
interpolation, gap filling, resampling, or conversion of counts into rates.

## Related Snippets

<!-- catalog:related:start -->
- [Overlay Aligned Time Series at the Finest Step](overlay-aligned-time-series-at-the-finest-step.md)
- [Measure Time in a State Within a Half-Open Window](measure-time-in-a-state-within-a-half-open-window.md)
- [Count Values in Fixed Upper-Bound Bins](../observability-operations/count-values-in-fixed-upper-bound-bins.md)
<!-- catalog:related:end -->
