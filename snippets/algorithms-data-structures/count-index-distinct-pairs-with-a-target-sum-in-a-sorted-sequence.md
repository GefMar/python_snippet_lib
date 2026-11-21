---
title: "Count Index-Distinct Pairs with a Target Sum in a Sorted Sequence"
snippet_type: algorithm
use_cases:
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - enumerate-the-k-smallest-pair-sums-from-two-sorted-integer-sequences.md
  - count-equal-value-index-pairs-in-offline-ranges-with-mos-algorithm.md
---

# Count Index-Distinct Pairs with a Target Sum in a Sorted Sequence

## Idea and Problem

Count all unordered index pairs whose values sum to a target in one sorted list.

Move a pointer at each end. When a pair matches, multiply the counts in the two matching value runs. If the end values are equal, all remaining pairs match and can be counted together.

## When to Use

Use this for a nondecreasing integer sequence when repeated values represent different positions. Return the number of pairs with the first index strictly smaller than the second, without enumerating the pairs.

## Implementation

```python
from collections.abc import Sequence
from itertools import pairwise
from math import comb


def count_pairs_with_sum(values: Sequence[int], target: int) -> int:
    if any(a > b for a, b in pairwise(values)):
        raise ValueError("values must be sorted in nondecreasing order")

    left, right = 0, len(values) - 1
    count = 0
    while left < right:
        low, high = values[left], values[right]
        total = low + high
        if total < target:
            left += 1
        elif total > target:
            right -= 1
        elif low == high:
            return count + comb(right - left + 1, 2)
        else:
            old_left, old_right = left, right
            while values[left] == low:
                left += 1
            while values[right] == high:
                right -= 1
            count += (left - old_left) * (old_right - right)
    return count
```

## Example

```python
assert (
    count_pairs_with_sum((-3, -3, 0, 1, 1, 4, 4, 4), 1),
    count_pairs_with_sum((3, 3, 3, 3), 6),
    count_pairs_with_sum((-2, 0, 2), 9),
    count_pairs_with_sum((), 0),
) == (8, 6, 0, 0)
```

## Trade-offs and Limitations

This does two linear passes, including the sortedness check, with constant auxiliary storage when indexing is constant time. Do not mutate the sequence during the call. The result counts positions, not distinct value pairs or ordered pairs.

## Related Snippets

<!-- catalog:related:start -->
- [Enumerate the K Smallest Pair Sums from Two Sorted Integer Sequences](enumerate-the-k-smallest-pair-sums-from-two-sorted-integer-sequences.md)
- [Count Equal-Value Index Pairs in Offline Ranges with Mo's Algorithm](count-equal-value-index-pairs-in-offline-ranges-with-mos-algorithm.md)
<!-- catalog:related:end -->
