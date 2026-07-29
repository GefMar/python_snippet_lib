---
title: "Rank and Unrank Index Permutations in itertools.permutations Order"
snippet_type: algorithm
use_cases:
  - data-transformation
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - rank-and-unrank-index-combinations-in-itertools-combinations-order.md
  - return-the-next-lexicographic-permutation-of-bounded-integers.md
  - count-strict-inversions-in-a-bounded-integer-sequence.md
---

# Rank and Unrank Index Permutations in itertools.permutations Order

## Idea and Problem

Map each bounded index permutation to its zero-based lexicographic position and recover the permutation at any valid position without enumerating earlier arrangements.

For a fixed prefix, every unused smaller index starts a block containing the
same number of suffix permutations. Those block sizes are successive
factorials, so the unused index's position is one Lehmer-code digit. Ranking
adds the skipped factorial blocks; unranking divides a rank into the same
factoradic digits and removes the selected indexes.

## When to Use

Use these functions when permutations of a small closed index set need stable
numeric identifiers, direct checkpoint restoration, or sparse access in the
exact order produced by `itertools.permutations(range(size))`. The ranked
tuple must contain every index exactly once, and the same size must be supplied
when that rank is decoded.

Prefer ordinary `itertools.permutations` when every arrangement will be
consumed sequentially. Use a multiset-specific algorithm when values repeat,
or carry a separate immutable pool when indexes must later resolve to
application values.

## Implementation

```python
from math import factorial

_MAX_PERMUTATION_SIZE = 256


def _validate_permutation_size(size: int) -> None:
    if type(size) is not int:
        raise TypeError("size must be an exact non-boolean integer")
    if not 0 <= size <= _MAX_PERMUTATION_SIZE:
        raise ValueError("size is outside the supported range")


def rank_index_permutation(permutation: tuple[int, ...]) -> int:
    """Return the zero-based itertools.permutations rank."""
    if type(permutation) is not tuple:
        raise TypeError("permutation must be an exact tuple")
    size = len(permutation)
    _validate_permutation_size(size)

    seen = [False] * size
    for position, index in enumerate(permutation):
        if type(index) is not int:
            raise TypeError(f"permutation[{position}] must be an exact non-boolean integer")
        if not 0 <= index < size:
            raise ValueError(f"permutation[{position}] is outside the index domain")
        if seen[index]:
            raise ValueError(f"permutation[{position}] duplicates an earlier index")
        seen[index] = True

    remaining = list(range(size))
    block_size = factorial(size - 1) if size else 1
    result = 0
    for position, index in enumerate(permutation):
        digit = remaining.index(index)
        result += digit * block_size
        remaining.pop(digit)

        slots_after = size - position - 1
        if slots_after:
            block_size //= slots_after
    return result


def unrank_index_permutation(size: int, rank: int) -> tuple[int, ...]:
    """Return the index permutation at one zero-based lexicographic rank."""
    _validate_permutation_size(size)
    if type(rank) is not int:
        raise TypeError("rank must be an exact non-boolean integer")

    permutation_count = factorial(size)
    if not 0 <= rank < permutation_count:
        raise ValueError("rank is outside the permutation space")

    remaining = list(range(size))
    remaining_rank = rank
    block_size = permutation_count // size if size else 1
    result: list[int] = []
    for slots in range(size, 0, -1):
        digit, remaining_rank = divmod(remaining_rank, block_size)
        result.append(remaining.pop(digit))
        if slots > 1:
            block_size //= slots - 1
    return tuple(result)
```

## Example

```python
def exercise_small_permutation_spaces() -> int:
    from itertools import permutations

    checked = 0
    for size in range(9):
        for expected_rank, permutation in enumerate(permutations(range(size))):
            assert rank_index_permutation(permutation) == expected_rank
            assert unrank_index_permutation(size, expected_rank) == permutation
            checked += 1
    return checked


maximum_size = _MAX_PERMUTATION_SIZE
permutation_count = factorial(maximum_size)
identity = tuple(range(maximum_size))
reverse = tuple(reversed(identity))
middle_rank = permutation_count // 2
middle_permutation = unrank_index_permutation(maximum_size, middle_rank)

value_errors = 0
for invalid_permutation in ((0, 2), (0, 0, 2)):
    try:
        rank_index_permutation(invalid_permutation)
    except ValueError:
        value_errors += 1

for invalid_size, invalid_rank in (
    (3, -1),
    (3, factorial(3)),
    (_MAX_PERMUTATION_SIZE + 1, 0),
):
    try:
        unrank_index_permutation(invalid_size, invalid_rank)
    except ValueError:
        value_errors += 1

type_errors = 0
try:
    rank_index_permutation((0, 1, True))
except TypeError:
    type_errors += 1

try:
    unrank_index_permutation(3, True)
except TypeError:
    type_errors += 1

assert (
    exercise_small_permutation_spaces(),
    rank_index_permutation(()),
    unrank_index_permutation(0, 0),
    rank_index_permutation(identity),
    rank_index_permutation(reverse),
    unrank_index_permutation(maximum_size, permutation_count - 1),
    rank_index_permutation(middle_permutation),
    value_errors,
    type_errors,
) == (
    46_234,
    0,
    (),
    0,
    permutation_count - 1,
    reverse,
    middle_rank,
    5,
    2,
)
```

## Trade-offs and Limitations

Validation uses `O(n)` time. Each function then performs up to `n` searches
or removals in a shrinking Python list, so its worst-case running time is
`O(n^2)`; the remaining-index list and returned permutation use `O(n)`
slots. The deliberate `n <= 256` cap keeps this direct implementation small
and avoids adding a Fenwick tree solely to improve the asymptotic bound.

Factorials and ranks are exact Python integers rather than unit-cost machine
words. A worst-case rank contains `Theta(n log n)` bits, and the cost of
factorial multiplication, division, `divmod`, and addition grows with operand
width. Direct access to one arrangement does not make the factorial-size space
practical to enumerate.

The functions rank positions, not arbitrary pool values. They reject missing,
repeated, Boolean, and out-of-range indexes and do not support multiset
permutations, custom comparison orders, random selection, mutation of the
domain between operations, or enumeration of preceding and following
arrangements.

## Related Snippets

<!-- catalog:related:start -->
- [Rank and Unrank Index Combinations in itertools.combinations Order](rank-and-unrank-index-combinations-in-itertools-combinations-order.md)
- [Return the Next Lexicographic Permutation of Bounded Integers](return-the-next-lexicographic-permutation-of-bounded-integers.md)
- [Count Strict Inversions in a Bounded Integer Sequence](count-strict-inversions-in-a-bounded-integer-sequence.md)
<!-- catalog:related:end -->
