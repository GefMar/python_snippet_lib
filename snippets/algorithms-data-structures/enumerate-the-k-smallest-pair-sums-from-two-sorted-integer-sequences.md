---
title: "Enumerate the K Smallest Pair Sums from Two Sorted Integer Sequences"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
tested_python:
  - "3.14"
dependencies: []
related:
  - merge-bounded-sorted-integer-runs-with-observable-source-order-ties.md
  - find-the-narrowest-integer-range-covering-every-sorted-source.md
---

# Enumerate the K Smallest Pair Sums from Two Sorted Integer Sequences

## Idea and Problem

List the smallest sums of pairs from two sorted inputs without constructing their Cartesian product.

For each left value, adding all right values defines a sorted row of sums.
A min-heap merges row heads in `(sum, left_index, right_index)` order. Only
the first `k` rows can matter to the first `k` results. Every later row's
first pair comes after at least `k` earlier row heads under that ordering.

## When to Use

Use this to rank a small number of combinations from two independently
ordered integer cost lists. Both sequences must support random access, stay
stable during iteration and be sorted in nondecreasing order. Each output
contains the sum and the two original indices, so equal values at different
positions remain distinct choices.

## Implementation

```python
from collections.abc import Iterator, Sequence
from heapq import heapify, heappop, heappush


def smallest_pair_sums(
    left: Sequence[int], right: Sequence[int], k: int
) -> Iterator[tuple[int, int, int]]:
    """Yield up to k pairs ordered by sum, then left and right indices."""
    if type(k) is not int or k < 0:
        raise ValueError("k must be a non-negative integer")
    right_count = len(right)
    if right_count == 0 or k == 0:
        return

    heap = [(left[i] + right[0], i, 0) for i in range(min(len(left), k))]
    heapify(heap)

    for _ in range(k):
        if not heap:
            break
        pair = heappop(heap)
        yield pair
        _, i, j = pair
        if j + 1 < right_count:
            heappush(heap, (left[i] + right[j + 1], i, j + 1))
```

## Example

```python
base_prices = (1, 1, 4)
option_prices = (0, 2)

assert (
    tuple(smallest_pair_sums(base_prices, option_prices, k=4)),
    tuple(smallest_pair_sums((-2,), (1,), k=10)),
    tuple(smallest_pair_sums(base_prices, (), k=4)),
) == (
    ((1, 0, 0), (1, 1, 0), (3, 0, 1), (3, 1, 1)),
    ((-1, 0, 0),),
    (),
)
```

## Trade-offs and Limitations

The heap holds at most `min(k, len(left))` entries. After initialization,
each emitted pair costs one pop and at most one push. The iterator stops
after all pairs are exhausted when `k` exceeds their number.

The sorted-order precondition belongs to the caller. The algorithm ranks
additive costs and allows any left choice to be combined with any right
choice. Pair-specific costs or compatibility filters require a different
ranking strategy.

## Related Snippets

<!-- catalog:related:start -->
- [Merge Bounded Sorted Integer Runs with Observable Source-Order Ties](merge-bounded-sorted-integer-runs-with-observable-source-order-ties.md)
- [Find the Narrowest Integer Range Covering Every Sorted Source](find-the-narrowest-integer-range-covering-every-sorted-source.md)
<!-- catalog:related:end -->
