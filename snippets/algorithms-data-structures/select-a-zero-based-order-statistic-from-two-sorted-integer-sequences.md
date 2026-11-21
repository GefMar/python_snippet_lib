---
title: "Select a Zero-Based Order Statistic from Two Sorted Integer Sequences"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
tested_python:
  - "3.14"
dependencies: []
related:
  - select-a-zero-based-order-statistic-from-bounded-integers-with-three-way-quickselect.md
  - merge-bounded-sorted-integer-runs-with-observable-source-order-ties.md
---

# Select a Zero-Based Order Statistic from Two Sorted Integer Sequences

## Idea and Problem

Find one value by its rank in the combined sorted order without materializing the merge.

The first `rank + 1` values must consist of a prefix of each input. Binary
search the prefix length in the shorter sequence. The other prefix length
follows from the desired total. At a valid partition, neither prefix ends
above the beginning of the other suffix. The greater of the two prefix
endpoints is the requested value.

## When to Use

Use this to select an order statistic from two stable, random-access integer
sequences that are already in nondecreasing order. Each occurrence counts
toward rank, including duplicates. The function selects a value, so ties do
not require deciding which original record supplied it.

## Implementation

```python
from collections.abc import Sequence


def order_statistic(left: Sequence[int], right: Sequence[int], rank: int) -> int:
    """Return the value at zero-based rank in the combined sorted order."""
    if type(rank) is not int:
        raise TypeError("rank must be an integer")
    if not 0 <= rank < len(left) + len(right):
        raise ValueError("rank lies outside the combined input")

    if len(left) > len(right):
        left, right = right, left
    left_size, right_size = len(left), len(right)
    prefix_size = rank + 1
    low = max(0, prefix_size - right_size)
    high = min(left_size, prefix_size)

    while low <= high:
        take_left = (low + high) // 2
        take_right = prefix_size - take_left
        if take_left and take_right < right_size and left[take_left - 1] > right[take_right]:
            high = take_left - 1
        elif take_right and take_left < left_size and right[take_right - 1] > left[take_left]:
            low = take_left + 1
        elif not take_left:
            return right[take_right - 1]
        elif not take_right:
            return left[take_left - 1]
        else:
            return max(left[take_left - 1], right[take_right - 1])

    raise ValueError("no rank partition found")
```

## Example

```python
morning = (-3, 2, 2, 9)
afternoon = (-6, 2, 4, 11, 13)

assert (
    order_statistic(morning, afternoon, rank=0),
    order_statistic(morning, afternoon, rank=4),
    order_statistic(afternoon, morning, rank=5),
    order_statistic(morning, afternoon, rank=8),
    order_statistic((), afternoon, rank=2),
) == (-6, 2, 4, 13, 4)
```

## Trade-offs and Limitations

With lists or tuples, the function uses constant auxiliary storage and
logarithmically many comparisons in the shorter input length. Unsorted inputs
can invalidate the partition decision. Verify sorting once upstream if the
caller cannot guarantee it.

Use a merge when many ranks or record origins are needed. For an even-sized
sample whose median is the mean of two middle values, request both ranks and
apply that averaging rule separately.

## Related Snippets

<!-- catalog:related:start -->
- [Select a Zero-Based Order Statistic from Bounded Integers with Three-Way Quickselect](select-a-zero-based-order-statistic-from-bounded-integers-with-three-way-quickselect.md)
- [Merge Bounded Sorted Integer Runs with Observable Source-Order Ties](merge-bounded-sorted-integer-runs-with-observable-source-order-ties.md)
<!-- catalog:related:end -->
