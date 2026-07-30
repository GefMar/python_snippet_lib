---
title: "Compute the Longest Palindromic Subsequence Length of Bounded Unicode Text"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - find-the-leftmost-longest-exact-text-palindrome-with-manachers-algorithm.md
  - find-a-longest-common-integer-subsequence-with-earliest-index-pair-ties.md
  - build-a-canonical-minimum-unit-cost-edit-script-between-bounded-unicode-texts.md
---

# Compute the Longest Palindromic Subsequence Length of Bounded Unicode Text

## Idea and Problem

Compute the maximum number of code points in a palindromic subsequence without constructing the subsequence itself.

For each half-open text interval, equal endpoints can surround the best inner
subsequence. Unequal endpoints require dropping either the left or the right
endpoint and retaining the larger result. Iterating left endpoints backward
allows one mutable row to represent all intervals beginning at the current or
next position.

## When to Use

Use this dynamic program when characters may be skipped, exact code-point
comparison defines equality, and only the optimum length is needed. The single
row is useful for bounded text where a quadratic table would retain unnecessary
predecessor state.

Use a contiguous-palindrome algorithm when skipped positions are forbidden.
Keep the full dynamic-programming table or separate reconstruction state when
an actual subsequence and deterministic tie rule are required. Normalize or
segment text first when grapheme clusters or canonical equivalence define a
character in the application.

## Implementation

```python
_MAX_LPS_CODE_POINTS = 2_048


def longest_palindromic_subsequence_length(text: str) -> int:
    """Return the longest exact code-point palindromic subsequence length."""
    if type(text) is not str:
        raise TypeError("text must be an exact string")
    if len(text) > _MAX_LPS_CODE_POINTS:
        raise ValueError("text exceeds the code-point limit")
    try:
        text.encode("utf-8")
    except UnicodeEncodeError:
        raise ValueError("text must not contain lone surrogate code points") from None
    if not text:
        return 0

    best_by_right = [0] * len(text)
    for left in range(len(text) - 1, -1, -1):
        previous_diagonal = 0
        best_by_right[left] = 1
        for right in range(left + 1, len(text)):
            previous_row_value = best_by_right[right]
            if text[left] == text[right]:
                best_by_right[right] = previous_diagonal + 2
            else:
                best_by_right[right] = max(
                    best_by_right[right],
                    best_by_right[right - 1],
                )
            previous_diagonal = previous_row_value

    return best_by_right[-1]
```

## Example

```python
def longest_palindromic_subsequence_length_by_table(text: str) -> int:
    if not text:
        return 0

    length = len(text)
    table = [[0] * length for _ in range(length)]
    for index in range(length):
        table[index][index] = 1
    for span in range(2, length + 1):
        for left in range(length - span + 1):
            right = left + span - 1
            if text[left] == text[right]:
                inner = 0 if span == 2 else table[left + 1][right - 1]
                table[left][right] = inner + 2
            else:
                table[left][right] = max(
                    table[left + 1][right],
                    table[left][right - 1],
                )
    return table[0][-1]


def exercise_short_texts() -> int:
    from itertools import product

    checked = 0
    for length in range(8):
        for code_points in product("abc", repeat=length):
            text = "".join(code_points)
            expected = longest_palindromic_subsequence_length_by_table(text)
            assert longest_palindromic_subsequence_length(text) == expected
            assert longest_palindromic_subsequence_length(text[::-1]) == expected
            checked += 1
    return checked


checked_texts = exercise_short_texts()
cases = ("", "a", "bbbab", "cbbd", "agbdba", "abcda", "été")
lengths = tuple(longest_palindromic_subsequence_length(text) for text in cases)
maximum_length = longest_palindromic_subsequence_length("a" * _MAX_LPS_CODE_POINTS)


class TextSubclass(str):
    pass


rejected = 0
for invalid_text in (
    TextSubclass("aba"),
    "\ud800",
    "x" * (_MAX_LPS_CODE_POINTS + 1),
):
    try:
        longest_palindromic_subsequence_length(invalid_text)
    except (TypeError, ValueError):
        rejected += 1

assert (
    checked_texts,
    lengths,
    maximum_length,
    rejected,
) == (3_280, (0, 1, 4, 2, 5, 3, 3), _MAX_LPS_CODE_POINTS, 3)
```

## Trade-offs and Limitations

For `N` Python code points, the nested interval recurrence takes `O(N**2)`
time. The mutable dynamic-programming row uses `O(N)` integers. UTF-8
validation creates a temporary of at most four bytes per admitted code point,
and the returned length uses constant additional space.

Comparison is exact, case-sensitive, and normalization-sensitive. A combining
sequence and a precomposed character are different, and one displayed grapheme
may contain several code points. Lone surrogate code points are rejected. The
2,048-code-point cap bounds quadratic work rather than encoded byte size.

The function returns only a length. It does not reconstruct a subsequence,
define ties among equal-length witnesses, require contiguity, normalize or
case-fold text, segment graphemes, stream input, or update after text changes.

## Related Snippets

<!-- catalog:related:start -->
- [Find the Leftmost Longest Exact Text Palindrome with Manacher's Algorithm](find-the-leftmost-longest-exact-text-palindrome-with-manachers-algorithm.md)
- [Find a Longest Common Integer Subsequence with Earliest Index-Pair Ties](find-a-longest-common-integer-subsequence-with-earliest-index-pair-ties.md)
- [Build a Canonical Minimum Unit-Cost Edit Script Between Bounded Unicode Texts](build-a-canonical-minimum-unit-cost-edit-script-between-bounded-unicode-texts.md)
<!-- catalog:related:end -->
