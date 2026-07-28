---
title: "Locate an Exact Quantile Rank in Bounded Fixed-Bucket Counts"
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
  - share-bounded-counters-and-duration-histograms-across-spawned-processes.md
  - ../machine-learning-statistics/measure-drift-between-two-fixed-bin-count-distributions-with-psi.md
---

# Locate an Exact Quantile Rank in Bounded Fixed-Bucket Counts

## Idea and Problem

Locate the fixed histogram bucket containing an exact nearest-rank quantile without pretending that bucket counts retain the underlying observation.

For `q` in `(0, 1]` and `n` observations, the one-based target rank is
`ceil(q * n)`. A cumulative scan selects the first bucket whose count reaches
that rank. The result reports the bucket's interval and rank evidence rather
than inventing a point estimate inside a bucket.

## When to Use

Use this algorithm after collecting a complete fixed-boundary histogram when a
report needs an honest percentile bracket and the original observations are no
longer available. All producers must use the same right-closed buckets:
`(-infinity, b0]`, `(b0, b1]`, and a final `(bn, +infinity)` overflow bucket.

Use the same boundary tuple for every compared snapshot. Retain raw values or a
quantile sketch when a numeric estimate, interpolation, rebucketing, or tighter
error guarantees are required.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction

_MAX_BOUND_COUNT = 256
_MAX_QUANTILE_DENOMINATOR = 1_000_000
_SIGNED_64_MIN = -(1 << 63)
_SIGNED_64_MAX = (1 << 63) - 1


@dataclass(frozen=True, slots=True)
class QuantileRankBucket:
    index: int
    lower_exclusive: int | None
    upper_inclusive: int | None
    bucket_count: int
    cumulative_before: int
    target_rank: int


def _signed_64_integer(value: object, *, name: str) -> int:
    if type(value) is not int:
        raise TypeError(f"{name} must be an exact integer")
    if not _SIGNED_64_MIN <= value <= _SIGNED_64_MAX:
        raise ValueError(f"{name} is outside the signed 64-bit range")
    return value


def locate_quantile_rank(
    upper_bounds: tuple[int, ...],
    bucket_counts: tuple[int, ...],
    quantile: Fraction,
) -> QuantileRankBucket:
    """Return the fixed bucket containing the exact nearest-rank quantile."""
    if type(upper_bounds) is not tuple:
        raise TypeError("upper_bounds must be an exact tuple")
    if not 1 <= len(upper_bounds) <= _MAX_BOUND_COUNT:
        raise ValueError("upper bound count is outside the supported range")

    previous: int | None = None
    for index, bound in enumerate(upper_bounds):
        normalized = _signed_64_integer(bound, name=f"upper_bounds[{index}]")
        if previous is not None and normalized <= previous:
            raise ValueError("upper bounds must be strictly increasing")
        previous = normalized

    if type(bucket_counts) is not tuple:
        raise TypeError("bucket_counts must be an exact tuple")
    if len(bucket_counts) != len(upper_bounds) + 1:
        raise ValueError("bucket_counts must include one count per bucket")

    total = 0
    for index, count in enumerate(bucket_counts):
        normalized = _signed_64_integer(count, name=f"bucket_counts[{index}]")
        if normalized < 0:
            raise ValueError("bucket counts must be non-negative")
        total += normalized
        if total > _SIGNED_64_MAX:
            raise ValueError("aggregate bucket count exceeds the signed 64-bit range")
    if total == 0:
        raise ValueError("aggregate bucket count must be positive")

    if type(quantile) is not Fraction:
        raise TypeError("quantile must be an exact Fraction")
    if not Fraction(0, 1) < quantile <= Fraction(1, 1):
        raise ValueError("quantile must be in the interval (0, 1]")
    if quantile.denominator > _MAX_QUANTILE_DENOMINATOR:
        raise ValueError("quantile denominator exceeds the supported limit")

    target_rank = (total * quantile.numerator + quantile.denominator - 1) // quantile.denominator
    cumulative = 0
    for index, count in enumerate(bucket_counts):
        if cumulative + count >= target_rank:
            return QuantileRankBucket(
                index=index,
                lower_exclusive=None if index == 0 else upper_bounds[index - 1],
                upper_inclusive=(upper_bounds[index] if index < len(upper_bounds) else None),
                bucket_count=count,
                cumulative_before=cumulative,
                target_rank=target_rank,
            )
        cumulative += count

    raise AssertionError("validated counts must contain the target rank")
```

## Example

```python
bounds = (0, 10)
counts = (2, 3, 1)

median_bucket = locate_quantile_rank(bounds, counts, Fraction(1, 2))
maximum_bucket = locate_quantile_rank(bounds, counts, Fraction(1, 1))

try:
    locate_quantile_rank(bounds, (0, 0, 0), Fraction(1, 2))
except ValueError:
    empty_histogram_rejected = True
else:
    empty_histogram_rejected = False

assert (median_bucket, maximum_bucket, empty_histogram_rejected) == (
    QuantileRankBucket(
        index=1,
        lower_exclusive=0,
        upper_inclusive=10,
        bucket_count=3,
        cumulative_before=2,
        target_rank=3,
    ),
    QuantileRankBucket(
        index=2,
        lower_exclusive=10,
        upper_inclusive=None,
        bucket_count=1,
        cumulative_before=5,
        target_rank=6,
    ),
    True,
)
```

## Trade-offs and Limitations

Validation and selection take `O(B)` time for `B` buckets and `O(1)` additional
space. The function accepts per-bucket counts, not cumulative counts, and it
does not check whether a separately collected snapshot was coherent.

The output identifies only a rank-containing interval. It cannot recover an
observation, interpolate within a bucket, merge incompatible boundary layouts,
or provide the error properties of a quantile sketch. A result with no lower
or upper bound is unbounded on that side and must not be presented as a finite
numeric quantile.

## Related Snippets

<!-- catalog:related:start -->
- [Count Values in Fixed Upper-Bound Bins](count-values-in-fixed-upper-bound-bins.md)
- [Share Bounded Counters and Duration Histograms Across Spawned Processes](share-bounded-counters-and-duration-histograms-across-spawned-processes.md)
- [Measure Drift Between Two Fixed-Bin Count Distributions with PSI](../machine-learning-statistics/measure-drift-between-two-fixed-bin-count-distributions-with-psi.md)
<!-- catalog:related:end -->
