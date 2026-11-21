---
title: "Apportion an Integer Total Evenly Under Per-Recipient Caps"
snippet_type: algorithm
use_cases:
  - data-transformation
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - apportion-a-non-negative-integer-total-without-rounding-drift.md
  - generate-smooth-weighted-round-robin-slots-for-fixed-consumers.md
---

# Apportion an Integer Total Evenly Under Per-Recipient Caps

## Idea and Problem

Distribute a fixed number of whole units in input order while giving each recipient its share up to its own capacity.

Imagine passing one unit to every eligible recipient per round and skipping
recipients once they reach their caps. At level `q`, complete rounds have
given each recipient `min(cap, q)` units. The sum is monotone in `q`, so find
the highest affordable level by binary search. Give the remaining units to
the first recipients eligible in the final partial round.

## When to Use

Use this to divide unweighted slots, capacity or batch work among recipients
with different hard limits. Provide a finite sequence or iterable of
non-negative integer caps. The output follows that order, which also decides
who receives any final extra unit. The total must fit the combined capacity.

## Implementation

```python
from collections.abc import Iterable


def apportion_with_caps(total: int, caps: Iterable[int]) -> tuple[int, ...]:
    if type(total) is not int:
        raise TypeError("total must be an integer")
    if total < 0:
        raise ValueError("total must be non-negative")

    capacities = tuple(caps)
    if any(type(cap) is not int for cap in capacities):
        raise TypeError("caps must be integers")
    if any(cap < 0 for cap in capacities):
        raise ValueError("caps must be non-negative")
    if total > sum(capacities):
        raise ValueError("total exceeds combined capacity")

    low, high = 0, min(total, max(capacities, default=0))
    while low < high:
        level = (low + high + 1) // 2
        if sum(min(cap, level) for cap in capacities) <= total:
            low = level
        else:
            high = level - 1

    shares = [min(cap, low) for cap in capacities]
    remaining = total - sum(shares)
    for index, cap in enumerate(capacities):
        if remaining == 0:
            break
        if shares[index] < cap:
            shares[index] += 1
            remaining -= 1
    return tuple(shares)
```

## Example

```python
assert (
    apportion_with_caps(8, (1, 6, 4, 0)),
    apportion_with_caps(4, (3, 3, 3)),
    apportion_with_caps(5, (2, 3)),
    apportion_with_caps(0, ()),
) == ((1, 4, 3, 0), (2, 1, 1), (2, 3), ())
```

## Trade-offs and Limitations

The function consumes the cap iterable before allocating anything and returns
a complete tuple. Each binary search step scans the caps once. The number of
steps grows logarithmically in the smaller of the total and the largest cap,
so the method avoids a separate iteration for every unit allocated.

This allocation treats each eligible recipient equally. Repeated calls with
the same input order favor the same recipients in partial rounds. A persistent
rotation, unequal weights or minimum guarantees require a different policy.

## Related Snippets

<!-- catalog:related:start -->
- [Apportion a Non-Negative Integer Total Without Rounding Drift](apportion-a-non-negative-integer-total-without-rounding-drift.md)
- [Generate Smooth Weighted Round-Robin Slots for Fixed Consumers](generate-smooth-weighted-round-robin-slots-for-fixed-consumers.md)
<!-- catalog:related:end -->
