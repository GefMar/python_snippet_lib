---
title: "Round Signed Integer Quantities to a Step with Ties to Even"
snippet_type: recipe
use_cases:
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - build-quantized-decimal-views-with-deltas-at-explicit-scales.md
  - ../algorithms-data-structures/apportion-a-non-negative-integer-total-without-rounding-drift.md
---

# Round Signed Integer Quantities to a Step with Ties to Even

## Idea and Problem

Round an integer quantity to its nearest multiple of an arbitrary positive step without converting to floating point.

Use the quotient and remainder to compare the distances to the two adjacent multiples. If they are equally distant, choose the one with an even step index. With a positive divisor, floor division makes the remainder nonnegative even for a negative input.

## When to Use

Use this for signed values already expressed in whole units, such as aligning measured tick counts to a fixed output resolution. Choose this convention only when halfway cases should prefer an even multiple index rather than always rounding toward zero or away from it.

## Implementation

```python
def round_to_step(value: int, step: int) -> int:
    if type(value) is not int or type(step) is not int:
        raise TypeError("value and step must be integers")
    if step <= 0:
        raise ValueError("step must be positive")

    lower_index, remainder = divmod(value, step)
    doubled = 2 * remainder
    if doubled > step or (doubled == step and lower_index % 2):
        lower_index += 1
    return lower_index * step
```

## Example

```python
values = (-15, -10, -5, -4, 4, 5, 10, 15)
rounded = tuple(round_to_step(value, 10) for value in values)
base = 10**40

try:
    round_to_step(8, 0)
except ValueError:
    rejected_zero_step = True
else:
    rejected_zero_step = False

assert (
    rounded,
    round_to_step(base + 5, 10),
    round_to_step(11, 7),
    rejected_zero_step,
) == ((-20, -10, 0, 0, 0, 0, 10, 20), base, 14, True)
```

## Trade-offs and Limitations

The even choice refers to the integer index of the multiple, not the rounded value itself. This rounds each value independently; it does not preserve the sum of a collection or guarantee monotonic progress by a minimum amount. For non-integer amounts, choose an exact input representation before applying a rounding policy.

## Related Snippets

<!-- catalog:related:start -->
- [Build Quantized Decimal Views with Deltas at Explicit Scales](build-quantized-decimal-views-with-deltas-at-explicit-scales.md)
- [Apportion a Non-Negative Integer Total Without Rounding Drift](../algorithms-data-structures/apportion-a-non-negative-integer-total-without-rounding-drift.md)
<!-- catalog:related:end -->
