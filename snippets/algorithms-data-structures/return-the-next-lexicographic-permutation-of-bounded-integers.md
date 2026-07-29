---
title: "Return the Next Lexicographic Permutation of Bounded Integers"
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
  - find-the-earliest-lexicographically-smallest-rotation-of-bounded-unicode-text-with-booths-algorithm.md
  - count-strict-inversions-in-a-bounded-integer-sequence.md
---

# Return the Next Lexicographic Permutation of Bounded Integers

## Idea and Problem

Advance one bounded integer multiset arrangement to its next distinct lexicographic permutation without enumerating the permutation space.

Among the distinct tuples that contain exactly the same values as the input,
the result is the unique smallest tuple that compares greater than the current
arrangement. The rightmost position that can increase is the pivot. Its
non-increasing suffix contains the least greater replacement at the rightmost
matching position, and reversing the remaining suffix makes that suffix as
small as possible.

## When to Use

Use this algorithm when one materialized arrangement must advance through
distinct permutations in ordinary integer lexicographic order. It is useful
for deterministic search, bounded exhaustive fixtures, and resumable iteration
when the current arrangement is already available.

Start from the sorted values to visit every distinct arrangement in increasing
order, stopping when the function returns `None`. Use `itertools.permutations`
for tiny one-off enumeration when its duplicate results for repeated values are
acceptable. Use a dedicated rank/unrank algorithm when direct random access to
the permutation space is required.

## Implementation

```python
_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_PERMUTATION_VALUES = 100_000


def next_lexicographic_permutation(
    values: tuple[int, ...],
) -> tuple[int, ...] | None:
    """Return the next distinct permutation, or None after the last one."""
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    if len(values) > _MAX_PERMUTATION_VALUES:
        raise ValueError("value count exceeds the supported limit")

    for index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"values[{index}] must be an exact non-boolean integer")
        if not _MIN_INT64 <= value <= _MAX_INT64:
            raise ValueError(f"values[{index}] is outside the signed 64-bit range")

    result = list(values)
    pivot = len(result) - 2
    while pivot >= 0 and result[pivot] >= result[pivot + 1]:
        pivot -= 1
    if pivot < 0:
        return None

    successor = len(result) - 1
    while result[successor] <= result[pivot]:
        successor -= 1

    result[pivot], result[successor] = result[successor], result[pivot]
    result[pivot + 1 :] = reversed(result[pivot + 1 :])
    return tuple(result)
```

## Example

```python
def next_permutation_by_enumeration(
    values: tuple[int, ...],
) -> tuple[int, ...] | None:
    from itertools import permutations

    ordered = sorted(set(permutations(values)))
    position = ordered.index(values)
    return None if position + 1 == len(ordered) else ordered[position + 1]


def exercise_short_permutations() -> int:
    from itertools import product

    checked = 0
    for length in range(7):
        for values in product((-1, 0, 1), repeat=length):
            assert next_lexicographic_permutation(values) == next_permutation_by_enumeration(values)
            checked += 1
    return checked


duplicate_values = (1, 1, 2)
duplicate_snapshot = duplicate_values
duplicate_next = next_lexicographic_permutation(duplicate_values)
extrema_next = next_lexicographic_permutation((_MIN_INT64, _MAX_INT64, _MIN_INT64))

maximum_ascending = tuple(range(_MAX_PERMUTATION_VALUES))
maximum_next = next_lexicographic_permutation(maximum_ascending)
maximum_descending = tuple(range(_MAX_PERMUTATION_VALUES, 0, -1))

try:
    next_lexicographic_permutation((0, True))
except TypeError:
    boolean_rejected = True
else:
    boolean_rejected = False

try:
    next_lexicographic_permutation((0,) * (_MAX_PERMUTATION_VALUES + 1))
except ValueError:
    count_cap_enforced = True
else:
    count_cap_enforced = False

assert (
    exercise_short_permutations(),
    duplicate_next,
    duplicate_values == duplicate_snapshot,
    extrema_next,
    maximum_next is not None,
    maximum_next[:3] if maximum_next is not None else (),
    maximum_next[-3:] if maximum_next is not None else (),
    next_lexicographic_permutation(maximum_descending),
    boolean_rejected,
    count_cap_enforced,
) == (
    1_093,
    (1, 2, 1),
    True,
    (_MAX_INT64, _MIN_INT64, _MIN_INT64),
    True,
    (0, 1, 2),
    (99_997, 99_999, 99_998),
    None,
    True,
    True,
)
```

## Trade-offs and Limitations

Validation, pivot discovery, successor discovery, and suffix reversal take
`O(n)` time. The working list and returned tuple use `O(n)` memory and briefly
coexist. Even when no greater permutation exists, the implementation validates
the complete input and constructs its private working list before returning
`None`.

Lexicographic comparison uses exact Python integer ordering. Repeated values are
allowed, but the result advances to a distinct tuple rather than treating equal
positions as distinguishable. Empty, single-value, and entirely non-increasing
inputs are already their final arrangements and therefore return `None`.

The function handles one immutable snapshot. It does not mutate the input,
return the previous permutation, accept custom comparison keys, enumerate the
remaining space, rank or unrank permutations, or make a factorial-size search
space practical.

## Related Snippets

<!-- catalog:related:start -->
- [Rank and Unrank Index Combinations in itertools.combinations Order](rank-and-unrank-index-combinations-in-itertools-combinations-order.md)
- [Find the Earliest Lexicographically Smallest Rotation of Bounded Unicode Text with Booth's Algorithm](find-the-earliest-lexicographically-smallest-rotation-of-bounded-unicode-text-with-booths-algorithm.md)
- [Count Strict Inversions in a Bounded Integer Sequence](count-strict-inversions-in-a-bounded-integer-sequence.md)
<!-- catalog:related:end -->
