---
title: "Find the Earliest Maximum-XOR Index Pair in Bounded Unsigned 64-Bit Integers with a Binary Trie"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - build-a-canonical-reduced-xor-basis-for-bounded-unsigned-integers.md
  - find-the-canonical-farthest-pair-of-bounded-integer-points-with-rotating-calipers.md
---

# Find the Earliest Maximum-XOR Index Pair in Bounded Unsigned 64-Bit Integers with a Binary Trie

## Idea and Problem

Choose two distinct indexes whose unsigned 64-bit values have the greatest XOR, resolving equal XOR results by the earliest index pair.

A binary trie stores the bits of values at earlier indexes. For each later
value, a query greedily follows the opposite bit whenever that branch exists,
which maximizes the XOR from the most significant bit downward. Querying
before insertion guarantees distinct indexes and `first_index < second_index`.

XOR with one fixed value maps each possible partner value to a unique result.
Only duplicate partner values can tie within one query, so retaining the
earliest index at each leaf is sufficient. Comparing complete pairs
lexicographically resolves ties across different queries.

## When to Use

Use this function for a bounded materialized tuple of unsigned 64-bit integers
when the most bitwise-different pair is required and a quadratic all-pairs scan
would be wasteful. Index-based output preserves duplicate occurrences and
makes the tie rule directly usable in deterministic checks.

Use direct pair enumeration for tiny inputs or as a reference oracle. Use an
XOR basis when arbitrary subsets, rather than exactly two indexed values, may
be combined. Choose a mutable trie design when values must be inserted,
removed, or queried across a longer-lived data structure.

## Implementation

```python
_MAX_XOR_PAIR_VALUE_COUNT = 4_096
_MAX_UINT64 = (1 << 64) - 1
_MISSING_XOR_CHILD = -1

_MaximumXorPair = tuple[int, int, int]


def earliest_maximum_xor_index_pair(
    values: tuple[int, ...],
) -> _MaximumXorPair:
    """Return maximum XOR and the lexicographically earliest index pair."""
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    if not 2 <= len(values) <= _MAX_XOR_PAIR_VALUE_COUNT:
        raise ValueError("value count is outside 2..4096")
    for value_index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"values[{value_index}] must be an exact integer")
        if not 0 <= value <= _MAX_UINT64:
            raise ValueError(f"values[{value_index}] is outside unsigned 64-bit")

    zero_children = [_MISSING_XOR_CHILD]
    one_children = [_MISSING_XOR_CHILD]
    earliest_leaf_indexes = [_MISSING_XOR_CHILD]

    def insert(value: int, value_index: int) -> None:
        node = 0
        for shift in range(63, -1, -1):
            bit = (value >> shift) & 1
            children = one_children if bit else zero_children
            next_node = children[node]
            if next_node == _MISSING_XOR_CHILD:
                next_node = len(zero_children)
                children[node] = next_node
                zero_children.append(_MISSING_XOR_CHILD)
                one_children.append(_MISSING_XOR_CHILD)
                earliest_leaf_indexes.append(_MISSING_XOR_CHILD)
            node = next_node
        if earliest_leaf_indexes[node] == _MISSING_XOR_CHILD:
            earliest_leaf_indexes[node] = value_index

    def best_prior(value: int) -> tuple[int, int]:
        node = 0
        xor_value = 0
        for shift in range(63, -1, -1):
            bit = (value >> shift) & 1
            preferred = zero_children[node] if bit else one_children[node]
            if preferred != _MISSING_XOR_CHILD:
                node = preferred
                xor_value |= 1 << shift
            else:
                node = one_children[node] if bit else zero_children[node]
        return xor_value, earliest_leaf_indexes[node]

    insert(values[0], 0)
    best_xor = -1
    best_indexes = (0, 1)

    for second_index in range(1, len(values)):
        xor_value, first_index = best_prior(values[second_index])
        indexes = (first_index, second_index)
        if xor_value > best_xor or (
            xor_value == best_xor and indexes < best_indexes
        ):
            best_xor = xor_value
            best_indexes = indexes
        insert(values[second_index], second_index)

    return best_xor, best_indexes[0], best_indexes[1]
```

## Example

```python
def maximum_xor_pair_by_search(values: tuple[int, ...]) -> _MaximumXorPair:
    best_xor = -1
    best_indexes = (0, 1)
    for first_index in range(len(values)):
        for second_index in range(first_index + 1, len(values)):
            xor_value = values[first_index] ^ values[second_index]
            indexes = (first_index, second_index)
            if xor_value > best_xor or (
                xor_value == best_xor and indexes < best_indexes
            ):
                best_xor = xor_value
                best_indexes = indexes
    return best_xor, best_indexes[0], best_indexes[1]


def exercise_duplicate_heavy_tuples() -> int:
    from itertools import product

    checked = 0
    for length in range(2, 9):
        for values in product((0, 1, 3), repeat=length):
            assert earliest_maximum_xor_index_pair(
                values
            ) == maximum_xor_pair_by_search(values)
            checked += 1
    return checked


def exercise_seeded_random_tuples() -> int:
    from random import Random

    generator = Random(20_260_730)
    checked = 0
    for _ in range(500):
        values = tuple(
            generator.getrandbits(64)
            for _ in range(generator.randint(2, 48))
        )
        assert earliest_maximum_xor_index_pair(
            values
        ) == maximum_xor_pair_by_search(values)
        checked += 1
    return checked


all_equal = (17,) * _MAX_XOR_PAIR_VALUE_COUNT
unsigned_boundaries = (0, 1 << 63, _MAX_UINT64, 1)

rejected = 0
for invalid_values in (
    [0, 1],
    (0,),
    (0,) * (_MAX_XOR_PAIR_VALUE_COUNT + 1),
    (0, True),
    (-1, 0),
    (0, _MAX_UINT64 + 1),
):
    try:
        earliest_maximum_xor_index_pair(invalid_values)
    except (TypeError, ValueError):
        rejected += 1

assert (
    exercise_duplicate_heavy_tuples(),
    exercise_seeded_random_tuples(),
    earliest_maximum_xor_index_pair((5, 5, 2)),
    earliest_maximum_xor_index_pair((0, 7, 7, 0)),
    earliest_maximum_xor_index_pair(all_equal),
    earliest_maximum_xor_index_pair(unsigned_boundaries),
    rejected,
) == (
    9_837,
    500,
    (7, 0, 2),
    (7, 0, 1),
    (0, 0, 1),
    (_MAX_UINT64, 0, 2),
    6,
)
```

## Trade-offs and Limitations

Every insertion and query walks exactly 64 trie levels. There is one insertion
per value and one query for each value after the first, so total time is
`O(64 * N)`. The root plus at most 64 newly allocated nodes per value gives a
maximum of `1 + 64 * N` nodes across the three integer arrays.

The return order is `(xor_value, first_index, second_index)`. Indexes identify
distinct tuple positions even when their values are equal. Equal maximum XORs
choose the lexicographically smallest index pair, and retaining only the first
index at a duplicate-value leaf is sufficient for that rule.

The input contract rejects signed values, booleans, and integers wider than 64
bits. The function does not optimize XOR over arbitrary subsets or more than
two values, enumerate every maximizing pair, accept custom bit widths, expose
the trie, or support insertions and removals after the call.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Canonical Reduced XOR Basis for Bounded Unsigned Integers](build-a-canonical-reduced-xor-basis-for-bounded-unsigned-integers.md)
- [Find the Canonical Farthest Pair of Bounded Integer Points with Rotating Calipers](find-the-canonical-farthest-pair-of-bounded-integer-points-with-rotating-calipers.md)
<!-- catalog:related:end -->
