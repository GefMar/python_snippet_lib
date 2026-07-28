---
title: "Select a Maximum-Weight Set of Non-Overlapping Half-Open Integer Intervals"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - select-a-maximum-cardinality-set-of-non-overlapping-half-open-integer-intervals.md
  - coalesce-bounded-half-open-integer-intervals-under-an-explicit-adjacency-policy.md
  - find-a-point-in-disjoint-half-open-intervals.md
---

# Select a Maximum-Weight Set of Non-Overlapping Half-Open Integer Intervals

## Idea and Problem

Choose a compatible interval set with the greatest total weight while resolving equal totals by one global index-tuple rule.

Sort intervals by their stop, start, and declaration index. For every canonical
prefix, dynamic programming compares the best plan that excludes the current
interval with the plan that appends it to the best compatible predecessor.
Keeping each selected original-index tuple makes the global tie policy explicit
instead of relying on incidental loop order.

## When to Use

Use this algorithm for one bounded, in-memory collection of weighted activities
when intervals on the same timeline may touch but must not overlap. It is useful
when maximizing item count is wrong because some compatible selections have
greater total value than larger selections.

Use the earliest-finish greedy algorithm when every interval has equal value
and only cardinality matters. Choose a specialized scheduler when work is
preemptible, setup time matters, several resources are available, or plans must
be updated online.

## Implementation

```python
from bisect import bisect_right
from dataclasses import dataclass

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_INTERVALS = 512


@dataclass(frozen=True, slots=True)
class WeightedHalfOpenInterval:
    start: int
    stop: int
    weight: int


@dataclass(frozen=True, slots=True)
class MaximumWeightIntervalSelection:
    total_weight: int
    indexes: tuple[int, ...]


def select_maximum_weight_intervals(
    intervals: tuple[WeightedHalfOpenInterval, ...],
) -> MaximumWeightIntervalSelection:
    """Return one globally index-tied maximum-weight compatible selection."""
    if type(intervals) is not tuple:
        raise TypeError("intervals must be an exact tuple")
    if len(intervals) > _MAX_INTERVALS:
        raise ValueError("interval count exceeds the supported limit")

    for index, interval in enumerate(intervals):
        if type(interval) is not WeightedHalfOpenInterval:
            raise TypeError(f"intervals[{index}] must be an exact WeightedHalfOpenInterval")
        if type(interval.start) is not int or type(interval.stop) is not int:
            raise TypeError(f"intervals[{index}] endpoints must be exact integers")
        if not _MIN_INT64 <= interval.start <= _MAX_INT64:
            raise ValueError(f"intervals[{index}].start is outside the signed 64-bit range")
        if not _MIN_INT64 <= interval.stop <= _MAX_INT64:
            raise ValueError(f"intervals[{index}].stop is outside the signed 64-bit range")
        if interval.start >= interval.stop:
            raise ValueError(f"intervals[{index}] must have start < stop")
        if type(interval.weight) is not int:
            raise TypeError(f"intervals[{index}].weight must be an exact integer")
        if not 1 <= interval.weight <= _MAX_INT64:
            raise ValueError(f"intervals[{index}].weight is outside the positive int64 range")

    ordered_indexes = sorted(
        range(len(intervals)),
        key=lambda index: (
            intervals[index].stop,
            intervals[index].start,
            index,
        ),
    )
    ordered_stops = tuple(intervals[index].stop for index in ordered_indexes)

    best_weights = [0]
    best_index_tuples: list[tuple[int, ...]] = [()]

    for position, original_index in enumerate(ordered_indexes):
        interval = intervals[original_index]
        predecessor_count = bisect_right(
            ordered_stops,
            interval.start,
            hi=position,
        )
        include_weight = best_weights[predecessor_count] + interval.weight
        include_indexes = (*best_index_tuples[predecessor_count], original_index)
        exclude_weight = best_weights[position]
        exclude_indexes = best_index_tuples[position]

        if include_weight > exclude_weight or (
            include_weight == exclude_weight and include_indexes < exclude_indexes
        ):
            best_weights.append(include_weight)
            best_index_tuples.append(include_indexes)
        else:
            best_weights.append(exclude_weight)
            best_index_tuples.append(exclude_indexes)

    return MaximumWeightIntervalSelection(
        total_weight=best_weights[-1],
        indexes=best_index_tuples[-1],
    )
```

## Example

```python
intervals = (
    WeightedHalfOpenInterval(2, 4, 5),
    WeightedHalfOpenInterval(4, 6, 1),
    WeightedHalfOpenInterval(0, 2, 5),
    WeightedHalfOpenInterval(0, 4, 10),
)

selection = select_maximum_weight_intervals(intervals)
selected_intervals = tuple(intervals[index] for index in selection.indexes)
empty = select_maximum_weight_intervals(())

try:
    select_maximum_weight_intervals((WeightedHalfOpenInterval(0, 1, True),))
except TypeError:
    bool_rejected = True
else:
    bool_rejected = False

assert (selection, selected_intervals, empty, bool_rejected) == (
    MaximumWeightIntervalSelection(total_weight=11, indexes=(2, 0, 1)),
    (
        WeightedHalfOpenInterval(0, 2, 5),
        WeightedHalfOpenInterval(2, 4, 5),
        WeightedHalfOpenInterval(4, 6, 1),
    ),
    MaximumWeightIntervalSelection(total_weight=0, indexes=()),
    True,
)
```

## Trade-offs and Limitations

Validation takes `O(n)` time. Sorting and predecessor searches take
`O(n log n)` time. Numeric dynamic programming is linear after sorting, but
copying, storing, and comparing complete tie tuples makes the honest worst case
`O(n^2)` time and memory. The 512-interval limit bounds that deliberate cost.

The result maximizes an exact Python integer total. Among equal totals it is the
lexicographically smallest complete tuple of original indexes in chronological
selection order. Positive weights ensure that equally weighted predecessor
plans cannot differ only by one being a proper prefix, so appending the same
current index preserves their lexicographic order.

Half-open intervals are compatible when the later start is greater than or
equal to the earlier stop. Duplicate interval values remain distinct inputs,
and declaration indexes are part of the tie policy. The contract excludes
zero-width or closed intervals, zero or negative weights, floats, preemption,
setup times, multiple resources, online updates, and requests for every
optimum.

## Related Snippets

<!-- catalog:related:start -->
- [Select a Maximum-Cardinality Set of Non-Overlapping Half-Open Integer Intervals](select-a-maximum-cardinality-set-of-non-overlapping-half-open-integer-intervals.md)
- [Coalesce Bounded Half-Open Integer Intervals Under an Explicit Adjacency Policy](coalesce-bounded-half-open-integer-intervals-under-an-explicit-adjacency-policy.md)
- [Find a Point in Disjoint Half-Open Intervals](find-a-point-in-disjoint-half-open-intervals.md)
<!-- catalog:related:end -->
