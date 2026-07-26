---
title: "Cover a Half-Open Integer Range with Dyadic Intervals"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
tested_python:
  - "3.14"
dependencies: []
related: []
---

# Cover a Half-Open Integer Range with Dyadic Intervals

## Idea and Problem

Decompose a non-negative half-open integer range into a small ordered set of aligned power-of-two blocks without gaps or overlaps.

Dyadic blocks have a power-of-two size and begin at a multiple of that size.
Choosing the largest valid block at each position produces a deterministic
cover suitable for querying pre-aggregated buckets or describing compact
integer ranges.

## When to Use

Use this algorithm when another data structure already stores summaries for
aligned power-of-two intervals. Inputs must use half-open semantics
`[start, stop)` and non-negative integer coordinates. For arbitrary real
intervals, inclusive endpoints, or direct iteration over a small range, first
normalize the contract or use a simpler representation.

## Implementation

```python
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class DyadicInterval:
    start: int
    stop: int

    @property
    def size(self) -> int:
        return self.stop - self.start


def cover_with_dyadic_intervals(
    start: int,
    stop: int,
) -> tuple[DyadicInterval, ...]:
    if type(start) is not int or type(stop) is not int:
        raise TypeError("range bounds must be integers")
    if start < 0 or stop < 0:
        raise ValueError("range bounds must not be negative")
    if start > stop:
        raise ValueError("start must not exceed stop")

    blocks = []
    cursor = start
    while cursor < stop:
        remaining = stop - cursor
        largest_remaining_block = 1 << (remaining.bit_length() - 1)
        alignment = cursor & -cursor
        block_size = (
            largest_remaining_block
            if alignment == 0
            else min(alignment, largest_remaining_block)
        )
        blocks.append(DyadicInterval(cursor, cursor + block_size))
        cursor += block_size

    return tuple(blocks)
```

## Example

```python
cover = cover_with_dyadic_intervals(3, 11)
covered_points = tuple(
    point
    for block in cover
    for point in range(block.start, block.stop)
)
valid_blocks = all(
    block.size > 0
    and block.size & (block.size - 1) == 0
    and block.start % block.size == 0
    for block in cover
)

try:
    cover_with_dyadic_intervals(-1, 3)
except ValueError:
    negative_rejected = True
else:
    negative_rejected = False

try:
    cover_with_dyadic_intervals(4, 3)
except ValueError:
    reversed_rejected = True
else:
    reversed_rejected = False

assert (
    cover,
    covered_points,
    valid_blocks,
    cover_with_dyadic_intervals(0, 8),
    cover_with_dyadic_intervals(5, 5),
    cover_with_dyadic_intervals(7, 8),
    negative_rejected,
    reversed_rejected,
) == (
    (
        DyadicInterval(3, 4),
        DyadicInterval(4, 8),
        DyadicInterval(8, 10),
        DyadicInterval(10, 11),
    ),
    tuple(range(3, 11)),
    True,
    (DyadicInterval(0, 8),),
    (),
    (DyadicInterval(7, 8),),
    True,
    True,
)
```

## Trade-offs and Limitations

The cover describes intervals; it does not build or update the summaries that
make them useful. Output size is logarithmic for typical ranges but depends on
endpoint alignment. The greedy result is specific to non-negative integers
and half-open bounds. Converting an inclusive range by blindly adding one to
its upper endpoint can overflow or violate an external domain, so normalization
belongs at that boundary.

## Related Snippets

<!-- catalog:related:start -->
No related snippets.
<!-- catalog:related:end -->
