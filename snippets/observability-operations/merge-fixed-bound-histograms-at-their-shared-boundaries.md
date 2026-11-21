---
title: "Merge Fixed-Bound Histograms at Their Shared Boundaries"
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
  - compute-a-validated-delta-between-cumulative-histogram-snapshots.md
---

# Merge Fixed-Bound Histograms at Their Shared Boundaries

## Idea and Problem

Combine two fixed-bound histograms without guessing how observations were distributed inside their original buckets.

At every shared upper bound, both histograms know exactly how many observations
are at or below it. Add those cumulative counts, then subtract consecutive
totals to recover counts in the shared buckets. An upper bound present on only
one side could split a bucket on the other side, so the merged histogram keeps
only the common boundaries.

## When to Use

Use this when combining non-overlapping observation sets collected in the same
units with different integer bucket boundaries. Inputs are per-bucket counts
for `(-infinity, b0]`, `(b0, b1]`, and so on, with a final overflow bucket
`(bn, +infinity)`. A histogram without finite bounds has one all-values bucket.

## Implementation

```python
from dataclasses import dataclass
from itertools import accumulate, pairwise


@dataclass(frozen=True, slots=True)
class FixedHistogram:
    upper_bounds: tuple[int, ...]
    counts: tuple[int, ...]

    def __post_init__(self) -> None:
        if type(self.upper_bounds) is not tuple or type(self.counts) is not tuple:
            raise TypeError("upper_bounds and counts must be tuples")
        if len(self.counts) != len(self.upper_bounds) + 1:
            raise ValueError("counts must include one overflow bucket")
        if any(type(bound) is not int for bound in self.upper_bounds):
            raise TypeError("upper bounds must be integers")
        if any(left >= right for left, right in pairwise(self.upper_bounds)):
            raise ValueError("upper bounds must be strictly increasing")
        if any(type(count) is not int for count in self.counts):
            raise TypeError("counts must be integers")
        if any(count < 0 for count in self.counts):
            raise ValueError("counts must be non-negative")


def _counts_at_bounds(histogram: FixedHistogram) -> dict[int, int]:
    finite_bucket_counts = histogram.counts[:-1]
    return dict(zip(histogram.upper_bounds, accumulate(finite_bucket_counts), strict=True))


def merge_fixed_histograms(left: FixedHistogram, right: FixedHistogram) -> FixedHistogram:
    """Merge per-bucket counts at the boundaries present in both inputs."""
    left_at_bound = _counts_at_bounds(left)
    right_at_bound = _counts_at_bounds(right)
    shared_bounds = tuple(bound for bound in left.upper_bounds if bound in right_at_bound)
    cumulative = [0]
    cumulative.extend(left_at_bound[bound] + right_at_bound[bound] for bound in shared_bounds)
    cumulative.append(sum(left.counts) + sum(right.counts))
    counts = tuple(end - start for start, end in pairwise(cumulative))
    return FixedHistogram(shared_bounds, counts)
```

## Example

```python
# Durations from two separate collections, in milliseconds.
# Left observations: 8, 10, 12, 15, 20, 30.
# Right observations: 5, 10, 20, 25, 25.
left = FixedHistogram((10, 15, 20), (2, 2, 1, 1))
right = FixedHistogram((5, 10, 20), (1, 1, 1, 2))

assert (
    merge_fixed_histograms(left, right),
    merge_fixed_histograms(left, FixedHistogram((7,), (1, 2))),
) == (
    FixedHistogram((10, 20), (4, 4, 3)),
    FixedHistogram((), (9,)),
)
```

## Trade-offs and Limitations

The merged counts are exact, with a possible loss of resolution when boundaries
differ. If no boundaries coincide, the only result is the combined total.
Keep raw observations or agree on a finer common layout before collection
when downstream queries need boundaries discarded by the merge.

The implementation stores cumulative counts for the input boundaries. It cannot
verify that both inputs measure the same quantity, share units, or represent
disjoint observations. It also assumes each input already represents the
intended interval. Convert cumulative counters to per-bucket interval counts
before merging when a collector reports cumulative snapshots.

## Related Snippets

<!-- catalog:related:start -->
- [Count Values in Fixed Upper-Bound Bins](count-values-in-fixed-upper-bound-bins.md)
- [Compute a Validated Delta Between Cumulative Histogram Snapshots](compute-a-validated-delta-between-cumulative-histogram-snapshots.md)
<!-- catalog:related:end -->
