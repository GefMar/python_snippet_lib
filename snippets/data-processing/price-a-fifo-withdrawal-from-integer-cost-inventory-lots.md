---
title: "Price a FIFO Withdrawal from Integer-Cost Inventory Lots"
snippet_type: recipe
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../configuration-serialization/convert-decimal-values-to-exact-minor-units.md
  - stream-count-deltas-between-two-sorted-inventories.md
---

# Price a FIFO Withdrawal from Integer-Cost Inventory Lots

## Idea and Problem

Calculate the exact cost of drawing units from the oldest inventory lots and return the ordered lots that remain.

Each lot has a quantity and one unit cost. Consume as much as needed from
each lot in receipt order. Multiply the quantity taken by its unit cost,
and keep any unused portion at the head of the remaining inventory. A large
lot requires one arithmetic step for its contribution.

## When to Use

Use this to check or preview a first-in, first-out withdrawal from a finite
inventory snapshot. Supply lots oldest first, with positive whole-unit
quantities and non-negative unit costs in the same integer minor unit. Use a
separate persistence layer to apply the result to live inventory.

## Implementation

```python
from collections.abc import Iterable
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class InventoryLot:
    quantity: int
    unit_cost: int

    def __post_init__(self) -> None:
        if type(self.quantity) is not int or type(self.unit_cost) is not int:
            raise TypeError("quantity and unit_cost must be integers")
        if self.quantity <= 0 or self.unit_cost < 0:
            raise ValueError("quantity must be positive and unit_cost non-negative")


@dataclass(frozen=True, slots=True)
class Withdrawal:
    cost: int
    remaining: tuple[InventoryLot, ...]


def price_fifo_withdrawal(lots: Iterable[InventoryLot], quantity: int) -> Withdrawal:
    if type(quantity) is not int:
        raise TypeError("requested quantity must be an integer")
    if quantity < 0:
        raise ValueError("requested quantity must be non-negative")

    outstanding = quantity
    cost = 0
    remaining: list[InventoryLot] = []
    for lot in lots:
        if type(lot) is not InventoryLot:
            raise TypeError("lots must contain InventoryLot values")
        taken = min(lot.quantity, outstanding)
        outstanding -= taken
        cost += taken * lot.unit_cost
        if taken == 0:
            remaining.append(lot)
        elif taken < lot.quantity:
            remaining.append(InventoryLot(lot.quantity - taken, lot.unit_cost))

    if outstanding:
        raise ValueError("requested quantity exceeds available inventory")
    return Withdrawal(cost, tuple(remaining))
```

## Example

```python
lots = (InventoryLot(3, 400), InventoryLot(5, 700), InventoryLot(2, 800))
try:
    price_fifo_withdrawal(lots, quantity=11)
except ValueError:
    overdraw_rejected = True
else:
    overdraw_rejected = False

assert (
    price_fifo_withdrawal(iter(lots), quantity=5),
    price_fifo_withdrawal(lots, quantity=0),
    overdraw_rejected,
) == (Withdrawal(2600, (InventoryLot(3, 700), InventoryLot(2, 800))), Withdrawal(0, lots), True)
```

## Trade-offs and Limitations

The function consumes the lot iterable once and retains the remaining lots
as immutable records in their original order. On overdraw it raises before
returning a result. Callers that pass a one-shot iterator need a fresh source
to retry.

Receipt order and unit cost are inputs. Returns, adjustments, fractional
quantities and mixed currencies need additional policy. For live inventory,
verify that the snapshot is still current when applying the withdrawal.

## Related Snippets

<!-- catalog:related:start -->
- [Convert Decimal Values to Exact Minor Units](../configuration-serialization/convert-decimal-values-to-exact-minor-units.md)
- [Stream Count Deltas Between Two Sorted Inventories](stream-count-deltas-between-two-sorted-inventories.md)
<!-- catalog:related:end -->
