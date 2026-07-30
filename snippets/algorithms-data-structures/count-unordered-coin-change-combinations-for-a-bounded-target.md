---
title: "Count Unordered Coin-Change Combinations for a Bounded Target"
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
  - solve-a-bounded-zero-one-knapsack-with-canonical-item-ties.md
  - apportion-a-non-negative-integer-total-without-rounding-drift.md
---

# Count Unordered Coin-Change Combinations for a Bounded Target

## Idea and Problem

Count how many multisets of reusable positive denominations sum exactly to one bounded non-negative target.

A one-dimensional dynamic program stores the number of combinations reaching
each subtotal. Processing one denomination at a time makes denomination choice,
rather than coin position, the outer decision. Scanning subtotals upward permits
the current denomination to be reused while counting each multiset once instead
of counting every ordering of the same coins.

## When to Use

Use this algorithm when denominations are distinct, every denomination may be
used any non-negative number of times, and only the exact combination count is
needed. It fits small planning, fixture, and integer-partition-style problems
whose target is low enough for a dense table.

Use a different dynamic program when every input position may be selected only
once, coin order matters, or a minimum-size witness is required. Avoid a dense
target-indexed table when the target is large or the reachable sums are sparse.

## Implementation

```python
_MAX_DENOMINATION_COUNT = 32
_MAX_DENOMINATION = 1_000_000
_MAX_CHANGE_TARGET = 10_000


def count_unordered_coin_change_combinations(
    denominations: tuple[int, ...],
    target: int,
) -> int:
    """Return the number of denomination multisets summing to target."""
    if type(denominations) is not tuple:
        raise TypeError("denominations must be an exact tuple")
    if len(denominations) > _MAX_DENOMINATION_COUNT:
        raise ValueError("denomination count exceeds the supported limit")
    if type(target) is not int:
        raise TypeError("target must be an exact non-boolean integer")
    if not 0 <= target <= _MAX_CHANGE_TARGET:
        raise ValueError("target is outside the supported range")

    seen: set[int] = set()
    for index, denomination in enumerate(denominations):
        if type(denomination) is not int:
            raise TypeError(f"denominations[{index}] must be an exact non-boolean integer")
        if not 1 <= denomination <= _MAX_DENOMINATION:
            raise ValueError(f"denominations[{index}] is outside the supported range")
        if denomination in seen:
            raise ValueError("denominations must be distinct")
        seen.add(denomination)

    combinations = [0] * (target + 1)
    combinations[0] = 1
    for denomination in sorted(denominations):
        for subtotal in range(denomination, target + 1):
            combinations[subtotal] += combinations[subtotal - denomination]

    return combinations[target]
```

## Example

```python
def count_by_enumeration(
    denominations: tuple[int, ...],
    target: int,
    denomination_index: int = 0,
) -> int:
    if denomination_index == len(denominations):
        return int(target == 0)
    denomination = denominations[denomination_index]
    return sum(
        count_by_enumeration(
            denominations,
            target - used_count * denomination,
            denomination_index + 1,
        )
        for used_count in range(target // denomination + 1)
    )


def exercise_small_targets() -> int:
    from itertools import combinations

    checked = 0
    for denomination_count in range(5):
        for denominations in combinations(range(1, 7), denomination_count):
            for target in range(13):
                expected = count_by_enumeration(denominations, target)
                assert count_unordered_coin_change_combinations(denominations, target) == expected
                assert (
                    count_unordered_coin_change_combinations(tuple(reversed(denominations)), target)
                    == expected
                )
                checked += 1
    return checked


checked_cases = exercise_small_targets()
ordinary = count_unordered_coin_change_combinations((1, 2, 5), 5)
empty_cases = (
    count_unordered_coin_change_combinations((), 0),
    count_unordered_coin_change_combinations((), 7),
)
oversized_denominations_skipped = count_unordered_coin_change_combinations(
    (1, 20_000, 1_000_000), 4
)
maximum_target_count = count_unordered_coin_change_combinations(
    tuple(range(1, _MAX_DENOMINATION_COUNT + 1)),
    _MAX_CHANGE_TARGET,
)

rejected = 0
invalid_calls = (
    lambda: count_unordered_coin_change_combinations([1, 2], 3),
    lambda: count_unordered_coin_change_combinations((True,), 1),
    lambda: count_unordered_coin_change_combinations((0,), 1),
    lambda: count_unordered_coin_change_combinations((1, 1), 2),
    lambda: count_unordered_coin_change_combinations((_MAX_DENOMINATION + 1,), 1),
    lambda: count_unordered_coin_change_combinations(
        tuple(range(1, _MAX_DENOMINATION_COUNT + 2)), 1
    ),
    lambda: count_unordered_coin_change_combinations((1,), True),
    lambda: count_unordered_coin_change_combinations((1,), -1),
    lambda: count_unordered_coin_change_combinations((1,), _MAX_CHANGE_TARGET + 1),
)
for invalid_call in invalid_calls:
    try:
        invalid_call()
    except (TypeError, ValueError):
        rejected += 1

assert (
    checked_cases,
    ordinary,
    empty_cases,
    oversized_denominations_skipped,
    type(maximum_target_count) is int and maximum_target_count > 0,
    rejected,
) == (741, 4, (1, 0), 1, True, 9)
```

## Trade-offs and Limitations

For `D` denominations and target `T`, sorting takes `O(D * log(D))`
comparisons and the table performs at most `D * T` updates. The sorted list
uses `O(D)` references and the table stores `T + 1` Python integers. Additions
are not constant-cost when counts grow, so time and memory also depend on the
integers' bit lengths.

The function accepts at most 32 distinct exact positive denominations no larger
than 1,000,000 and a target from zero through 10,000. Denominations above the
target are valid but contribute no updates. Input denomination order cannot
change the count. The one combination for target zero is the empty multiset.

Only the count is returned. The function does not enumerate combinations,
select a witness, minimize coin count, distinguish equal-valued coin identities,
accept duplicate or non-positive denominations, or support bounded supplies.

## Related Snippets

<!-- catalog:related:start -->
- [Decide Bounded Non-Negative Subset-Sum Reachability with an Integer Bitset](decide-bounded-non-negative-subset-sum-reachability-with-an-integer-bitset.md)
- [Solve a Bounded Zero-One Knapsack with Canonical Item Ties](solve-a-bounded-zero-one-knapsack-with-canonical-item-ties.md)
- [Apportion a Non-Negative Integer Total Without Rounding Drift](apportion-a-non-negative-integer-total-without-rounding-drift.md)
<!-- catalog:related:end -->
