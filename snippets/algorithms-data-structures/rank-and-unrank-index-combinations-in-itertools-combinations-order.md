---
title: "Rank and Unrank Index Combinations in itertools.combinations Order"
snippet_type: algorithm
use_cases:
  - data-transformation
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../testing-tooling/audit-a-bounded-test-matrix-for-complete-pairwise-coverage.md
  - ../configuration-serialization/expand-a-bounded-plan-matrix-with-explicit-target-overrides.md
  - cover-a-half-open-integer-range-with-dyadic-intervals.md
---

# Rank and Unrank Index Combinations in itertools.combinations Order

## Idea and Problem

Map each fixed-size index combination to its zero-based lexicographic position and recover the combination at any valid position without enumerating earlier choices.

At each position, every smaller admissible index starts a contiguous block of
combinations. The size of that block is a binomial coefficient, so ranking adds
the skipped blocks and unranking subtracts them until the block containing the
requested rank is found. This is exactly the order produced by
`itertools.combinations(range(n), k)`.

## When to Use

Use these functions when combinations of positions need stable numeric IDs,
direct checkpoint restoration, or sparse access to a bounded combinatorial
space. The contract is useful when `n` and `k` are shared explicitly and the
combination values are already strictly increasing integer indexes.

Prefer ordinary `itertools.combinations` when every result will be consumed in
sequence. Ranking does not make a very large combination space practical to
enumerate, and unranking is not a substitute for a sampling policy.

## Implementation

```python
from math import comb

_MAX_COMBINATION_DOMAIN = 256


def _validate_combination_dimensions(n: int, k: int) -> None:
    if type(n) is not int:
        raise TypeError("n must be an exact non-boolean integer")
    if type(k) is not int:
        raise TypeError("k must be an exact non-boolean integer")
    if not 0 <= n <= _MAX_COMBINATION_DOMAIN:
        raise ValueError("n is outside the supported range")
    if not 0 <= k <= n:
        raise ValueError("k must be between zero and n")


def rank_combination(n: int, k: int, indices: tuple[int, ...]) -> int:
    """Return the zero-based itertools.combinations rank of indices."""
    _validate_combination_dimensions(n, k)
    if type(indices) is not tuple:
        raise TypeError("indices must be an exact tuple")
    if len(indices) != k:
        raise ValueError("indices must contain exactly k values")

    previous = -1
    for position, index in enumerate(indices):
        if type(index) is not int:
            raise TypeError(f"indices[{position}] must be an exact non-boolean integer")
        if not 0 <= index < n:
            raise ValueError(f"indices[{position}] is outside the index domain")
        if index <= previous:
            raise ValueError("indices must be strictly increasing")
        previous = index

    result = 0
    previous = -1
    for position, index in enumerate(indices):
        remaining = k - position - 1
        for skipped in range(previous + 1, index):
            result += comb(n - skipped - 1, remaining)
        previous = index
    return result


def unrank_combination(n: int, k: int, rank: int) -> tuple[int, ...]:
    """Return the index combination at one zero-based lexicographic rank."""
    _validate_combination_dimensions(n, k)
    if type(rank) is not int:
        raise TypeError("rank must be an exact non-boolean integer")

    combination_count = comb(n, k)
    if not 0 <= rank < combination_count:
        raise ValueError("rank is outside the combination space")

    remaining_rank = rank
    next_index = 0
    result: list[int] = []
    for position in range(k):
        remaining = k - position - 1
        last_candidate = n - remaining - 1
        for candidate in range(next_index, last_candidate + 1):
            block_size = comb(n - candidate - 1, remaining)
            if remaining_rank < block_size:
                result.append(candidate)
                next_index = candidate + 1
                break
            remaining_rank -= block_size

    return tuple(result)
```

## Example

```python
def exercise_small_combination_spaces() -> None:
    from itertools import combinations

    for small_n in range(9):
        for small_k in range(small_n + 1):
            expected = tuple(combinations(range(small_n), small_k))
            assert tuple(
                rank_combination(small_n, small_k, indices) for indices in expected
            ) == tuple(range(len(expected)))
            assert (
                tuple(unrank_combination(small_n, small_k, rank) for rank in range(len(expected)))
                == expected
            )


exercise_small_combination_spaces()

large_count = comb(256, 128)
large_last = tuple(range(128, 256))

try:
    unrank_combination(5, 3, True)
except TypeError:
    boolean_rank_rejected = True
else:
    boolean_rank_rejected = False

assert (
    rank_combination(5, 3, (0, 2, 4)),
    unrank_combination(5, 3, 4),
    rank_combination(0, 0, ()),
    unrank_combination(0, 0, 0),
    rank_combination(256, 128, large_last),
    unrank_combination(256, 128, large_count - 1),
    boolean_rank_rejected,
) == (4, (0, 2, 4), 0, (), large_count - 1, large_last, True)
```

## Trade-offs and Limitations

Each operation performs at most `O(n)` bounded `math.comb` evaluations.
Ranking uses `O(1)` auxiliary containers, while unranking uses `O(k)` output
memory. Binomial coefficients are exact Python integers rather than unit-cost
machine words; their arithmetic cost grows with their bit length, although the
`n <= 256` contract bounds every value involved.

The functions rank positions, not arbitrary pool values. They deliberately
reject repeated or unsorted indexes and do not support permutations,
multisets, colexicographic order, random selection, or mutation of `n` and `k`
between ranking and unranking. A rank can identify one combination compactly,
but the complete space can still be far too large to traverse or store.

## Related Snippets

<!-- catalog:related:start -->
- [Audit a Bounded Test Matrix for Complete Pairwise Coverage](../testing-tooling/audit-a-bounded-test-matrix-for-complete-pairwise-coverage.md)
- [Expand a Bounded Plan Matrix with Explicit Target Overrides](../configuration-serialization/expand-a-bounded-plan-matrix-with-explicit-target-overrides.md)
- [Cover a Half-Open Integer Range with Dyadic Intervals](cover-a-half-open-integer-range-with-dyadic-intervals.md)
<!-- catalog:related:end -->
