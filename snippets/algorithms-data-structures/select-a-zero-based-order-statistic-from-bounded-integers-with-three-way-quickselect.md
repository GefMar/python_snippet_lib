---
title: "Select a Zero-Based Order Statistic from Bounded Integers with Three-Way Quickselect"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - select-the-stable-k-smallest-bounded-records-with-a-max-heap.md
  - ../machine-learning-statistics/compute-an-exact-integer-median-and-unscaled-median-absolute-deviation.md
  - ../machine-learning-statistics/select-the-lower-weighted-median-of-bounded-integer-observations.md
---

# Select a Zero-Based Order Statistic from Bounded Integers with Three-Way Quickselect

## Idea and Problem

Return the value at one zero-based rank in sorted order without sorting the complete input or changing the caller's tuple.

Quickselect partitions a working copy around one pivot and continues only in
the region containing the requested rank. A three-way partition groups values
smaller than, equal to, and greater than the pivot. All pivot duplicates then
occupy one closed rank interval, so a rank inside that interval can return
immediately instead of repeatedly partitioning equal values.

## When to Use

Use this function when one order statistic is needed from a bounded snapshot
of integers and record identity or stable ordering does not matter. Typical
examples include selecting a raw percentile rank, a threshold candidate, or a
median value before a separate domain-specific calculation.

Use `sorted(values)[rank]` when predictable `O(n log n)` work and simplicity
matter more than avoiding a full sort. Use a heap for a small sorted prefix,
and use a dedicated weighted-selection method when observations carry weights.

## Implementation

```python
_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_VALUE_COUNT = 65_536


def select_zero_based_order_statistic(
    values: tuple[int, ...],
    rank: int,
) -> int:
    """Return the value at rank in ordinary ascending sorted order."""
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    if not 1 <= len(values) <= _MAX_VALUE_COUNT:
        raise ValueError("value count is outside the supported range")
    if type(rank) is not int:
        raise TypeError("rank must be an exact non-boolean integer")
    if not 0 <= rank < len(values):
        raise ValueError("rank is outside the input")

    for index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"values[{index}] must be an exact non-boolean integer")
        if not _MIN_INT64 <= value <= _MAX_INT64:
            raise ValueError(f"values[{index}] is outside the signed 64-bit range")

    working = list(values)
    start = 0
    stop = len(working)

    while True:
        pivot = working[start + (stop - start) // 2]
        less_stop = start
        current = start
        greater_start = stop

        while current < greater_start:
            if working[current] < pivot:
                working[less_stop], working[current] = (
                    working[current],
                    working[less_stop],
                )
                less_stop += 1
                current += 1
            elif working[current] > pivot:
                greater_start -= 1
                working[current], working[greater_start] = (
                    working[greater_start],
                    working[current],
                )
            else:
                current += 1

        if rank < less_stop:
            stop = less_stop
        elif rank >= greater_start:
            start = greater_start
        else:
            return pivot
```

## Example

```python
def verify_against_full_sort(values: tuple[int, ...]) -> int:
    snapshot = values
    checked_ranks = 0
    expected = sorted(values)
    for rank, expected_value in enumerate(expected):
        assert select_zero_based_order_statistic(values, rank) == expected_value
        assert values == snapshot
        checked_ranks += 1
    return checked_ranks


def exercise_small_sequences() -> int:
    from itertools import product

    checked_ranks = 0
    for length in range(1, 8):
        for values in product((-2, 0, 3), repeat=length):
            checked_ranks += verify_against_full_sort(values)
    return checked_ranks


checked_ranks = exercise_small_sequences()

all_equal = (7,) * _MAX_VALUE_COUNT
duplicate_heavy = (5, -1, 5, 2, -1, 5, 2)
extrema = (_MAX_INT64, 0, _MIN_INT64)

rejected = 0
invalid_calls = (
    lambda: select_zero_based_order_statistic((), 0),
    lambda: select_zero_based_order_statistic((1, True), 0),
    lambda: select_zero_based_order_statistic((0,), True),
    lambda: select_zero_based_order_statistic((0,), -1),
    lambda: select_zero_based_order_statistic((0,), 1),
    lambda: select_zero_based_order_statistic((0,) * (_MAX_VALUE_COUNT + 1), 0),
)
for invalid_call in invalid_calls:
    try:
        invalid_call()
    except (TypeError, ValueError):
        rejected += 1

assert (
    checked_ranks,
    select_zero_based_order_statistic(all_equal, _MAX_VALUE_COUNT - 1),
    tuple(select_zero_based_order_statistic(duplicate_heavy, rank) for rank in (0, 3, 6)),
    tuple(select_zero_based_order_statistic(extrema, rank) for rank in range(3)),
    rejected,
) == (
    21_324,
    7,
    (-1, 2, 5),
    (_MIN_INT64, 0, _MAX_INT64),
    6,
)
```

## Trade-offs and Limitations

Validation and copying take `O(n)` time. Each partition pass is linear in its
active region and uses constant state beyond the working copy. The copy uses
`O(n)` memory and preserves the caller's tuple.

The middle-position pivot is deterministic; it is not a randomized or
median-of-medians guarantee. Some input arrangements can repeatedly produce
highly unbalanced partitions, making total running time `O(n**2)`. This page
therefore makes no unconditional expected-linear-time claim.

The function accepts 1-65,536 exact signed 64-bit non-Boolean integers and one
valid exact rank. It returns a value, not a particular equal record. It does
not preserve stable identities, select multiple ranks in one pass, accept a
key function, mutate the input, or protect against adversarial worst cases.

## Related Snippets

<!-- catalog:related:start -->
- [Select the Stable K Smallest Bounded Records with a Max-Heap](select-the-stable-k-smallest-bounded-records-with-a-max-heap.md)
- [Compute an Exact Integer Median and Unscaled Median Absolute Deviation](../machine-learning-statistics/compute-an-exact-integer-median-and-unscaled-median-absolute-deviation.md)
- [Select the Lower Weighted Median of Bounded Integer Observations](../machine-learning-statistics/select-the-lower-weighted-median-of-bounded-integer-observations.md)
<!-- catalog:related:end -->
