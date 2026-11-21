---
title: "Find the Matching Range for a Literal Prefix in Sorted Strings"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
tested_python:
  - "3.14"
dependencies: []
related:
  - build-a-bounded-immutable-text-trie-for-longest-prefix-lookup.md
  - ../data-processing/route-items-by-ordered-text-prefixes.md
---

# Find the Matching Range for a Literal Prefix in Sorted Strings

## Idea and Problem

Use two binary searches to find every string beginning with a literal prefix in a sorted sequence.

Strings with the same prefix occupy adjacent positions. The first search
finds the insertion point for the prefix itself. The second searches the
remaining keys after projecting each to its first `len(prefix)` characters.
That projection preserves the ordering and lets `bisect_right()` stop after
the final match, including duplicates.

## When to Use

Use this for repeated range queries over a stable, random-access sequence
sorted by Python's ordinary string ordering. The returned indices define a
half-open range. Use them to count matches without copying, or slice the input
if you need the matching strings. An empty prefix selects the entire sequence.

## Implementation

```python
from bisect import bisect_left, bisect_right
from collections.abc import Sequence


def literal_prefix_range(keys: Sequence[str], prefix: str) -> tuple[int, int]:
    """Return [start, stop) for a prefix in already sorted string keys."""
    if type(prefix) is not str:
        raise TypeError("prefix must be a string")

    start = bisect_left(keys, prefix)
    width = len(prefix)
    stop = bisect_right(keys, prefix, lo=start, key=lambda key: key[:width])
    return start, stop
```

## Example

```python
keys = ("a", "app", "app", "apple", "apply", "apt", "b", "\U0010ffff", "\U0010ffffx")
start, stop = literal_prefix_range(keys, "app")

assert (
    (start, stop),
    keys[start:stop],
    literal_prefix_range(keys, "aq"),
    literal_prefix_range(keys, ""),
    literal_prefix_range(keys, "\U0010ffff"),
) == (
    (1, 5),
    ("app", "app", "apple", "apply"),
    (6, 6),
    (0, 9),
    (7, 9),
)
```

## Trade-offs and Limitations

For tuple or list input, each search takes logarithmically many comparisons
in the number of keys. Comparisons and slices can each inspect up to the
prefix length. The caller owns the sorted-order precondition and must keep
the sequence stable during the search. A `startswith()` scan suits occasional
queries over unsorted input.

Matching is case-sensitive and uses the supplied string spelling. For locale,
case-folded or normalized matching, construct and order comparison keys under
that policy before choosing a search algorithm.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Bounded Immutable Text Trie for Longest-Prefix Lookup](build-a-bounded-immutable-text-trie-for-longest-prefix-lookup.md)
- [Route Items by Ordered Text Prefixes](../data-processing/route-items-by-ordered-text-prefixes.md)
<!-- catalog:related:end -->
