---
title: "Yield Only Complete Trailing Windows from a One-Shot Iterable"
snippet_type: recipe
use_cases:
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - yield-stream-items-with-bounded-neighbor-context.md
  - ../machine-learning-statistics/compute-exact-full-window-trailing-medians-for-bounded-integers.md
---

# Yield Only Complete Trailing Windows from a One-Shot Iterable

## Idea and Problem

Yield each full trailing window of a one-shot input without retaining every preceding item.

A bounded deque evicts the oldest item as new values arrive. Take an immutable tuple for each completed window so continuing the iteration does not replace the previous window.

## When to Use

Use this when a consumer needs the contents of complete fixed-width windows and does not want partial warm-up or padding. The first window requires exactly as many items as its width.

## Implementation

```python
from collections import deque
from collections.abc import Iterable, Iterator


def full_windows[T](items: Iterable[T], width: int) -> Iterator[tuple[T, ...]]:
    if type(width) is not int or width < 1:
        raise ValueError("window width must be a positive integer")

    recent: deque[T] = deque(maxlen=width)
    for item in items:
        recent.append(item)
        if len(recent) == width:
            yield tuple(recent)
```

## Example

```python
items = iter((3, 8, 2, 5, 9))
windows = full_windows(items, 3)
first = next(windows)
rest = tuple(windows)

assert (first, rest, tuple(full_windows(iter([1]), 2)), tuple(full_windows([7, 4], 1))) == (
    (3, 8, 2),
    ((8, 2, 5), (2, 5, 9)),
    (),
    ((7,), (4,)),
)
```

## Trade-offs and Limitations

It retains at most the window width in internal state but allocates a new tuple for each result. Tuple elements remain references to the original values. Validation and input consumption begin when iterating, not when creating the generator. For an aggregate such as a running sum, maintaining the aggregate can be cheaper than copying each window.

## Related Snippets

<!-- catalog:related:start -->
- [Yield Stream Items with Bounded Neighbor Context](yield-stream-items-with-bounded-neighbor-context.md)
- [Compute Exact Full-Window Trailing Medians for Bounded Integers](../machine-learning-statistics/compute-exact-full-window-trailing-medians-for-bounded-integers.md)
<!-- catalog:related:end -->
