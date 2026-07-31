---
title: "Rank and Unrank Bounded Integer Partitions in Lexicographic Order"
snippet_type: algorithm
use_cases:
  - data-transformation
  - testing
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - rank-and-unrank-bounded-balanced-parentheses-strings-in-lexicographic-order.md
  - rank-and-unrank-index-combinations-in-itertools-combinations-order.md
  - rank-and-unrank-distinct-permutations-of-a-bounded-integer-multiset-in-lexicographic-order.md
---

# Rank and Unrank Bounded Integer Partitions in Lexicographic Order

## Idea and Problem

Map every non-increasing integer partition of one bounded total to its zero-based lexicographic rank and recover the partition at any valid rank.

A dynamic-programming table counts ways to finish a remaining total when the
next part cannot exceed a given maximum. Ranking adds the completion blocks for
every smaller possible part before consuming the actual one. Unranking
subtracts the same blocks until the requested rank identifies its next part.
Neither operation enumerates the partitions preceding the requested result.

## When to Use

Use these functions when partitions of a fixed integer need stable identifiers,
direct access by ordinal, or deterministic test-case selection. The order is
ordinary Python tuple order: the all-ones partition comes first and the
single-part tuple comes last.

Use a generator when every partition must be visited. Use combination or
permutation ranking when positions or arrangements, rather than unordered
positive summands, define the state space. A specialized combinatorics package
is more suitable for substantially larger totals or several partition orders.

## Implementation

```python
_MAX_PARTITION_TOTAL = 256


def _partition_completion_counts(total: int) -> list[list[int]]:
    counts = [[0] * (total + 1) for _ in range(total + 1)]
    for maximum_part in range(total + 1):
        counts[0][maximum_part] = 1

    for remaining in range(1, total + 1):
        for maximum_part in range(1, total + 1):
            counts[remaining][maximum_part] = counts[remaining][maximum_part - 1]
            if maximum_part <= remaining:
                counts[remaining][maximum_part] += counts[remaining - maximum_part][maximum_part]
    return counts


def rank_integer_partition(parts: tuple[int, ...]) -> int:
    """Return a non-increasing partition's zero-based tuple-order rank."""
    if type(parts) is not tuple:
        raise TypeError("parts must be an exact tuple")

    total = 0
    previous_part = _MAX_PARTITION_TOTAL
    for index, part in enumerate(parts):
        if type(part) is not int:
            raise TypeError(f"parts[{index}] must be an exact integer")
        if part <= 0:
            raise ValueError("partition parts must be positive")
        if part > previous_part:
            raise ValueError("partition parts must be non-increasing")
        total += part
        if total > _MAX_PARTITION_TOTAL:
            raise ValueError("partition total exceeds 256")
        previous_part = part

    counts = _partition_completion_counts(total)
    remaining = total
    maximum_part = total
    rank = 0
    for part in parts:
        for candidate in range(1, part):
            if candidate > maximum_part or candidate > remaining:
                break
            rank += counts[remaining - candidate][candidate]
        remaining -= part
        maximum_part = part
    return rank


def unrank_integer_partition(total: int, rank: int) -> tuple[int, ...]:
    """Return the partition at one zero-based tuple-order rank."""
    if type(total) is not int:
        raise TypeError("total must be an exact integer")
    if not 0 <= total <= _MAX_PARTITION_TOTAL:
        raise ValueError("total is outside 0..256")
    if type(rank) is not int:
        raise TypeError("rank must be an exact integer")

    counts = _partition_completion_counts(total)
    partition_count = counts[total][total]
    if not 0 <= rank < partition_count:
        raise ValueError("rank is outside the partition space")

    remaining = total
    maximum_part = total
    result: list[int] = []
    while remaining:
        for candidate in range(1, min(maximum_part, remaining) + 1):
            candidate_count = counts[remaining - candidate][candidate]
            if rank >= candidate_count:
                rank -= candidate_count
                continue
            result.append(candidate)
            remaining -= candidate
            maximum_part = candidate
            break
        else:
            raise RuntimeError("integer-partition unranking invariant failed")
    return tuple(result)
```

## Example

```python
def enumerate_partitions(total: int, maximum_part: int) -> tuple[tuple[int, ...], ...]:
    if total == 0:
        return ((),)
    found: list[tuple[int, ...]] = []
    for first_part in range(1, min(total, maximum_part) + 1):
        for suffix in enumerate_partitions(total - first_part, first_part):
            found.append((first_part, *suffix))
    return tuple(found)


exercised = 0
for small_total in range(19):
    expected = tuple(sorted(enumerate_partitions(small_total, small_total)))
    for expected_rank, partition in enumerate(expected):
        assert rank_integer_partition(partition) == expected_rank
        assert unrank_integer_partition(small_total, expected_rank) == partition
        exercised += 1

independent_counts = [1] + [0] * _MAX_PARTITION_TOTAL
for available_part in range(1, _MAX_PARTITION_TOTAL + 1):
    for subtotal in range(available_part, _MAX_PARTITION_TOTAL + 1):
        independent_counts[subtotal] += independent_counts[subtotal - available_part]

boundary_counts = _partition_completion_counts(_MAX_PARTITION_TOTAL)
boundary_partition_count = boundary_counts[-1][-1]
boundary_first = (1,) * _MAX_PARTITION_TOTAL
boundary_last = (_MAX_PARTITION_TOTAL,)
boundary_middle = unrank_integer_partition(
    _MAX_PARTITION_TOTAL,
    boundary_partition_count // 2,
)


class TupleSubclass(tuple):
    pass


rejected = 0
invalid_actions = (
    lambda: rank_integer_partition(TupleSubclass((1,))),
    lambda: rank_integer_partition((True,)),
    lambda: rank_integer_partition((2, 3)),
    lambda: rank_integer_partition((1, 0)),
    lambda: rank_integer_partition((257,)),
    lambda: unrank_integer_partition(True, 0),
    lambda: unrank_integer_partition(257, 0),
    lambda: unrank_integer_partition(4, True),
    lambda: unrank_integer_partition(4, -1),
    lambda: unrank_integer_partition(4, 5),
)
for invalid_action in invalid_actions:
    try:
        invalid_action()
    except (TypeError, ValueError):
        rejected += 1

assert (
    exercised > 0
    and rank_integer_partition(()) == 0
    and unrank_integer_partition(0, 0) == ()
    and independent_counts[-1] == boundary_partition_count
    and rank_integer_partition(boundary_first) == 0
    and rank_integer_partition(boundary_last) == boundary_partition_count - 1
    and unrank_integer_partition(_MAX_PARTITION_TOTAL, 0) == boundary_first
    and unrank_integer_partition(
        _MAX_PARTITION_TOTAL,
        boundary_partition_count - 1,
    )
    == boundary_last
    and rank_integer_partition(boundary_middle) == boundary_partition_count // 2
    and rejected == len(invalid_actions)
)
```

## Trade-offs and Limitations

For total `N`, table construction performs `O(N**2)` exact-integer additions
and stores `O(N**2)` integer references. Ranking and unranking inspect at most
`O(N**2)` candidate parts and return at most `N` parts. Partition counts grow
quickly, so integer bit width and arithmetic cost are not constant.

Parts must be positive and non-increasing, and ranks compare only partitions
of the same total. The empty tuple is the sole partition of zero. The
implementation rejects Booleans, tuple subclasses, malformed ordering and
out-of-range ranks rather than normalizing them.

The functions do not enumerate the whole space, rank partitions across mixed
totals, support ascending-part or dominance orders, impose a fixed part count,
sample randomly, or retain the count table between calls.

## Related Snippets

<!-- catalog:related:start -->
- [Rank and Unrank Bounded Balanced-Parentheses Strings in Lexicographic Order](rank-and-unrank-bounded-balanced-parentheses-strings-in-lexicographic-order.md)
- [Rank and Unrank Index Combinations in itertools.combinations Order](rank-and-unrank-index-combinations-in-itertools-combinations-order.md)
- [Rank and Unrank Distinct Permutations of a Bounded Integer Multiset in Lexicographic Order](rank-and-unrank-distinct-permutations-of-a-bounded-integer-multiset-in-lexicographic-order.md)
<!-- catalog:related:end -->
