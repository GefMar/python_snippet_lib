---
title: "Compute Exact Trailing Sample Covariance for Integer Pairs"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-exact-squared-pearson-correlation-with-direction.md
  - compute-exact-full-window-trailing-medians-for-bounded-integers.md
---

# Compute Exact Trailing Sample Covariance for Integer Pairs

## Idea and Problem

Compute sample covariance for each complete trailing window of paired integers without recalculating both means at every step.

For a window of `n` pairs, the sum of centered products is
`sum(x * y) - sum(x) * sum(y) / n`. Dividing by `n - 1` yields sample
covariance. Maintain the three sums by subtracting the departing pair and
adding the incoming one. Combining the fractions into one expression keeps
all intermediate sums exact integers.

## When to Use

Use this for paired integer measurements already aligned in input order.
Choose a width of at least two observations. The first result describes the
first complete window, and each subsequent pair produces one more result.
This window advances by observation count. Apply any missing-value or
timestamp alignment policy before computing covariance.

## Implementation

```python
from collections import deque
from collections.abc import Iterable, Iterator
from fractions import Fraction


def trailing_sample_covariance(pairs: Iterable[tuple[int, int]], width: int) -> Iterator[Fraction]:
    """Yield exact sample covariance for each complete window of width pairs."""
    if type(width) is not int:
        raise TypeError("width must be an integer")
    if width < 2:
        raise ValueError("sample covariance needs at least two pairs per window")

    window: deque[tuple[int, int]] = deque()
    sum_x = sum_y = sum_xy = 0
    denominator = width * (width - 1)

    for x, y in pairs:
        if type(x) is not int or type(y) is not int:
            raise TypeError("each observation must contain two integers")
        if len(window) == width:
            old_x, old_y = window.popleft()
            sum_x -= old_x
            sum_y -= old_y
            sum_xy -= old_x * old_y

        window.append((x, y))
        sum_x += x
        sum_y += y
        sum_xy += x * y

        if len(window) == width:
            yield Fraction(width * sum_xy - sum_x * sum_y, denominator)
```

## Example

```python
observations = ((0, 0), (1, 2), (2, 4), (3, -1))
constant_first_coordinate = ((9, 0), (9, 1), (9, 2))

assert (
    tuple(trailing_sample_covariance(iter(observations), width=3)),
    tuple(trailing_sample_covariance(constant_first_coordinate, width=3)),
    tuple(trailing_sample_covariance(observations, width=5)),
) == ((Fraction(2), Fraction(-3, 2)), (Fraction(0),), ())
```

## Trade-offs and Limitations

The deque retains at most `width` pairs. Each update uses a fixed number of
arithmetic operations and one fraction reduction when a window is complete.
The cost of those operations grows with the size of the integers. Invalid
pairs raise during iteration, potentially after earlier results were yielded.

Covariance has units equal to the product of the two inputs' units, and a
constant coordinate gives zero covariance. Use correlation for a normalized
measure of linear association. To describe a complete population instead of
a sample, use the population denominator `n` in the centered formula.

## Related Snippets

<!-- catalog:related:start -->
- [Compute Exact Squared Pearson Correlation with Direction](compute-exact-squared-pearson-correlation-with-direction.md)
- [Compute Exact Full-Window Trailing Medians for Bounded Integers](compute-exact-full-window-trailing-medians-for-bounded-integers.md)
<!-- catalog:related:end -->
