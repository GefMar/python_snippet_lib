---
title: "Find the Leftmost Longest Exact Text Palindrome with Manacher's Algorithm"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - find-all-overlapping-exact-text-matches-with-knuth-morris-pratt.md
  - find-a-longest-common-integer-subsequence-with-earliest-index-pair-ties.md
  - build-a-canonical-unicode-caseless-comparison-key.md
---

# Find the Leftmost Longest Exact Text Palindrome with Manacher's Algorithm

## Idea and Problem

Find the longest contiguous palindrome in bounded text while resolving an equal-length choice by its leftmost position.

Expanding independently around every possible center can revisit the same text
quadratically. Manacher's algorithm instead remembers the rightmost palindrome
already found. A mirrored radius supplies a safe starting point inside that
palindrome, so each boundary extension contributes to one linear scan.

## When to Use

Use this algorithm when the input is one bounded, already decoded string and
the result must be exact under Python code-point comparison. It supports both
odd- and even-length palindromes and returns a half-open span so the caller can
slice the original text without copying every candidate.

Choose a simpler center-expansion scan for very short text when constant memory
matters more than worst-case time. Use a normalization or grapheme-aware layer
first when visual characters, canonical equivalence, or caseless comparison
define equality in the application.

## Implementation

```python
from dataclasses import dataclass

_MAX_TEXT_CODE_POINTS = 65_536
_MAX_TEXT_UTF8_BYTES = 262_144


@dataclass(frozen=True, slots=True)
class PalindromeSpan:
    start: int
    stop: int


def find_leftmost_longest_palindrome(text: str) -> PalindromeSpan | None:
    """Return the leftmost longest exact palindrome as a half-open span."""
    if type(text) is not str:
        raise TypeError("text must be an exact string")
    if len(text) > _MAX_TEXT_CODE_POINTS:
        raise ValueError("text exceeds the code-point limit")
    try:
        encoded_size = len(text.encode("utf-8"))
    except UnicodeEncodeError:
        raise ValueError("text must be valid UTF-8 text") from None
    if encoded_size > _MAX_TEXT_UTF8_BYTES:
        raise ValueError("text exceeds the UTF-8 byte limit")
    if not text:
        return None

    length = len(text)
    odd_radii = [0] * length
    left = 0
    right = -1
    for center in range(length):
        radius = (
            1
            if center > right
            else min(odd_radii[left + right - center], right - center + 1)
        )
        while (
            center - radius >= 0
            and center + radius < length
            and text[center - radius] == text[center + radius]
        ):
            radius += 1
        odd_radii[center] = radius
        if center + radius - 1 > right:
            left = center - radius + 1
            right = center + radius - 1

    even_radii = [0] * length
    left = 0
    right = -1
    for center in range(length):
        radius = (
            0
            if center > right
            else min(even_radii[left + right - center + 1], right - center + 1)
        )
        while (
            center - radius - 1 >= 0
            and center + radius < length
            and text[center - radius - 1] == text[center + radius]
        ):
            radius += 1
        even_radii[center] = radius
        if center + radius - 1 > right:
            left = center - radius
            right = center + radius - 1

    best = PalindromeSpan(0, 1)
    for center, radius in enumerate(odd_radii):
        candidate = PalindromeSpan(center - radius + 1, center + radius)
        if (candidate.stop - candidate.start, -candidate.start) > (
            best.stop - best.start,
            -best.start,
        ):
            best = candidate
    for center, radius in enumerate(even_radii):
        candidate = PalindromeSpan(center - radius, center + radius)
        if (candidate.stop - candidate.start, -candidate.start) > (
            best.stop - best.start,
            -best.start,
        ):
            best = candidate
    return best
```

## Example

```python
even = find_leftmost_longest_palindrome("forgeeksskeegfor")
leftmost_tie = find_leftmost_longest_palindrome("abacdfgdcaba")
unicode_text = find_leftmost_longest_palindrome("xétéz")
empty = find_leftmost_longest_palindrome("")

assert (even, leftmost_tie, unicode_text, empty) == (
    PalindromeSpan(3, 13),
    PalindromeSpan(0, 3),
    PalindromeSpan(1, 4),
    None,
)
```

## Trade-offs and Limitations

Validation and the two Manacher scans take `O(n)` time for `n` Python code
points. The odd and even radius arrays use `O(n)` memory; UTF-8 validation also
creates a bounded temporary encoding. The returned span itself uses constant
space and refers to code-point indexes in the original string.

Comparison is exact, case-sensitive, and normalization-sensitive. A combining
sequence and a precomposed character are different, and one displayed grapheme
may contain several code points. The function rejects lone surrogates and does
not normalize, case-fold, approximate, stream, or return every maximum.

## Related Snippets

<!-- catalog:related:start -->
- [Find All Overlapping Exact Text Matches with Knuth-Morris-Pratt](find-all-overlapping-exact-text-matches-with-knuth-morris-pratt.md)
- [Find a Longest Common Integer Subsequence with Earliest Index-Pair Ties](find-a-longest-common-integer-subsequence-with-earliest-index-pair-ties.md)
- [Build a Canonical Unicode Caseless Comparison Key](build-a-canonical-unicode-caseless-comparison-key.md)
<!-- catalog:related:end -->
