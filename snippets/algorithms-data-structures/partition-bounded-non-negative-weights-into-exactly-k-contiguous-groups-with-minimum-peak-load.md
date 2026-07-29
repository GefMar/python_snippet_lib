---
title: "Partition Bounded Non-Negative Weights into Exact-K Contiguous Groups by Minimum Peak Load"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../data-processing/batch-items-by-estimated-byte-size.md
  - apportion-a-non-negative-integer-total-without-rounding-drift.md
  - report-exact-capacity-deficits-for-bounded-resource-profiles.md
---

# Partition Bounded Non-Negative Weights into Exact-K Contiguous Groups by Minimum Peak Load

## Idea and Problem

Partition bounded non-negative weights into exactly a requested number of contiguous groups while minimizing the greatest group sum.

A candidate peak is feasible when a greedy left-to-right scan needs no more
than the requested number of groups. That monotone decision supports binary
search. Once the minimum peak is known, another forward scan packs each early
group as far right as possible while leaving one item for every later group,
which fixes equal-cost plans deterministically.

## When to Use

Use this algorithm when item order must remain fixed, every group must be
non-empty, and the largest aggregate load determines the quality of a
partition. Examples include contiguous file shards, ordered work ranges, and
sequential storage segments whose weights are exact non-negative units.

Use a fixed-limit batching algorithm when the peak is already prescribed and
the number of groups may vary. Use a scheduler or general partitioning solver
when items may be reordered, groups have different capacities, or additional
affinity and placement constraints matter.

## Implementation

```python
from dataclasses import dataclass

_MAX_WEIGHT_COUNT = 100_000
_MAX_WEIGHT = (1 << 63) - 1


@dataclass(frozen=True, slots=True)
class GroupSpan:
    start: int
    stop: int
    total: int


@dataclass(frozen=True, slots=True)
class ContiguousPartition:
    minimum_peak: int
    groups: tuple[GroupSpan, ...]


def _fits_peak(
    weights: tuple[int, ...],
    *,
    group_count: int,
    peak: int,
) -> bool:
    groups_used = 1
    current_total = 0
    for weight in weights:
        if current_total + weight <= peak:
            current_total += weight
            continue

        groups_used += 1
        if groups_used > group_count:
            return False
        current_total = weight
    return True


def minimum_peak_contiguous_partition(
    weights: tuple[int, ...],
    *,
    group_count: int,
) -> ContiguousPartition:
    """Return the exact-k minimum-peak partition with latest-stop ties."""
    if type(weights) is not tuple:
        raise TypeError("weights must be an exact tuple")
    if not 1 <= len(weights) <= _MAX_WEIGHT_COUNT:
        raise ValueError("weight count is outside the supported range")
    if type(group_count) is not int:
        raise TypeError("group_count must be an exact integer")
    if not 1 <= group_count <= len(weights):
        raise ValueError("group_count must be between one and the weight count")

    for index, weight in enumerate(weights):
        if type(weight) is not int:
            raise TypeError(f"weights[{index}] must be an exact integer")
        if not 0 <= weight <= _MAX_WEIGHT:
            raise ValueError(f"weights[{index}] is outside the supported range")

    total_weight = sum(weights)
    if group_count == 1:
        minimum_peak = total_weight
    elif total_weight == 0:
        minimum_peak = 0
    else:
        lower = max(weights)
        upper = total_weight
        while lower < upper:
            candidate = (lower + upper) // 2
            if _fits_peak(
                weights,
                group_count=group_count,
                peak=candidate,
            ):
                upper = candidate
            else:
                lower = candidate + 1
        minimum_peak = lower

    groups: list[GroupSpan] = []
    assigned_total = 0
    start = 0
    for group_index in range(group_count - 1):
        remaining_groups = group_count - group_index - 1
        latest_stop = len(weights) - remaining_groups
        stop = start
        group_total = 0
        while stop < latest_stop and group_total + weights[stop] <= minimum_peak:
            group_total += weights[stop]
            stop += 1

        groups.append(GroupSpan(start, stop, group_total))
        assigned_total += group_total
        start = stop

    groups.append(
        GroupSpan(
            start=start,
            stop=len(weights),
            total=total_weight - assigned_total,
        )
    )
    return ContiguousPartition(minimum_peak, tuple(groups))
```

## Example

```python
def brute_force_partition(
    weights: tuple[int, ...],
    group_count: int,
) -> ContiguousPartition:
    best_key: tuple[int, tuple[int, ...]] | None = None
    best_result: ContiguousPartition | None = None

    for cut_mask in range(1 << (len(weights) - 1)):
        if cut_mask.bit_count() != group_count - 1:
            continue
        cuts = tuple(index + 1 for index in range(len(weights) - 1) if cut_mask & (1 << index))
        boundaries = (0, *cuts, len(weights))
        groups = tuple(
            GroupSpan(
                boundaries[index],
                boundaries[index + 1],
                sum(weights[boundaries[index] : boundaries[index + 1]]),
            )
            for index in range(group_count)
        )
        peak = max(group.total for group in groups)
        key = peak, tuple(-stop for stop in cuts)
        if best_key is None or key < best_key:
            best_key = key
            best_result = ContiguousPartition(peak, groups)

    if best_result is None:
        raise AssertionError("the brute-force search found no partition")
    return best_result


checked = 0
for length in range(1, 7):
    for encoded in range(3**length):
        weights = tuple((encoded // (3**index)) % 3 for index in range(length))
        for group_count in range(1, length + 1):
            assert minimum_peak_contiguous_partition(
                weights,
                group_count=group_count,
            ) == brute_force_partition(weights, group_count)
            checked += 1

ordinary = minimum_peak_contiguous_partition(
    (7, 2, 5, 10, 8),
    group_count=2,
)
latest_tie = minimum_peak_contiguous_partition(
    (1, 1, 1),
    group_count=2,
)
large_aggregate = minimum_peak_contiguous_partition(
    (_MAX_WEIGHT, _MAX_WEIGHT, _MAX_WEIGHT),
    group_count=2,
)
maximum_count = minimum_peak_contiguous_partition(
    (0,) * _MAX_WEIGHT_COUNT,
    group_count=3,
)

try:
    minimum_peak_contiguous_partition((1, True), group_count=1)
except TypeError:
    boolean_rejected = True
else:
    boolean_rejected = False

try:
    minimum_peak_contiguous_partition((1, -1), group_count=1)
except ValueError:
    negative_rejected = True
else:
    negative_rejected = False

assert (
    checked,
    ordinary,
    latest_tie,
    large_aggregate,
    maximum_count,
    boolean_rejected,
    negative_rejected,
) == (
    6_015,
    ContiguousPartition(
        minimum_peak=18,
        groups=(
            GroupSpan(0, 3, 14),
            GroupSpan(3, 5, 18),
        ),
    ),
    ContiguousPartition(
        minimum_peak=2,
        groups=(
            GroupSpan(0, 2, 2),
            GroupSpan(2, 3, 1),
        ),
    ),
    ContiguousPartition(
        minimum_peak=2 * _MAX_WEIGHT,
        groups=(
            GroupSpan(0, 2, 2 * _MAX_WEIGHT),
            GroupSpan(2, 3, _MAX_WEIGHT),
        ),
    ),
    ContiguousPartition(
        minimum_peak=0,
        groups=(
            GroupSpan(0, 99_998, 0),
            GroupSpan(99_998, 99_999, 0),
            GroupSpan(99_999, 100_000, 0),
        ),
    ),
    True,
    True,
)
```

## Trade-offs and Limitations

For `n` weights with total `S`, validation, each feasibility pass, and final
reconstruction are linear. Binary search therefore takes
`O(n log(S + 2))` time. The scan uses `O(1)` auxiliary state and the immutable
result uses `O(k)` memory. Python keeps aggregate sums exact even when they
exceed the signed 64-bit input range.

The forward tie rule is intentionally biased toward later stop indexes. At a
feasible peak, moving an early stop right removes non-negative weight from the
remaining suffix; any resulting shortfall in group count can be recovered by
splitting later non-empty groups. This proves that greedily choosing each
latest stop preserves feasibility and yields the lexicographically largest
stop tuple.

Weights must be non-negative because both the greedy feasibility test and its
monotonicity rely on that property. The algorithm does not reorder items,
allow empty groups, optimize variance or total deviation, enforce unequal
capacities, or enumerate every optimal partition.

## Related Snippets

<!-- catalog:related:start -->
- [Batch Items by Estimated Byte Size](../data-processing/batch-items-by-estimated-byte-size.md)
- [Apportion a Non-Negative Integer Total Without Rounding Drift](apportion-a-non-negative-integer-total-without-rounding-drift.md)
- [Report Exact Capacity Deficits for Bounded Resource Profiles](report-exact-capacity-deficits-for-bounded-resource-profiles.md)
<!-- catalog:related:end -->
