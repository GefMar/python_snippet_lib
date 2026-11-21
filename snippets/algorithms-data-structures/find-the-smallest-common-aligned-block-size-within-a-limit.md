---
title: "Find the Smallest Common Aligned Block Size Within a Limit"
snippet_type: recipe
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - combine-a-bounded-system-of-possibly-non-coprime-integer-congruences.md
  - ../data-processing/overlay-aligned-time-series-at-the-finest-step.md
---

# Find the Smallest Common Aligned Block Size Within a Limit

## Idea and Problem

Choose the smallest positive buffer size that aligns to several block widths without exceeding a cap.

The least common multiple is the first possible aligned size. Accumulate it across the widths and stop when it exceeds the cap, because adding more alignment requirements cannot reduce it.

## When to Use

Use this when a transfer or file layout has several simultaneous positive alignment widths and one hard size limit. A result of None means that there is no positive multiple within that limit.

## Implementation

```python
from collections.abc import Iterable
from math import lcm


def common_aligned_size(widths: Iterable[int], cap: int) -> int | None:
    if type(cap) is not int or cap < 1:
        raise ValueError("cap must be a positive integer")

    alignment = 1
    seen_width = False
    for width in widths:
        if type(width) is not int or width < 1:
            raise ValueError("widths must be positive integers")
        seen_width = True
        alignment = lcm(alignment, width)
        if alignment > cap:
            return None

    if not seen_width:
        raise ValueError("expected at least one width")
    return alignment
```

## Example

```python
assert (
    common_aligned_size(iter([12, 18, 8]), 72),
    common_aligned_size([12, 18, 8], 71),
    common_aligned_size([5, 5, 1], 100),
) == (72, None, 5)
```

## Trade-offs and Limitations

The cap is on the resulting count, not memory already allocated by a caller. It does not impose a minimum target other than a positive size. On early rejection this deliberately does not consume or validate the rest of the iterable.

## Related Snippets

<!-- catalog:related:start -->
- [Combine a Bounded System of Possibly Non-Coprime Integer Congruences](combine-a-bounded-system-of-possibly-non-coprime-integer-congruences.md)
- [Overlay Aligned Time Series at the Finest Step](../data-processing/overlay-aligned-time-series-at-the-finest-step.md)
<!-- catalog:related:end -->
