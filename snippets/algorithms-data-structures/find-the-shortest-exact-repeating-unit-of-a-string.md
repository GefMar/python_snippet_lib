---
title: "Find the Shortest Exact Repeating Unit of a String"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - find-all-overlapping-exact-text-matches-with-knuth-morris-pratt.md
  - compute-the-z-array-of-bounded-unicode-text.md
---

# Find the Shortest Exact Repeating Unit of a String

## Idea and Problem

Find the shortest literal unit whose complete repetitions reconstruct the entire string.

The longest proper prefix that also appears as a suffix is called a border.
Subtract its length from the string length to find the shortest period. The
period becomes an exact repeating unit only when its length divides the
whole string. Otherwise, the entire string is the shortest exact unit.

## When to Use

Use this to recognize repeated labels, patterns or symbols in one complete
string. Exact repetition compares Python string positions in their original
order. The empty string returns an empty unit by convention.

## Implementation

```python
def shortest_repeating_unit(text: str) -> str:
    """Return the shortest prefix whose repetitions reconstruct text exactly."""
    if type(text) is not str:
        raise TypeError("text must be a string")
    if not text:
        return ""

    borders = [0] * len(text)
    for index in range(1, len(text)):
        matched = borders[index - 1]
        while matched and text[index] != text[matched]:
            matched = borders[matched - 1]
        if text[index] == text[matched]:
            matched += 1
        borders[index] = matched

    width = len(text) - borders[-1]
    return text[:width] if len(text) % width == 0 else text
```

## Example

```python
assert (
    shortest_repeating_unit("step-step-step-"),
    shortest_repeating_unit("abcdabc"),
    shortest_repeating_unit("aaaaa"),
    shortest_repeating_unit("\U0001f680a\U0001f680a"),
    shortest_repeating_unit(""),
) == ("step-", "abcdabc", "a", "\U0001f680a", "")
```

## Trade-offs and Limitations

The algorithm uses linear time and stores one border length per position.
A mismatch falls back to a shorter border that was already computed. The
table is useful when an input contains many overlapping candidate prefixes.

The result depends on the exact string spelling. If equivalence requires
case folding, Unicode normalization or a different unit of comparison,
apply that policy before searching.

## Related Snippets

<!-- catalog:related:start -->
- [Find All Overlapping Exact Text Matches with Knuth-Morris-Pratt](find-all-overlapping-exact-text-matches-with-knuth-morris-pratt.md)
- [Compute the Z-Array of Bounded Unicode Text](compute-the-z-array-of-bounded-unicode-text.md)
<!-- catalog:related:end -->
