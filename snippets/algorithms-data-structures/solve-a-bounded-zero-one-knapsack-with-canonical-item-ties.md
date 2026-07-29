---
title: "Solve a Bounded Zero-One Knapsack with Canonical Item Ties"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - decide-bounded-non-negative-subset-sum-reachability-with-an-integer-bitset.md
  - select-a-maximum-weight-set-of-non-overlapping-half-open-integer-intervals.md
  - find-a-lexicographically-first-minimum-cost-perfect-assignment-by-bitmask-dp.md
---

# Solve a Bounded Zero-One Knapsack with Canonical Item Ties

## Idea and Problem

Choose a capacity-feasible subset with maximum value while making weight and item-name ties deterministic and independent of input order.

A suffix dynamic program stores the best attainable value and, among equal
values, the least used weight for each remaining capacity. It does not store a
selected-name tuple in every cell. Instead, reconstruction scans lexically
normalized items and takes the earliest item exactly when doing so can still
attain the remaining optimal value-and-weight pair.

That reconstruction rule selects the lexicographically smallest tuple of names
without multiplying every DP cell by a growing witness tuple.

## When to Use

Use this bounded solver when every named item can be chosen at most once, has
one positive integer weight and one non-negative integer value, and a complete
pseudo-polynomial DP table fits the declared state budget. It is useful for
small deterministic allocation, packing, and reference-oracle problems.

Use a specialized optimizer when capacity is large, several resource
dimensions interact, fractional choices are allowed, or an approximation is
preferable to `O(item_count * capacity)` work and state.

## Implementation

```python
from array import array
from dataclasses import dataclass

_MAX_ITEMS = 64
_MAX_CAPACITY = 10_000
_MAX_WEIGHT = 10_000
_MAX_VALUE = 1_000_000_000
_MAX_TOTAL_VALUE = (1 << 63) - 1
_MAX_STATE_CELLS = 500_000
_MAX_NAME_LENGTH = 64
_NAME_CHARACTERS = frozenset(
    "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz._-"
)


@dataclass(frozen=True, slots=True)
class KnapsackItem:
    name: str
    weight: int
    value: int


@dataclass(frozen=True, slots=True)
class KnapsackResult:
    maximum_value: int
    total_weight: int
    selected_names: tuple[str, ...]


def _validate_name(value: object) -> str:
    if type(value) is not str:
        raise TypeError("item name must be an exact string")
    if not 1 <= len(value) <= _MAX_NAME_LENGTH:
        raise ValueError("item name length is outside the supported range")
    if any(character not in _NAME_CHARACTERS for character in value):
        raise ValueError("item name contains a forbidden character")
    return value


def solve_zero_one_knapsack(
    items: tuple[KnapsackItem, ...],
    *,
    capacity: int,
) -> KnapsackResult:
    """Return the canonical maximum-value subset under one capacity."""
    if type(items) is not tuple:
        raise TypeError("items must be an exact tuple")
    if len(items) > _MAX_ITEMS:
        raise ValueError("item count exceeds the supported limit")
    if type(capacity) is not int:
        raise TypeError("capacity must be an exact integer")
    if not 0 <= capacity <= _MAX_CAPACITY:
        raise ValueError("capacity is outside the supported range")

    validated_items: list[KnapsackItem] = []
    names: set[str] = set()
    total_value = 0
    for item in items:
        if type(item) is not KnapsackItem:
            raise TypeError("items must contain exact KnapsackItem values")
        name = _validate_name(item.name)
        if name in names:
            raise ValueError("item names must be unique")
        names.add(name)
        if type(item.weight) is not int:
            raise TypeError("item weight must be an exact integer")
        if not 1 <= item.weight <= _MAX_WEIGHT:
            raise ValueError("item weight is outside the supported range")
        if type(item.value) is not int:
            raise TypeError("item value must be an exact integer")
        if not 0 <= item.value <= _MAX_VALUE:
            raise ValueError("item value is outside the supported range")
        total_value += item.value
        if total_value > _MAX_TOTAL_VALUE:
            raise ValueError("total item value exceeds the signed 64-bit range")
        validated_items.append(KnapsackItem(name, item.weight, item.value))

    canonical_items = tuple(
        sorted(validated_items, key=lambda item: item.name)
    )
    state_cells = (len(canonical_items) + 1) * (capacity + 1)
    if state_cells > _MAX_STATE_CELLS:
        raise ValueError("knapsack DP state exceeds the supported limit")

    best_values = [
        array("q", [0]) * (capacity + 1)
        for _ in range(len(canonical_items) + 1)
    ]
    best_weights = [
        array("I", [0]) * (capacity + 1)
        for _ in range(len(canonical_items) + 1)
    ]

    for item_index in range(len(canonical_items) - 1, -1, -1):
        item = canonical_items[item_index]
        suffix_values = best_values[item_index + 1]
        suffix_weights = best_weights[item_index + 1]
        current_values = best_values[item_index]
        current_weights = best_weights[item_index]
        for available in range(capacity + 1):
            best_value = suffix_values[available]
            best_weight = suffix_weights[available]
            if item.weight <= available:
                suffix_capacity = available - item.weight
                take_value = item.value + suffix_values[suffix_capacity]
                take_weight = item.weight + suffix_weights[suffix_capacity]
                if take_value > best_value or (
                    take_value == best_value
                    and take_weight < best_weight
                ):
                    best_value = take_value
                    best_weight = take_weight
            current_values[available] = best_value
            current_weights[available] = best_weight

    remaining_capacity = capacity
    remaining_value = best_values[0][capacity]
    remaining_weight = best_weights[0][capacity]
    selected_names: list[str] = []
    for item_index, item in enumerate(canonical_items):
        if remaining_value == 0 and remaining_weight == 0:
            break
        if item.weight > remaining_capacity:
            continue
        suffix_capacity = remaining_capacity - item.weight
        if (
            item.value + best_values[item_index + 1][suffix_capacity]
            == remaining_value
            and item.weight + best_weights[item_index + 1][suffix_capacity]
            == remaining_weight
        ):
            selected_names.append(item.name)
            remaining_capacity = suffix_capacity
            remaining_value -= item.value
            remaining_weight -= item.weight

    if remaining_value != 0 or remaining_weight != 0:
        raise RuntimeError("knapsack witness reconstruction failed")
    return KnapsackResult(
        maximum_value=best_values[0][capacity],
        total_weight=best_weights[0][capacity],
        selected_names=tuple(selected_names),
    )
```

## Example

```python
items = (
    KnapsackItem("gamma", 4, 8),
    KnapsackItem("beta", 2, 4),
    KnapsackItem("alpha", 2, 4),
    KnapsackItem("unused", 7, 100),
)
result = solve_zero_one_knapsack(items, capacity=4)

assert result == KnapsackResult(
    maximum_value=8,
    total_weight=4,
    selected_names=("alpha", "beta"),
)
assert solve_zero_one_knapsack(
    tuple(reversed(items)),
    capacity=4,
) == result

least_weight = solve_zero_one_knapsack(
    (
        KnapsackItem("heavy", 5, 10),
        KnapsackItem("light", 4, 10),
    ),
    capacity=5,
)
assert least_weight.selected_names == ("light",)
assert least_weight.total_weight == 4

assert solve_zero_one_knapsack(
    (KnapsackItem("zero", 1, 0),),
    capacity=1,
) == KnapsackResult(0, 0, ())
```

## Trade-offs and Limitations

After `O(n log n)` name normalization, the suffix table uses `O(n * C)` time
and state for `n` items and capacity `C`. Signed 64-bit and unsigned 32-bit
standard-library arrays keep each DP cell fixed-width, while the explicit
500,000-cell cap bounds allocation.

Capacity makes the algorithm pseudo-polynomial: a numerically large capacity
is expensive even when encoded with few digits. The returned subset first
maximizes value, then minimizes weight, then chooses the lexicographically
smallest selected-name tuple. Other tie policies can legitimately return a
different optimum.

This implementation does not support fractional or unbounded choices,
negative values, repeated copies, multiple capacity dimensions, dependencies
between items, approximation, memory-linear witness reconstruction, or claims
that a bounded dynamic program is suitable for large optimization workloads.

## Related Snippets

<!-- catalog:related:start -->
- [Decide Bounded Non-Negative Subset-Sum Reachability with an Integer Bitset](decide-bounded-non-negative-subset-sum-reachability-with-an-integer-bitset.md)
- [Select a Maximum-Weight Set of Non-Overlapping Half-Open Integer Intervals](select-a-maximum-weight-set-of-non-overlapping-half-open-integer-intervals.md)
- [Find a Lexicographically First Minimum-Cost Perfect Assignment by Bitmask DP](find-a-lexicographically-first-minimum-cost-perfect-assignment-by-bitmask-dp.md)
<!-- catalog:related:end -->
