---
title: "Compute Full-Window Trailing Maxima with a Monotonic Index Deque"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../machine-learning-statistics/compute-exact-full-window-trailing-medians-for-bounded-integers.md
  - ../machine-learning-statistics/compute-a-row-wise-maximum-of-rolling-minima.md
  - ../data-processing/yield-stream-items-with-bounded-neighbor-context.md
---

# Compute Full-Window Trailing Maxima with a Monotonic Index Deque

## Idea and Problem

Keep only useful candidate indices in a decreasing deque to compute every complete fixed-width window maximum in linear time.

Expired indices leave from the front. Before each new index enters at the back,
smaller or equal values are removed because the newer value will remain in the
window at least as long. The front therefore identifies the current maximum
without rescanning the window.

## When to Use

Use this algorithm for a bounded in-memory integer sequence when every complete
count-based trailing window needs a maximum and repeatedly calling `max()` would
do unnecessary work. Output position `i` summarizes
`values[i:i + width]` and ends at input position `i + width - 1`.

The complete input must be available for validation before calculation. Choose
a different contract for time-based windows, partial leading results, mutable
updates, or an unbounded stream.

## Implementation

```python
from collections import deque

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_VALUE_COUNT = 5_000


def full_window_trailing_maxima(
    values: tuple[int, ...],
    *,
    width: int,
) -> tuple[int, ...]:
    """Return one maximum for every complete fixed-width trailing window."""
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    if not 1 <= len(values) <= _MAX_VALUE_COUNT:
        raise ValueError("value count is outside the supported range")
    if type(width) is not int:
        raise TypeError("width must be an exact integer")
    if not 1 <= width <= len(values):
        raise ValueError("width must be between one and the value count")

    for value in values:
        if type(value) is not int:
            raise TypeError("every value must be an exact integer")
        if not _MIN_INT64 <= value <= _MAX_INT64:
            raise ValueError("every value must be in the signed 64-bit range")

    candidate_indices: deque[int] = deque()
    maxima: list[int] = []
    for index, value in enumerate(values):
        expired_index = index - width
        if candidate_indices and candidate_indices[0] <= expired_index:
            candidate_indices.popleft()

        while candidate_indices and values[candidate_indices[-1]] <= value:
            candidate_indices.pop()
        candidate_indices.append(index)

        if index >= width - 1:
            maxima.append(values[candidate_indices[0]])

    return tuple(maxima)
```

## Example

```python
samples = (4, 2, 2, 5, 1, 5, 3)
maxima = full_window_trailing_maxima(samples, width=3)

try:
    full_window_trailing_maxima((1, True, 3), width=2)
except TypeError:
    bool_value_rejected = True
else:
    bool_value_rejected = False

try:
    full_window_trailing_maxima(samples, width=True)
except TypeError:
    bool_width_rejected = True
else:
    bool_width_rejected = False

assert (
    maxima,
    full_window_trailing_maxima(samples, width=1),
    full_window_trailing_maxima(samples, width=len(samples)),
    bool_value_rejected,
    bool_width_rejected,
) == (
    (4, 5, 5, 5, 5),
    samples,
    (5,),
    True,
    True,
)
```

## Trade-offs and Limitations

Every index enters and leaves the deque at most once, so calculation takes
`O(n)` time after `O(n)` validation. The deque stores at most `width` indices,
and the materialized result stores `n - width + 1` integers.

Equal candidates keep only the newest index internally, but the public result
contains values only and defines no first-or-last argmax policy. Inputs are
exact signed 64-bit integers; floats, decimals, missing values, partial leading
windows, centered windows, time-based windows, incremental replacement, and
unbounded-stream behavior are outside the contract.

## Related Snippets

<!-- catalog:related:start -->
- [Compute Exact Full-Window Trailing Medians for Bounded Integers](../machine-learning-statistics/compute-exact-full-window-trailing-medians-for-bounded-integers.md)
- [Compute a Row-Wise Maximum of Rolling Minima](../machine-learning-statistics/compute-a-row-wise-maximum-of-rolling-minima.md)
- [Yield Stream Items with Bounded Neighbor Context](../data-processing/yield-stream-items-with-bounded-neighbor-context.md)
<!-- catalog:related:end -->
