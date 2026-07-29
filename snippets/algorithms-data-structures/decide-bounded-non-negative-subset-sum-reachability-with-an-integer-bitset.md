---
title: "Decide Bounded Non-Negative Subset-Sum Reachability with an Integer Bitset"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - find-the-first-non-empty-contiguous-integer-span-with-an-exact-sum.md
  - build-and-evaluate-a-bounded-binary-assignment-constraint-system.md
  - rank-and-unrank-index-combinations-in-itertools-combinations-order.md
---

# Decide Bounded Non-Negative Subset-Sum Reachability with an Integer Bitset

## Idea and Problem

Decide whether a bounded target is the sum of a subset of non-negative integers without storing one Python object per reachable sum.

Bit position `s` represents whether sum `s` is reachable. Shifting the bitset by
one input value represents adding that position once, and OR-ing the shifted
bits with the previous state preserves the option of excluding it. A mask keeps
only positions through the target, while values above the target are skipped
after validation so they never create oversized temporary integers.

## When to Use

Use this algorithm for one bounded in-memory 0/1 subset-sum decision when every
value and the target are non-negative integers. It is especially useful when
the target is moderate and a Python integer's compact bit operations are more
appropriate than a set containing many individual sums.

Use a predecessor table when the selected positions must be returned, or a
counting dynamic program when the number of solutions matters. Choose a
different formulation for negative values, reusable coins, approximate
optimization, or targets too large for a dense bitset.

## Implementation

```python
_MAX_INT64 = (1 << 63) - 1
_MAX_SUBSET_VALUES = 4_096
_MAX_SUBSET_TARGET = 100_000


def is_subset_sum_reachable(values: tuple[int, ...], target: int) -> bool:
    """Return whether a subset of input positions sums exactly to target."""
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    if type(target) is not int:
        raise TypeError("target must be an exact non-boolean integer")
    if len(values) > _MAX_SUBSET_VALUES:
        raise ValueError("value count exceeds the supported limit")
    if not 0 <= target <= _MAX_SUBSET_TARGET:
        raise ValueError("target is outside the supported range")

    for index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"values[{index}] must be an exact non-boolean integer")
        if not 0 <= value <= _MAX_INT64:
            raise ValueError(f"values[{index}] is outside the supported range")

    mask = (1 << (target + 1)) - 1
    reachable = 1
    for value in values:
        if value > target:
            continue
        reachable = (reachable | reachable << value) & mask

    return bool(reachable & (1 << target))
```

## Example

```python
def reachable_by_enumeration(values: tuple[int, ...], target: int) -> bool:
    from itertools import product

    return any(
        sum(value for value, selected in zip(values, choices, strict=True) if selected) == target
        for choices in product((False, True), repeat=len(values))
    )


def reachable_by_set_dp(values: tuple[int, ...], target: int) -> bool:
    reachable = {0}
    for value in values:
        reachable |= {partial + value for partial in tuple(reachable) if partial + value <= target}
    return target in reachable


def exercise_small_subset_sums() -> int:
    from itertools import product

    checked = 0
    for length in range(6):
        for values in product(range(4), repeat=length):
            for target in range(13):
                expected = reachable_by_enumeration(values, target)
                assert reachable_by_set_dp(values, target) is expected
                assert is_subset_sum_reachable(values, target) is expected
                checked += 1
    return checked


def results_for_every_order(values: tuple[int, ...], target: int) -> set[bool]:
    from itertools import permutations

    return {is_subset_sum_reachable(order, target) for order in permutations(values)}


checked_cases = exercise_small_subset_sums()
large_value_skipped = is_subset_sum_reachable((_MAX_INT64, 4, 5), 9)
maximum_target = is_subset_sum_reachable((40_000, 60_000, _MAX_INT64), _MAX_SUBSET_TARGET)
permutation_results = results_for_every_order((0, 2, 3, 7), 5)

try:
    is_subset_sum_reachable((1, True), 1)
except TypeError:
    boolean_rejected = True
else:
    boolean_rejected = False

try:
    is_subset_sum_reachable((0,) * (_MAX_SUBSET_VALUES + 1), 0)
except ValueError:
    count_cap_enforced = True
else:
    count_cap_enforced = False

assert (
    checked_cases,
    large_value_skipped,
    maximum_target,
    permutation_results,
    boolean_rejected,
    count_cap_enforced,
) == (17_745, True, True, {True}, True, True)
```

## Trade-offs and Limitations

Let `N` be the input count, `A` the number of values at most the target, and
`L = ceil((target + 1) / w)` for the Python integer implementation's limb width
`w`. Validation, mask construction, and updates require
`O(N + (A + 1) * L)` bounded limb work, or
`O(N + (N + 1) * L)` in the worst case. The persistent bitset uses
`O(target)` bits. A shifted temporary can briefly approach twice the target
width before the mask truncates it, but remains `O(target)` bits.

Every input position is considered at most once, so duplicate values are
separate choices rather than a reusable denomination. Zero values are valid
and do not change reachability. The empty subset makes target zero reachable,
including for empty input.

The result is only a Boolean decision. The function does not return a witness,
count solutions, minimize the number of selected positions, accept negative
values, solve unbounded coin change, or support an unbounded target.

## Related Snippets

<!-- catalog:related:start -->
- [Find the First Non-Empty Contiguous Integer Span with an Exact Sum](find-the-first-non-empty-contiguous-integer-span-with-an-exact-sum.md)
- [Build and Evaluate a Bounded Binary Assignment Constraint System](build-and-evaluate-a-bounded-binary-assignment-constraint-system.md)
- [Rank and Unrank Index Combinations in itertools.combinations Order](rank-and-unrank-index-combinations-in-itertools-combinations-order.md)
<!-- catalog:related:end -->
