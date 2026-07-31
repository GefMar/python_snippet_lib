---
title: "Find a Canonical Signed Subset-Sum Witness with Meet-in-the-Middle"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - decide-bounded-non-negative-subset-sum-reachability-with-an-integer-bitset.md
  - solve-a-bounded-zero-one-knapsack-with-canonical-item-ties.md
  - rank-and-unrank-index-combinations-in-itertools-combinations-order.md
---

# Find a Canonical Signed Subset-Sum Witness with Meet-in-the-Middle

## Idea and Problem

Find one subset of signed integers whose exact sum equals a target, minimizing first the selected count and then the ascending index tuple.

Meet-in-the-middle divides the positions into two contiguous halves and
enumerates every subset sum in each half. For every right-half sum, retain only
its canonical mask. Each left-half mask then needs one complementary right sum.
This replaces enumeration of up to `2**N` complete subsets with two tables whose
largest size is `2**ceil(N / 2)`.

## When to Use

Use this for one exact signed subset-sum query when the item count is small
enough for exponential work but the target magnitude makes dense dynamic
programming unsuitable. It is useful when negative values must be accepted and
a reproducible witness matters more than processing a large number of items.

Use an integer bitset for a Boolean-only query over a moderate non-negative
target. Use predecessor dynamic programming when the reachable sum range is
small, or an optimization solver when additional constraints must be combined.
A direct exhaustive search is simpler for substantially smaller inputs.

## Implementation

```python
_MAX_SIGNED_SUBSET_ITEMS = 36
_MIN_SIGNED_SUBSET_VALUE = -(1 << 63)
_MAX_SIGNED_SUBSET_VALUE = (1 << 63) - 1


def _lexicographically_earlier_mask(candidate: int, incumbent: int) -> bool:
    """Compare equal-cardinality subsets by their ascending index tuples."""
    differing = candidate ^ incumbent
    if differing == 0:
        return False
    first_differing_bit = differing & -differing
    return bool(candidate & first_differing_bit)


def find_canonical_signed_subset_sum(
    values: tuple[int, ...],
    target: int,
) -> tuple[int, ...] | None:
    """Return the canonical position subset whose values sum to target."""
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    if len(values) > _MAX_SIGNED_SUBSET_ITEMS:
        raise ValueError("values contains more than 36 items")
    for index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"values[{index}] must be an exact integer")
        if not _MIN_SIGNED_SUBSET_VALUE <= value <= _MAX_SIGNED_SUBSET_VALUE:
            raise ValueError(f"values[{index}] is outside the signed 64-bit range")

    if type(target) is not int:
        raise TypeError("target must be an exact integer")
    if not _MIN_SIGNED_SUBSET_VALUE <= target <= _MAX_SIGNED_SUBSET_VALUE:
        raise ValueError("target is outside the signed 64-bit range")
    if target == 0:
        return ()

    split = len(values) // 2
    right_width = len(values) - split
    right_sums = [0] * (1 << right_width)
    best_right_by_sum: dict[int, tuple[int, int]] = {0: (0, 0)}

    for mask in range(1, 1 << right_width):
        lowest_bit = mask & -mask
        previous = mask ^ lowest_bit
        local_index = lowest_bit.bit_length() - 1
        subset_sum = right_sums[previous] + values[split + local_index]
        right_sums[mask] = subset_sum

        cardinality = mask.bit_count()
        incumbent = best_right_by_sum.get(subset_sum)
        if (
            incumbent is None
            or cardinality < incumbent[0]
            or (cardinality == incumbent[0] and _lexicographically_earlier_mask(mask, incumbent[1]))
        ):
            best_right_by_sum[subset_sum] = (cardinality, mask)

    left_sums = [0] * (1 << split)
    best_count = len(values) + 1
    best_mask = 0
    found = False
    for left_mask in range(1 << split):
        if left_mask:
            lowest_bit = left_mask & -left_mask
            previous = left_mask ^ lowest_bit
            local_index = lowest_bit.bit_length() - 1
            left_sums[left_mask] = left_sums[previous] + values[local_index]

        right_choice = best_right_by_sum.get(target - left_sums[left_mask])
        if right_choice is None:
            continue
        right_count, right_mask = right_choice
        candidate_count = left_mask.bit_count() + right_count
        candidate_mask = left_mask | (right_mask << split)
        if (
            not found
            or candidate_count < best_count
            or (
                candidate_count == best_count
                and _lexicographically_earlier_mask(candidate_mask, best_mask)
            )
        ):
            best_count = candidate_count
            best_mask = candidate_mask
            found = True

    if not found:
        return None
    return tuple(index for index in range(len(values)) if best_mask & (1 << index))
```

## Example

```python
from itertools import combinations, product
from random import Random


def brute_canonical_subset_sum(
    values: tuple[int, ...],
    target: int,
) -> tuple[int, ...] | None:
    indexes = range(len(values))
    for cardinality in range(len(values) + 1):
        for selected in combinations(indexes, cardinality):
            if sum(values[index] for index in selected) == target:
                return selected
    return None


exhaustive_checked = 0
for size in range(8):
    for small_values in product((-1, 0, 1), repeat=size):
        for small_target in range(-size - 1, size + 2):
            assert find_canonical_signed_subset_sum(
                small_values,
                small_target,
            ) == brute_canonical_subset_sum(small_values, small_target)
            exhaustive_checked += 1

rng = Random(0)
random_checked = 0
for _ in range(120):
    size = rng.randrange(15)
    random_values = tuple(rng.randrange(-20, 21) for _ in range(size))
    random_target = rng.randrange(-100, 101)
    assert find_canonical_signed_subset_sum(
        random_values,
        random_target,
    ) == brute_canonical_subset_sum(random_values, random_target)
    random_checked += 1

canonical_cases = (
    ((), 0, ()),
    ((), 1, None),
    ((0, 0, 0), 0, ()),
    ((1, 1, 2), 2, (2,)),
    ((5, 4, 3, 2), 7, (0, 3)),
    ((-5, 2, 3), -3, (0, 1)),
    ((2, 4, 8), 7, None),
)
for case_values, case_target, expected in canonical_cases:
    assert find_canonical_signed_subset_sum(case_values, case_target) == expected

minimum = _MIN_SIGNED_SUBSET_VALUE
maximum = _MAX_SIGNED_SUBSET_VALUE
boundary_answers = (
    find_canonical_signed_subset_sum((minimum, maximum, 1), minimum),
    find_canonical_signed_subset_sum((minimum, maximum, 1), maximum),
    find_canonical_signed_subset_sum((maximum, maximum, minimum), maximum - 1),
)

maximum_values = tuple(1 << index for index in range(_MAX_SIGNED_SUBSET_ITEMS))
maximum_target = (1 << 35) + (1 << 17) + 1
maximum_answer = find_canonical_signed_subset_sum(maximum_values, maximum_target)


def rejects(values: object, target: object) -> bool:
    try:
        find_canonical_signed_subset_sum(values, target)  # type: ignore[arg-type]
    except (TypeError, ValueError):
        return True
    return False


assert (
    exhaustive_checked,
    random_checked,
    boundary_answers,
    maximum_answer,
    rejects([], 0),
    rejects((1, True), 1),
    rejects((minimum - 1,), 0),
    rejects((maximum + 1,), 0),
    rejects((0,) * (_MAX_SIGNED_SUBSET_ITEMS + 1), 0),
    rejects((), True),
    rejects((), minimum - 1),
    rejects((), maximum + 1),
) == (
    52_488,
    120,
    ((0,), (1,), (0, 1, 2)),
    (0, 17, 35),
    True,
    True,
    True,
    True,
    True,
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

For `N` values, the two halves contain at most `ceil(N / 2)` positions.
Enumeration therefore takes `O(2**ceil(N / 2))` expected time and memory,
assuming expected constant-time integer dictionary operations. The final index
tuple takes `O(N)` additional time and output space. The implementation's
36-item cap keeps the exponential table explicit and bounded.

Declared values and the target must fit signed 64-bit range, but Python integer
subset sums remain exact and may exceed it internally. Equal values and zeros
remain distinct choices because results identify positions. Target zero always
returns the empty tuple, which is the unique minimum-cardinality witness.

The right table discards non-canonical witnesses for the same sum. The function
does not count or enumerate all solutions, expose alternative tie policies,
approximate a target, reuse an item, handle an unbounded item count, cache work
between calls, or use a pseudo-polynomial bitset.

## Related Snippets

<!-- catalog:related:start -->
- [Decide Bounded Non-Negative Subset-Sum Reachability with an Integer Bitset](decide-bounded-non-negative-subset-sum-reachability-with-an-integer-bitset.md)
- [Solve a Bounded Zero-One Knapsack with Canonical Item Ties](solve-a-bounded-zero-one-knapsack-with-canonical-item-ties.md)
- [Rank and Unrank Index Combinations in itertools.combinations Order](rank-and-unrank-index-combinations-in-itertools-combinations-order.md)
<!-- catalog:related:end -->
