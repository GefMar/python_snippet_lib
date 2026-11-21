---
title: "Split an Integer Circular Window into Linear Half-Open Spans"
snippet_type: algorithm
use_cases:
  - data-transformation
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - ../data-processing/normalize-a-finite-binary64-value-into-a-canonical-half-open-period.md
  - ../storage-databases/split-a-half-open-utc-range-across-ordered-storage-tiers.md
---

# Split an Integer Circular Window into Linear Half-Open Spans

## Idea and Problem

Describe a forward window on a circular integer index as at most two ordinary slice ranges in traversal order.

The window starts at one canonical index and covers a supplied number of
positions. If the end falls beyond the period, stop the first span at that
boundary and continue from zero. An explicit width distinguishes a window
with no positions from a full circle whose end also returns to the start.

## When to Use

Use this for ring-buffer slices or other circular index ranges with a
positive integer period. The start must be in `[0, period)` and the width
in `[0, period]`. Each returned pair is a nonempty half-open span. Apply the
spans in the returned order to visit positions starting from the supplied
index, even when a full circle wraps.

## Implementation

```python
def split_circular_window(start: int, width: int, period: int) -> tuple[tuple[int, int], ...]:
    if any(type(value) is not int for value in (start, width, period)):
        raise TypeError("start, width and period must be integers")
    if period <= 0:
        raise ValueError("period must be positive")
    if not 0 <= start < period or not 0 <= width <= period:
        raise ValueError("start must be in [0, period) and width in [0, period]")
    if width == 0:
        return ()

    end = start + width
    if end <= period:
        return ((start, end),)
    return (start, period), (0, end - period)
```

## Example

```python
buffer = tuple("ABCDEFGH")
wrapped = split_circular_window(start=6, width=4, period=len(buffer))
selected = tuple(item for start, end in wrapped for item in buffer[start:end])

assert (
    wrapped,
    selected,
    split_circular_window(6, 0, 8),
    split_circular_window(6, 8, 8),
    split_circular_window(6, 2, 8),
) == (((6, 8), (0, 2)), ("G", "H", "A", "B"), (), ((6, 8), (0, 6)), ((6, 8),))
```

## Trade-offs and Limitations

The function uses a fixed number of arithmetic operations and returns at
most two spans. A span ending exactly at the period needs no second span.
These are indices only, so slicing or copying the underlying data remains
the caller's choice.

The model describes one forward traversal of at most one revolution. For
several full revolutions, keep the count of whole cycles separately. Normalize
an external starting index explicitly when it can lie outside the period.

## Related Snippets

<!-- catalog:related:start -->
- [Normalize a Finite Binary64 Value into a Canonical Half-Open Period](../data-processing/normalize-a-finite-binary64-value-into-a-canonical-half-open-period.md)
- [Split a Half-Open UTC Range Across Ordered Storage Tiers](../storage-databases/split-a-half-open-utc-range-across-ordered-storage-tiers.md)
<!-- catalog:related:end -->
