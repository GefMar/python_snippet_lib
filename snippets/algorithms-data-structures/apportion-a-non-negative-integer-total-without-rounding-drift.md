---
title: "Apportion a Non-Negative Integer Total Without Rounding Drift"
snippet_type: algorithm
use_cases:
  - data-transformation
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../configuration-serialization/convert-decimal-values-to-exact-minor-units.md
  - ../data-processing/batch-items-by-estimated-byte-size.md
---

# Apportion a Non-Negative Integer Total Without Rounding Drift

## Idea and Problem

Divide a non-negative integer into a bounded number of nearly equal integer shares while preserving the total exactly.

`divmod()` separates the common base from the remainder. Giving one extra unit
to each of the first remainder positions makes every share non-negative, keeps
the maximum difference at one, and avoids the accumulated drift caused by
rounding each fractional share independently.

## When to Use

Use this algorithm when identical recipients need whole-number units and input
order is an acceptable deterministic tie-breaker. Convert exact decimal amounts
to integer minor units before apportionment. Use a largest-remainder or
optimization method when shares have weights, priorities, minimums, maximums,
or fairness rules beyond equal division.

## Implementation

```python
_MAX_PARTS = 10_000


def apportion_integer_total(total: int, parts: int) -> tuple[int, ...]:
    if isinstance(total, bool) or not isinstance(total, int):
        raise TypeError("total must be an integer")
    if total < 0:
        raise ValueError("total must be non-negative")
    if isinstance(parts, bool) or not isinstance(parts, int):
        raise TypeError("parts must be an integer")
    if not 1 <= parts <= _MAX_PARTS:
        raise ValueError("parts is outside the supported range")

    base, remainder = divmod(total, parts)
    return tuple(
        base + (1 if index < remainder else 0)
        for index in range(parts)
    )
```

## Example

```python
allocations = {
    (101, 2): apportion_integer_total(101, 2),
    (2, 5): apportion_integer_total(2, 5),
    (0, 3): apportion_integer_total(0, 3),
}
invariants_hold = all(
    len(shares) == parts
    and sum(shares) == total
    and min(shares) >= 0
    and max(shares) - min(shares) <= 1
    for (total, parts), shares in allocations.items()
)

assert (allocations, invariants_hold) == (
    {
        (101, 2): (51, 50),
        (2, 5): (1, 1, 0, 0, 0),
        (0, 3): (0, 0, 0),
    },
    True,
)
```

## Trade-offs and Limitations

The first positions always receive the remainder, so repeated allocations can
create systematic front-of-list bias unless callers rotate or randomize a
separately governed order. The algorithm supports equal integer shares only;
it has no weights, fractional output, negative totals, minimums, maximums, or
cross-run fairness state. It materializes one integer per part, using `O(parts)`
time and memory even when many shares are zero. The 10,000-part ceiling is an
application safety bound, not a mathematical requirement.

## Related Snippets

<!-- catalog:related:start -->
- [Convert Decimal Values to Exact Minor Units](../configuration-serialization/convert-decimal-values-to-exact-minor-units.md)
- [Batch Items by Estimated Byte Size](../data-processing/batch-items-by-estimated-byte-size.md)
<!-- catalog:related:end -->
