---
title: "Count Index-Distinct Longest Strictly Increasing Subsequences of Bounded Integers"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - find-a-longest-strictly-increasing-integer-subsequence-with-earliest-index-ties.md
  - count-strict-inversions-in-a-bounded-integer-sequence.md
  - maintain-bounded-point-adds-and-half-open-range-sums-with-a-fenwick-tree.md
---

# Count Index-Distinct Longest Strictly Increasing Subsequences of Bounded Integers

## Idea and Problem

Return the maximum length and exact number of index-distinct strictly increasing subsequences in one bounded integer tuple.

Coordinate compression gives every distinct value an ascending rank. A
Fenwick tree stores the best `(length, count)` pair for each prefix of those
ranks. Querying ranks strictly below the current value finds the longest
subsequences that this position may extend; equal best lengths contribute
their counts because they represent different index choices.

## When to Use

Use this algorithm when duplicate values may occur, skipped positions are
allowed, and different tuples of source indexes must count as different
subsequences even when their value tuples are equal. It avoids enumerating an
answer set whose size can be exponential.

Use the canonical-witness snippet when one reproducible subsequence is needed
instead of the number of optima. Choose a non-decreasing variant when equal
successive values are admissible, or a quadratic dynamic program when the
input is small and direct recurrence code is preferable to coordinate
compression.

## Implementation

```python
_MIN_LIS_INTEGER = -(1 << 63)
_MAX_LIS_INTEGER = (1 << 63) - 1
_MAX_LIS_VALUE_COUNT = 100_000


def count_longest_strictly_increasing_subsequences(
    values: tuple[int, ...],
) -> tuple[int, int]:
    """Return the longest length and number of index-distinct witnesses."""
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    if len(values) > _MAX_LIS_VALUE_COUNT:
        raise ValueError("value count exceeds the supported limit")
    for index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"values[{index}] must be an exact integer")
        if not _MIN_LIS_INTEGER <= value <= _MAX_LIS_INTEGER:
            raise ValueError(f"values[{index}] is outside the signed 64-bit range")

    if not values:
        return 0, 1

    ordered_values = sorted(set(values))
    ranks = {value: index + 1 for index, value in enumerate(ordered_values)}
    tree_lengths = [0] * (len(ordered_values) + 1)
    tree_counts = [0] * (len(ordered_values) + 1)

    def prefix_best(stop: int) -> tuple[int, int]:
        best_length = 0
        best_count = 0
        while stop:
            candidate_length = tree_lengths[stop]
            if candidate_length > best_length:
                best_length = candidate_length
                best_count = tree_counts[stop]
            elif candidate_length == best_length and candidate_length:
                best_count += tree_counts[stop]
            stop -= stop & -stop
        return best_length, best_count

    for value in values:
        rank = ranks[value]
        previous_length, previous_count = prefix_best(rank - 1)
        current_length = previous_length + 1
        current_count = previous_count if previous_length else 1

        position = rank
        while position < len(tree_lengths):
            if current_length > tree_lengths[position]:
                tree_lengths[position] = current_length
                tree_counts[position] = current_count
            elif current_length == tree_lengths[position]:
                tree_counts[position] += current_count
            position += position & -position

    return prefix_best(len(ordered_values))
```

## Example

```python
def count_longest_by_subsets(values: tuple[int, ...]) -> tuple[int, int]:
    if not values:
        return 0, 1

    best_length = 0
    best_count = 0
    for selected_bits in range(1, 1 << len(values)):
        selected = tuple(
            values[index]
            for index in range(len(values))
            if selected_bits & (1 << index)
        )
        if any(
            selected[index - 1] >= selected[index]
            for index in range(1, len(selected))
        ):
            continue
        if len(selected) > best_length:
            best_length = len(selected)
            best_count = 1
        elif len(selected) == best_length:
            best_count += 1
    return best_length, best_count


def exercise_short_sequences() -> int:
    from itertools import product

    checked = 0
    for length in range(8):
        for values in product((-1, 0, 2), repeat=length):
            assert count_longest_strictly_increasing_subsequences(
                values
            ) == count_longest_by_subsets(values)
            checked += 1
    return checked


large_count_values = tuple(value for value in range(64) for _ in range(2))
maximum_equal = count_longest_strictly_increasing_subsequences(
    (0,) * _MAX_LIS_VALUE_COUNT
)

rejected = 0
for invalid_values in (
    [1, 2],
    (True,),
    (_MIN_LIS_INTEGER - 1,),
    (0,) * (_MAX_LIS_VALUE_COUNT + 1),
):
    try:
        count_longest_strictly_increasing_subsequences(invalid_values)
    except (TypeError, ValueError):
        rejected += 1

assert (
    exercise_short_sequences(),
    count_longest_strictly_increasing_subsequences(()),
    count_longest_strictly_increasing_subsequences((1, 3, 5, 4, 7)),
    count_longest_strictly_increasing_subsequences((2, 2, 2)),
    count_longest_strictly_increasing_subsequences((4, 3, 2, 1)),
    count_longest_strictly_increasing_subsequences(large_count_values),
    maximum_equal,
    rejected,
) == (
    3_280,
    (0, 1),
    (4, 2),
    (1, 3),
    (1, 4),
    (64, 1 << 64),
    (1, _MAX_LIS_VALUE_COUNT),
    4,
)
```

## Trade-offs and Limitations

For `N` values and `U` distinct values, coordinate compression takes
`O(N log N)` comparison time and each Fenwick query and update takes
`O(log U)` tree operations. The ordered values, rank mapping, and two Fenwick
arrays use `O(N + U)` slots.

Counts are exact Python integers and can be exponentially large. Their
additions and retained storage are therefore not constant-cost even though the
number of Fenwick operations is `O(N log U)`. Equal values cannot extend one
another, but occurrences at different indexes contribute separate witnesses.
The empty input's one empty index tuple is its unique optimum, so it returns
`(0, 1)`.

The function returns no indexes or values from a witness and does not
enumerate optimal subsequences. It does not count distinct value tuples,
accept custom comparison keys, admit equal successive values, process a
stream, support updates, or provide fixed-width overflow behavior.

## Related Snippets

<!-- catalog:related:start -->
- [Find a Longest Strictly Increasing Integer Subsequence with Earliest-Index Ties](find-a-longest-strictly-increasing-integer-subsequence-with-earliest-index-ties.md)
- [Count Strict Inversions in a Bounded Integer Sequence](count-strict-inversions-in-a-bounded-integer-sequence.md)
- [Maintain Bounded Point Adds and Half-Open Range Sums with a Fenwick Tree](maintain-bounded-point-adds-and-half-open-range-sums-with-a-fenwick-tree.md)
<!-- catalog:related:end -->
