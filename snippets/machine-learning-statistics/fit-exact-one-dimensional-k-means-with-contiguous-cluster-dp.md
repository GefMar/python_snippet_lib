---
title: "Fit Exact One-Dimensional k-Means with Contiguous-Cluster DP"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - fit-an-exact-unweighted-isotonic-regression-with-pool-adjacent-violators.md
  - compute-an-exact-integer-median-and-unscaled-median-absolute-deviation.md
  - ../algorithms-data-structures/partition-bounded-non-negative-weights-into-exactly-k-contiguous-groups-with-minimum-peak-load.md
---

# Fit Exact One-Dimensional k-Means with Contiguous-Cluster DP

## Idea and Problem

Partition one sorted integer sequence into a declared number of non-empty contiguous clusters with the minimum exact sum of squared deviations from cluster means.

Prefix sums and squared sums give the cost of any half-open span in constant
time. Dynamic programming then records the minimum cost of dividing every
suffix into a fixed number of clusters. Reconstruction considers each next
stop from left to right, so equal-cost solutions return the lexicographically
smallest tuple of cluster stop indexes.

## When to Use

Use this algorithm when one-dimensional observations are already
nondecreasing, every observation has equal weight, and exact reproducible
cluster boundaries matter. Each returned mean, span cost, and total cost is a
`Fraction`, so no floating-point comparison influences the selected partition.

Use a specialized implementation for unsorted or floating-point observations,
sample weights, several dimensions, online updates, or substantially larger
inputs. This contract solves only the declared contiguous partition problem;
it is not an implementation of Lloyd iteration or a general k-means API.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_OBSERVATIONS = 512
_MAX_CLUSTERS = 32


@dataclass(frozen=True, slots=True)
class ExactKMeansCluster:
    start: int
    stop: int
    count: int
    total: int
    mean: Fraction
    squared_error: Fraction


@dataclass(frozen=True, slots=True)
class ExactKMeansFit:
    clusters: tuple[ExactKMeansCluster, ...]
    total_squared_error: Fraction


def fit_exact_contiguous_k_means(
    values: tuple[int, ...],
    *,
    cluster_count: int,
) -> ExactKMeansFit:
    """Return the exact minimum-SSE contiguous partition with earliest stops."""
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    if not 1 <= len(values) <= _MAX_OBSERVATIONS:
        raise ValueError("observation count is outside the supported range")
    if type(cluster_count) is not int:
        raise TypeError("cluster_count must be an exact integer")
    if not 1 <= cluster_count <= min(_MAX_CLUSTERS, len(values)):
        raise ValueError("cluster_count is outside the supported range")

    previous: int | None = None
    for index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"values[{index}] must be an exact integer")
        if not _MIN_INT64 <= value <= _MAX_INT64:
            raise ValueError(f"values[{index}] is outside the signed 64-bit range")
        if previous is not None and value < previous:
            raise ValueError("values must be nondecreasing")
        previous = value

    prefix_totals = [0]
    prefix_squared_totals = [0]
    for value in values:
        prefix_totals.append(prefix_totals[-1] + value)
        prefix_squared_totals.append(prefix_squared_totals[-1] + value * value)

    def span_statistics(start: int, stop: int) -> tuple[int, Fraction]:
        count = stop - start
        total = prefix_totals[stop] - prefix_totals[start]
        squared_total = prefix_squared_totals[stop] - prefix_squared_totals[start]
        squared_error = Fraction(count * squared_total - total * total, count)
        return total, squared_error

    observation_count = len(values)
    span_costs: list[list[Fraction | None]] = [
        [None] * (observation_count + 1) for _ in range(observation_count)
    ]
    for start in range(observation_count):
        for stop in range(start + 1, observation_count + 1):
            span_costs[start][stop] = span_statistics(start, stop)[1]

    suffix_costs: list[list[Fraction | None]] = [
        [None] * (observation_count + 1) for _ in range(cluster_count + 1)
    ]
    for start in range(observation_count):
        suffix_costs[1][start] = span_costs[start][observation_count]

    for groups in range(2, cluster_count + 1):
        for start in range(observation_count - groups + 1):
            best: Fraction | None = None
            latest_stop = observation_count - groups + 1
            for stop in range(start + 1, latest_stop + 1):
                head_cost = span_costs[start][stop]
                tail_cost = suffix_costs[groups - 1][stop]
                if head_cost is None or tail_cost is None:
                    raise AssertionError("a feasible DP state must have exact costs")
                candidate = head_cost + tail_cost
                if best is None or candidate < best:
                    best = candidate
            suffix_costs[groups][start] = best

    clusters: list[ExactKMeansCluster] = []
    start = 0
    groups = cluster_count
    while groups:
        optimum = suffix_costs[groups][start]
        if optimum is None:
            raise AssertionError("the requested partition must be feasible")
        if groups == 1:
            stop = observation_count
        else:
            latest_stop = observation_count - groups + 1
            for candidate_stop in range(start + 1, latest_stop + 1):
                head_cost = span_costs[start][candidate_stop]
                tail_cost = suffix_costs[groups - 1][candidate_stop]
                if (
                    head_cost is not None
                    and tail_cost is not None
                    and head_cost + tail_cost == optimum
                ):
                    stop = candidate_stop
                    break
            else:
                raise AssertionError("an optimal DP transition must exist")

        total, squared_error = span_statistics(start, stop)
        count = stop - start
        clusters.append(
            ExactKMeansCluster(
                start=start,
                stop=stop,
                count=count,
                total=total,
                mean=Fraction(total, count),
                squared_error=squared_error,
            )
        )
        start = stop
        groups -= 1

    total_squared_error = suffix_costs[cluster_count][0]
    if total_squared_error is None:
        raise AssertionError("the complete fit must have an exact cost")
    return ExactKMeansFit(tuple(clusters), total_squared_error)
```

## Example

```python


fit = fit_exact_contiguous_k_means((0, 1, 9, 10), cluster_count=2)
equal_values = fit_exact_contiguous_k_means((5, 5, 5, 5), cluster_count=3)

assert fit == ExactKMeansFit(
    clusters=(
        ExactKMeansCluster(0, 2, 2, 1, Fraction(1, 2), Fraction(1, 2)),
        ExactKMeansCluster(2, 4, 2, 19, Fraction(19, 2), Fraction(1, 2)),
    ),
    total_squared_error=Fraction(1),
)
assert (
    tuple(cluster.stop for cluster in equal_values.clusters),
    equal_values.total_squared_error,
) == ((1, 2, 4), Fraction(0))
```

## Trade-offs and Limitations

Precomputing every span cost takes `O(n**2)` exact arithmetic operations.
The dynamic program takes `O(k*n**2)` exact additions and comparisons, and
the span and suffix tables use `O(n**2 + k*n)` memory. Integer, `Fraction`,
and greatest-common-divisor costs grow with operand bit length rather than
remaining constant.

All clusters are non-empty half-open spans that cover the input exactly. The
forward reconstruction chooses the earliest feasible stop at each optimal
suffix, which yields the lexicographically smallest complete stop-index tuple.
Requiring exactly `k` clusters means equal adjacent values may be split across
several zero-cost clusters.

Inputs are exact signed-64-bit integers in nondecreasing order. The function
rejects Booleans, floats, unsorted sequences, weights, empty inputs, and more
than 512 observations or 32 clusters. It does not sort data, fit
multidimensional points, update a fit online, or approximate costs.

## Related Snippets

<!-- catalog:related:start -->
- [Fit an Exact Unweighted Isotonic Regression with Pool-Adjacent Violators](fit-an-exact-unweighted-isotonic-regression-with-pool-adjacent-violators.md)
- [Compute an Exact Integer Median and Unscaled Median Absolute Deviation](compute-an-exact-integer-median-and-unscaled-median-absolute-deviation.md)
- [Partition Bounded Non-Negative Weights into Exact-K Contiguous Groups by Minimum Peak Load](../algorithms-data-structures/partition-bounded-non-negative-weights-into-exactly-k-contiguous-groups-with-minimum-peak-load.md)
<!-- catalog:related:end -->
