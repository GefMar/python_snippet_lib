---
title: "Rank and Unrank Distinct Permutations of a Bounded Integer Multiset in Lexicographic Order"
snippet_type: algorithm
use_cases:
  - data-transformation
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - rank-and-unrank-index-permutations-in-itertools-permutations-order.md
  - return-the-next-lexicographic-permutation-of-bounded-integers.md
  - rank-and-unrank-index-combinations-in-itertools-combinations-order.md
---

# Rank and Unrank Distinct Permutations of a Bounded Integer Multiset in Lexicographic Order

## Idea and Problem

Map every distinct arrangement of a bounded integer multiset to its zero-based lexicographic rank and recover the arrangement at any valid rank.

At a position with `r` values remaining, suppose the current multiset has
`ways` distinct permutations. Choosing a candidate whose remaining
multiplicity is `c` leaves exactly `ways * c // r` suffix arrangements.
Ranking adds the blocks for smaller available candidates. Unranking subtracts
those same blocks until the requested rank enters one of them. Repeated equal
values therefore share one block instead of being treated as distinct indexes.

## When to Use

Use this bounded space when repeated integer values need stable identifiers,
direct checkpoint restoration, or sparse access in ordinary tuple
lexicographic order. Declare the multiset once; its input order is irrelevant,
while each ranked permutation must contain exactly the declared
multiplicities.

Use index-permutation ranking when equal-looking positions must remain
distinguishable. Use the next-permutation algorithm for sequential traversal,
and materialize deduplicated permutations only for tiny tests where the full
factorial search space is intentionally affordable.

## Implementation

```python
from bisect import bisect_left
from collections import Counter
from dataclasses import dataclass
from math import factorial, prod

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_MULTISET_SIZE = 256


@dataclass(frozen=True, slots=True, init=False)
class MultisetPermutationSpace:
    symbols: tuple[int, ...]
    multiplicities: tuple[int, ...]
    size: int
    permutation_count: int

    def __init__(self, values: tuple[int, ...]) -> None:
        if type(values) is not tuple:
            raise TypeError("values must be an exact tuple")
        if len(values) > _MAX_MULTISET_SIZE:
            raise ValueError("value count exceeds the supported limit")

        for index, value in enumerate(values):
            if type(value) is not int:
                raise TypeError(f"values[{index}] must be an exact non-boolean integer")
            if not _MIN_INT64 <= value <= _MAX_INT64:
                raise ValueError(f"values[{index}] is outside the signed 64-bit range")

        counts = Counter(values)
        symbols = tuple(sorted(counts))
        multiplicities = tuple(counts[symbol] for symbol in symbols)
        permutation_count = factorial(len(values)) // prod(
            factorial(count) for count in multiplicities
        )
        object.__setattr__(self, "symbols", symbols)
        object.__setattr__(self, "multiplicities", multiplicities)
        object.__setattr__(self, "size", len(values))
        object.__setattr__(self, "permutation_count", permutation_count)

    def rank(self, permutation: tuple[int, ...]) -> int:
        """Return one declared multiset permutation's zero-based rank."""
        if type(permutation) is not tuple:
            raise TypeError("permutation must be an exact tuple")
        if len(permutation) != self.size:
            raise ValueError("permutation length differs from the declared multiset")

        remaining = list(self.multiplicities)
        remaining_slots = self.size
        ways = self.permutation_count
        result = 0
        for position, value in enumerate(permutation):
            if type(value) is not int:
                raise TypeError(
                    f"permutation[{position}] must be an exact non-boolean integer"
                )
            if not _MIN_INT64 <= value <= _MAX_INT64:
                raise ValueError(
                    f"permutation[{position}] is outside the signed 64-bit range"
                )

            symbol_index = bisect_left(self.symbols, value)
            if (
                symbol_index == len(self.symbols)
                or self.symbols[symbol_index] != value
                or remaining[symbol_index] == 0
            ):
                raise ValueError("permutation does not match the declared multiset")

            for candidate_index in range(symbol_index):
                candidate_count = remaining[candidate_index]
                result += ways * candidate_count // remaining_slots

            chosen_count = remaining[symbol_index]
            ways = ways * chosen_count // remaining_slots
            remaining[symbol_index] -= 1
            remaining_slots -= 1
        return result

    def unrank(self, rank: int) -> tuple[int, ...]:
        """Return the distinct lexicographic permutation at a zero-based rank."""
        if type(rank) is not int:
            raise TypeError("rank must be an exact non-boolean integer")
        if not 0 <= rank < self.permutation_count:
            raise ValueError("rank is outside the distinct permutation space")

        remaining = list(self.multiplicities)
        remaining_slots = self.size
        ways = self.permutation_count
        remaining_rank = rank
        result: list[int] = []
        while remaining_slots:
            for symbol_index, candidate_count in enumerate(remaining):
                if candidate_count == 0:
                    continue
                candidate_ways = ways * candidate_count // remaining_slots
                if remaining_rank >= candidate_ways:
                    remaining_rank -= candidate_ways
                    continue

                result.append(self.symbols[symbol_index])
                remaining[symbol_index] -= 1
                remaining_slots -= 1
                ways = candidate_ways
                break
            else:
                raise RuntimeError("multiset unranking invariant failed")
        return tuple(result)
```

## Example

```python
def exercise_tiny_multisets() -> tuple[int, int]:
    from itertools import combinations_with_replacement, permutations

    space_count = 0
    permutation_count = 0
    for length in range(7):
        for sorted_values in combinations_with_replacement((-1, 0, 1), length):
            expected = tuple(sorted(set(permutations(sorted_values))))
            declaration = tuple(reversed(sorted_values))
            space = MultisetPermutationSpace(declaration)
            assert space.permutation_count == len(expected)
            for expected_rank, permutation in enumerate(expected):
                assert space.rank(permutation) == expected_rank
                assert space.unrank(expected_rank) == permutation
                permutation_count += 1
            space_count += 1
    return space_count, permutation_count


duplicates = MultisetPermutationSpace((2, 1, 1))
maximum_values = tuple(range(_MAX_MULTISET_SIZE))
maximum = MultisetPermutationSpace(tuple(reversed(maximum_values)))

rejected = 0
invalid_actions = (
    lambda: MultisetPermutationSpace((True,)),
    lambda: MultisetPermutationSpace((_MAX_INT64 + 1,)),
    lambda: MultisetPermutationSpace((0,) * (_MAX_MULTISET_SIZE + 1)),
    lambda: duplicates.rank((1, 2, 2)),
    lambda: duplicates.unrank(-1),
    lambda: duplicates.unrank(duplicates.permutation_count),
    lambda: duplicates.unrank(True),
)
for invalid_action in invalid_actions:
    try:
        invalid_action()
    except (TypeError, ValueError):
        rejected += 1

tiny_space_count, tiny_permutation_count = exercise_tiny_multisets()
last_rank = maximum.permutation_count - 1

assert (
    tiny_space_count,
    tiny_permutation_count,
    MultisetPermutationSpace(()).rank(()),
    MultisetPermutationSpace(()).unrank(0),
    duplicates.permutation_count,
    duplicates.rank((2, 1, 1)),
    duplicates.unrank(1),
    maximum.rank(tuple(reversed(maximum_values))),
    maximum.unrank(last_rank),
    rejected,
) == (
    84,
    1_093,
    0,
    (),
    3,
    2,
    (1, 2, 1),
    last_rank,
    tuple(reversed(maximum_values)),
    7,
)
```

## Trade-offs and Limitations

Building a space counts values in `O(n)` time, sorts its `k` distinct symbols
in `O(k log k)` time, and stores `O(k)` integers. Ranking and unranking each
inspect up to `k` candidates at every one of `n` positions, taking `O(nk)`
arithmetic operations and `O(k)` working memory, plus the `O(n)` returned tuple
for unranking.

Counts, block sizes, and ranks are exact Python integers. Up to 256 distinct
values produce `256!` permutations, so bigint multiplication, division, and
storage costs grow with rank width even though one result is reached without
enumerating earlier arrangements. The size bound makes the domain finite; it
does not make exhaustive traversal practical.

Lexicographic order is fixed to signed integer values. The implementation
rejects Booleans, out-of-range integers, wrong multiplicities, and invalid
ranks. It does not accept custom comparison orders, treat equal-valued indexes
as distinguishable, materialize the permutation space, or provide random
selection from an unbounded factorial domain.

## Related Snippets

<!-- catalog:related:start -->
- [Rank and Unrank Index Permutations in itertools.permutations Order](rank-and-unrank-index-permutations-in-itertools-permutations-order.md)
- [Return the Next Lexicographic Permutation of Bounded Integers](return-the-next-lexicographic-permutation-of-bounded-integers.md)
- [Rank and Unrank Index Combinations in itertools.combinations Order](rank-and-unrank-index-combinations-in-itertools-combinations-order.md)
<!-- catalog:related:end -->
