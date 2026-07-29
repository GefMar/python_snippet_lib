---
title: "Compute a Validated Delta Between Cumulative Histogram Snapshots"
snippet_type: algorithm
use_cases:
  - data-transformation
  - observability
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - count-values-in-fixed-upper-bound-bins.md
  - locate-an-exact-quantile-rank-in-bounded-fixed-bucket-counts.md
  - measure-cache-hit-ratios-from-monotonic-counter-snapshots.md
---

# Compute a Validated Delta Between Cumulative Histogram Snapshots

## Idea and Problem

Recover exact interval bucket counts from two coherent snapshots of the same cumulative fixed-bound histogram.

Subtracting corresponding cumulative counters gives cumulative interval counts.
Adjacent differences then recover each finite bucket, while the difference
between the total and the last cumulative counter recovers the overflow bucket.
Validating those relationships prevents independently captured or partially
updated counters from producing negative bucket increments.

## When to Use

Use this calculation when both snapshots come from one histogram schema, every
counter is cumulative and monotonic, and collection provides coherent frozen
snapshots. The result is suitable for interval aggregation or later rate and
quantile calculations that keep their own semantics explicit.

Treat any visible counter decrease as a reset of the whole snapshot rather than
mixing pre-reset and post-reset values. Use an epoch or reset identifier when a
collector must distinguish a restart from counter wrap or replacement.

## Implementation

```python
from dataclasses import dataclass
from itertools import pairwise

_MAX_INT64 = (1 << 63) - 1
_MAX_BUCKETS = 256


@dataclass(frozen=True, slots=True)
class CumulativeHistogramSnapshot:
    timestamp: int
    upper_bounds: tuple[int, ...]
    cumulative_counts: tuple[int, ...]
    total_count: int


@dataclass(frozen=True, slots=True)
class CumulativeHistogramReset:
    earlier_timestamp: int
    later_timestamp: int


@dataclass(frozen=True, slots=True)
class CumulativeHistogramDelta:
    upper_bounds: tuple[int, ...]
    bucket_increments: tuple[int, ...]
    total_increment: int
    elapsed_time: int


def _validate_snapshot(
    value: object,
    *,
    name: str,
) -> CumulativeHistogramSnapshot:
    if type(value) is not CumulativeHistogramSnapshot:
        raise TypeError(f"{name} must be an exact CumulativeHistogramSnapshot")
    if type(value.timestamp) is not int:
        raise TypeError(f"{name}.timestamp must be an exact integer")
    if not 0 <= value.timestamp <= _MAX_INT64:
        raise ValueError(f"{name}.timestamp is outside the supported range")

    if type(value.upper_bounds) is not tuple:
        raise TypeError(f"{name}.upper_bounds must be an exact tuple")
    if not 1 <= len(value.upper_bounds) <= _MAX_BUCKETS:
        raise ValueError(f"{name}.upper_bounds has an unsupported length")
    previous_bound: int | None = None
    for index, bound in enumerate(value.upper_bounds):
        if type(bound) is not int:
            raise TypeError(f"{name}.upper_bounds[{index}] must be an exact integer")
        if not -(1 << 63) <= bound <= _MAX_INT64:
            raise ValueError(f"{name}.upper_bounds[{index}] is outside signed 64-bit range")
        if previous_bound is not None and bound <= previous_bound:
            raise ValueError(f"{name}.upper_bounds must be strictly increasing")
        previous_bound = bound

    if type(value.cumulative_counts) is not tuple:
        raise TypeError(f"{name}.cumulative_counts must be an exact tuple")
    if len(value.cumulative_counts) != len(value.upper_bounds):
        raise ValueError(f"{name}.cumulative_counts must align with upper_bounds")
    previous_count = 0
    for index, count in enumerate(value.cumulative_counts):
        if type(count) is not int:
            raise TypeError(f"{name}.cumulative_counts[{index}] must be an exact integer")
        if not 0 <= count <= _MAX_INT64:
            raise ValueError(f"{name}.cumulative_counts[{index}] is outside the supported range")
        if count < previous_count:
            raise ValueError(f"{name}.cumulative_counts must be nondecreasing")
        previous_count = count

    if type(value.total_count) is not int:
        raise TypeError(f"{name}.total_count must be an exact integer")
    if not 0 <= value.total_count <= _MAX_INT64:
        raise ValueError(f"{name}.total_count is outside the supported range")
    if value.total_count < value.cumulative_counts[-1]:
        raise ValueError(f"{name}.total_count is below its last cumulative count")
    return value


def cumulative_histogram_delta(
    earlier: CumulativeHistogramSnapshot,
    later: CumulativeHistogramSnapshot,
) -> CumulativeHistogramDelta | CumulativeHistogramReset:
    """Return a coherent interval delta or one whole-snapshot reset outcome."""
    first = _validate_snapshot(earlier, name="earlier")
    second = _validate_snapshot(later, name="later")

    if second.timestamp <= first.timestamp:
        raise ValueError("later.timestamp must be greater than earlier.timestamp")
    if second.upper_bounds != first.upper_bounds:
        raise ValueError("histogram upper bounds do not match")

    counter_reset = any(
        later_count < earlier_count
        for earlier_count, later_count in zip(
            first.cumulative_counts,
            second.cumulative_counts,
            strict=True,
        )
    )
    if counter_reset or second.total_count < first.total_count:
        return CumulativeHistogramReset(first.timestamp, second.timestamp)

    cumulative_deltas = tuple(
        later_count - earlier_count
        for earlier_count, later_count in zip(
            first.cumulative_counts,
            second.cumulative_counts,
            strict=True,
        )
    )
    if any(current < previous for previous, current in pairwise(cumulative_deltas)):
        raise ValueError("cumulative counter differences are not nondecreasing")

    total_increment = second.total_count - first.total_count
    if total_increment < cumulative_deltas[-1]:
        raise ValueError("total difference is below the last cumulative difference")

    finite_increments: list[int] = []
    previous_delta = 0
    for cumulative_delta in cumulative_deltas:
        finite_increments.append(cumulative_delta - previous_delta)
        previous_delta = cumulative_delta
    overflow_increment = total_increment - cumulative_deltas[-1]

    return CumulativeHistogramDelta(
        upper_bounds=first.upper_bounds,
        bucket_increments=(*finite_increments, overflow_increment),
        total_increment=total_increment,
        elapsed_time=second.timestamp - first.timestamp,
    )
```

## Example

```python
earlier = CumulativeHistogramSnapshot(100, (10, 20), (3, 7), 10)
later = CumulativeHistogramSnapshot(120, (10, 20), (5, 12), 17)

delta = cumulative_histogram_delta(earlier, later)
reset = cumulative_histogram_delta(
    earlier,
    CumulativeHistogramSnapshot(130, (10, 20), (2, 9), 11),
)

try:
    cumulative_histogram_delta(
        earlier,
        CumulativeHistogramSnapshot(140, (10, 20), (6, 9), 12),
    )
except ValueError:
    incoherent_rejected = True
else:
    incoherent_rejected = False

assert (delta, reset, incoherent_rejected) == (
    CumulativeHistogramDelta((10, 20), (2, 3, 2), 7, 20),
    CumulativeHistogramReset(100, 130),
    True,
)
```

## Trade-offs and Limitations

Validation and subtraction take `O(B)` time for `B` finite upper bounds. The
frozen result uses `O(B)` memory; the calculation also materializes `O(B)`
cumulative differences and finite increments. Python integers keep subtraction
exact within the admitted signed-64-bit counter range.

Finite bucket `i` covers observations at or below its bound and above the
previous bound; the final increment covers observations above the last bound.
A visible decrease in any cumulative counter or the total has reset priority,
so no partial delta is returned even if other differences look coherent.

Two snapshots cannot reveal a reset followed by enough new observations to
make every later counter exceed its earlier value. This function does not
calculate rates, interpolate values inside a bucket, estimate quantiles, merge
schemas, reconstruct reset amounts, capture snapshots, or establish their
atomicity.

## Related Snippets

<!-- catalog:related:start -->
- [Count Values in Fixed Upper-Bound Bins](count-values-in-fixed-upper-bound-bins.md)
- [Locate an Exact Quantile Rank in Bounded Fixed-Bucket Counts](locate-an-exact-quantile-rank-in-bounded-fixed-bucket-counts.md)
- [Measure Cache Hit Ratios from Monotonic Counter Snapshots](measure-cache-hit-ratios-from-monotonic-counter-snapshots.md)
<!-- catalog:related:end -->
