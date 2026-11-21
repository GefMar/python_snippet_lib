---
title: "Count Minimum Adjacent Swaps Between Equal Character Multisets"
snippet_type: algorithm
use_cases:
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - count-strict-inversions-in-a-bounded-integer-sequence.md
  - find-the-shortest-exact-repeating-unit-of-a-string.md
---

# Count Minimum Adjacent Swaps Between Equal Character Multisets

## Idea and Problem

Count the fewest swaps of neighboring characters needed to transform one string into an anagram.

Match each character occurrence to the next unused occurrence of that character in the target. Crossing the assignments of two equal characters cannot improve the swap count. Count the inversions of the resulting target positions with a Fenwick tree.

## When to Use

Use this for exact code-point strings when only adjacent swaps are allowed, and two equal characters are interchangeable. It returns a count rather than constructing the swap sequence.

## Implementation

```python
from collections import defaultdict, deque


def minimum_adjacent_swaps(source: str, target: str) -> int:
    n = len(source)
    if n != len(target):
        raise ValueError("source and target must have the same characters")

    positions: dict[str, deque[int]] = defaultdict(deque)
    for index, char in enumerate(target, start=1):
        positions[char].append(index)

    tree = [0] * (n + 1)
    swaps = 0
    for placed, char in enumerate(source):
        available = positions.get(char)
        if not available:
            raise ValueError("source and target must have the same characters")
        position = available.popleft()

        before = 0
        cursor = position
        while cursor:
            before += tree[cursor]
            cursor -= cursor & -cursor
        swaps += placed - before

        cursor = position
        while cursor <= n:
            tree[cursor] += 1
            cursor += cursor & -cursor
    return swaps
```

## Example

```python
try:
    minimum_adjacent_swaps("aab", "abb")
except ValueError:
    rejected_mismatch = True
else:
    rejected_mismatch = False

assert (
    minimum_adjacent_swaps("aabb", "bbaa"),
    minimum_adjacent_swaps("😀a😀", "😀😀a"),
    minimum_adjacent_swaps("", ""),
    rejected_mismatch,
) == (4, 1, 0, True)
```

## Trade-offs and Limitations

The count uses O(n log n) integer operations and O(n) storage. Matching is by code point, not by normalized Unicode text or grapheme clusters. If the caller needs a list of swaps, that list can be quadratic even though the count is cheaper to compute.

## Related Snippets

<!-- catalog:related:start -->
- [Count Strict Inversions in a Bounded Integer Sequence](count-strict-inversions-in-a-bounded-integer-sequence.md)
- [Find the Shortest Exact Repeating Unit of a String](find-the-shortest-exact-repeating-unit-of-a-string.md)
<!-- catalog:related:end -->
