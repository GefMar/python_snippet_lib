---
title: "Compute Exact Full-Window Trailing Medians for Bounded Integers"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - create-past-only-pandas-lag-and-rolling-mean-columns.md
  - compute-a-row-wise-maximum-of-rolling-minima.md
  - calculate-a-symmetrically-trimmed-mean.md
---

# Compute Exact Full-Window Trailing Medians for Bounded Integers

## Idea and Problem

Maintain one sorted list for a bounded trailing window and return its exact median at every position with complete history.

The first window is sorted once. Each later step removes one outgoing value
with `bisect_left` and inserts one incoming value with `insort`, so duplicates
remain valid without sorting the complete window again. Odd widths select the
middle integer; even widths return the exact midpoint as a `Fraction`.

## When to Use

Use this algorithm for a small in-memory integer series when the window is
defined by item count, every complete trailing window needs a median, and
binary floating-point rounding is unacceptable. Output position `i`
summarizes `values[i:i + width]` and ends at input position `i + width - 1`.

The complete input must be available for validation before calculation. Use a
specialized order-statistic structure when the series or window is large, and
define time, lateness, or missing-value semantics separately for irregular
observations.

## Implementation

```python
from bisect import bisect_left, insort
from fractions import Fraction

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_VALUE_COUNT = 5_000


def _median_of_sorted_integers(window: list[int]) -> Fraction:
    midpoint = len(window) // 2
    if len(window) % 2:
        return Fraction(window[midpoint])
    return Fraction(window[midpoint - 1] + window[midpoint], 2)


def exact_trailing_medians(
    values: tuple[int, ...],
    *,
    width: int,
) -> tuple[Fraction, ...]:
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

    sorted_window = sorted(values[:width])
    medians = [_median_of_sorted_integers(sorted_window)]

    for incoming_index in range(width, len(values)):
        outgoing = values[incoming_index - width]
        outgoing_index = bisect_left(sorted_window, outgoing)
        sorted_window.pop(outgoing_index)
        insort(sorted_window, values[incoming_index])
        medians.append(_median_of_sorted_integers(sorted_window))

    return tuple(medians)
```

## Example

```python
samples = (7, 1, 9, 3, 8, 2)
odd_width = exact_trailing_medians(samples, width=3)
even_width = exact_trailing_medians(samples, width=4)

try:
    exact_trailing_medians(list(samples), width=3)  # type: ignore[arg-type]
except TypeError:
    list_input_rejected = True
else:
    list_input_rejected = False

try:
    exact_trailing_medians((1, True, 3), width=2)
except TypeError:
    bool_value_rejected = True
else:
    bool_value_rejected = False

try:
    exact_trailing_medians(samples, width=True)
except TypeError:
    bool_width_rejected = True
else:
    bool_width_rejected = False

assert (
    odd_width,
    even_width,
    samples,
    list_input_rejected,
    bool_value_rejected,
    bool_width_rejected,
) == (
    (Fraction(7, 1), Fraction(3, 1), Fraction(8, 1), Fraction(3, 1)),
    (Fraction(5, 1), Fraction(11, 2), Fraction(11, 2)),
    (7, 1, 9, 3, 8, 2),
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

Validation takes `O(n)` time, and the first sort takes `O(width log width)`.
Each later binary search costs `O(log width)`, but removing from and inserting
into a Python list can move `O(width)` references. Total worst-case work is
therefore `O(n * width)`, not logarithmic per window. Working state uses
`O(width)` memory, while the materialized result uses `O(n - width + 1)`
`Fraction` objects.

Only complete trailing windows are returned; there are no partial leading
results. Inputs are exact signed 64-bit integers, while Python's unbounded
integer arithmetic keeps the sum for an even median exact. Fractions may need
an explicit conversion policy at serialization boundaries. The function has
no float, decimal, weighted, centered, time-based, approximate, missing-value,
incremental-update, or unbounded-stream behavior.

## Related Snippets

<!-- catalog:related:start -->
- [Create Past-Only pandas Lag and Rolling-Mean Columns](create-past-only-pandas-lag-and-rolling-mean-columns.md)
- [Compute a Row-Wise Maximum of Rolling Minima](compute-a-row-wise-maximum-of-rolling-minima.md)
- [Calculate a Symmetrically Trimmed Mean](calculate-a-symmetrically-trimmed-mean.md)
<!-- catalog:related:end -->
