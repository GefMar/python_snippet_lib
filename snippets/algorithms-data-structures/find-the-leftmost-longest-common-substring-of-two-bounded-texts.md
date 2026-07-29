---
title: "Find the Leftmost Longest Common Substring of Two Bounded Texts"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - find-a-longest-common-integer-subsequence-with-earliest-index-pair-ties.md
  - build-a-canonical-suffix-array-and-adjacent-lcp-table-for-bounded-unicode-text.md
  - find-all-overlapping-exact-text-matches-with-knuth-morris-pratt.md
---

# Find the Leftmost Longest Common Substring of Two Bounded Texts

## Idea and Problem

Find one longest non-empty contiguous string shared by two bounded texts, with an explicit earliest-position rule for equal lengths.

A rolling dynamic-programming row records the common suffix length of prefixes
ending at each pair of positions. Equal characters extend the diagonal value
from the previous row; unequal characters reset it to zero. Every positive
cell therefore identifies one common substring without storing the full
quadratic table.

The result first maximizes length, then minimizes the start in the left text,
then the start in the right text. These rules make repeated-text ties stable
without copying the selected substring.

## When to Use

Use this algorithm when both complete strings are already in memory, exact
contiguous equality is required, the bounded length product is acceptable, and
the caller needs source positions in both texts. Returning a span triple lets
the caller slice either original value only when the text itself is needed.

Do not confuse a common substring with a common subsequence: substring
characters must remain adjacent. Prefer direct nested comparison for tiny
inputs, a suffix-array or suffix-automaton approach for substantially larger
texts, and a streaming or approximate matcher when materialization or exact
equality is not part of the contract.

## Implementation

```python
_MAX_COMMON_TEXT_CODE_POINTS = 65_536
_MAX_COMMON_TEXT_UTF8_BYTES = 262_144
_MAX_COMMON_TEXT_PRODUCT = 1_000_000


def _validated_common_text(value: object, *, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if len(value) > _MAX_COMMON_TEXT_CODE_POINTS:
        raise ValueError(f"{field} exceeds the code-point limit")
    try:
        encoded_size = len(value.encode("utf-8"))
    except UnicodeEncodeError:
        raise ValueError(f"{field} must be valid UTF-8 text") from None
    if encoded_size > _MAX_COMMON_TEXT_UTF8_BYTES:
        raise ValueError(f"{field} exceeds the UTF-8 byte limit")
    return value


def find_leftmost_longest_common_substring(
    left: str,
    right: str,
) -> tuple[int, int, int] | None:
    """Return (left_start, right_start, length) under earliest-start ties."""
    checked_left = _validated_common_text(left, field="left")
    checked_right = _validated_common_text(right, field="right")
    if len(checked_left) * len(checked_right) > _MAX_COMMON_TEXT_PRODUCT:
        raise ValueError("text length product exceeds the supported limit")

    previous = [0] * (len(checked_right) + 1)
    best_length = 0
    best_left_start = 0
    best_right_start = 0

    for left_index, left_character in enumerate(checked_left):
        current = [0] * (len(checked_right) + 1)
        for right_index, right_character in enumerate(checked_right):
            if left_character != right_character:
                continue
            length = previous[right_index] + 1
            current[right_index + 1] = length
            left_start = left_index - length + 1
            right_start = right_index - length + 1
            if length > best_length or (
                length == best_length
                and (left_start, right_start) < (best_left_start, best_right_start)
            ):
                best_length = length
                best_left_start = left_start
                best_right_start = right_start
        previous = current

    if best_length == 0:
        return None
    return best_left_start, best_right_start, best_length
```

## Example

```python
def direct_leftmost_longest_common_substring(
    left: str,
    right: str,
) -> tuple[int, int, int] | None:
    best: tuple[int, int, int] | None = None
    for left_start in range(len(left)):
        for right_start in range(len(right)):
            length = 0
            while (
                left_start + length < len(left)
                and right_start + length < len(right)
                and left[left_start + length] == right[right_start + length]
            ):
                length += 1
            candidate = (left_start, right_start, length)
            if length and (
                best is None or length > best[2] or (length == best[2] and candidate[:2] < best[:2])
            ):
                best = candidate
    return best


def exercise_small_text_pairs() -> int:
    from itertools import product

    texts = tuple(
        "".join(characters) for length in range(6) for characters in product("ab", repeat=length)
    )
    checked = 0
    for left in texts:
        for right in texts:
            assert find_leftmost_longest_common_substring(
                left,
                right,
            ) == direct_leftmost_longest_common_substring(left, right)
            checked += 1
    return checked


tie = find_leftmost_longest_common_substring("ababa", "babab")
none = find_leftmost_longest_common_substring("abc", "XYZ")
boundary = find_leftmost_longest_common_substring("a" * 1_000, "x" + "a" * 999)

value_errors = 0
for left, right in (("a" * 1_001, "b" * 1_000), ("\ud800", "text")):
    try:
        find_leftmost_longest_common_substring(left, right)
    except ValueError:
        value_errors += 1

type_errors = 0
try:
    find_leftmost_longest_common_substring("text", b"text")
except TypeError:
    type_errors += 1

assert (exercise_small_text_pairs(), tie, none, boundary, value_errors, type_errors) == (
    3_969,
    (0, 1, 4),
    None,
    (0, 1, 999),
    2,
    1,
)
```

## Trade-offs and Limitations

For lengths `m` and `n`, the algorithm uses `O(m * n)` time and `O(n)`
auxiliary integers, where `n` is the right-text length. The explicit product
limit bounds the quadratic work. Swapping arguments can reduce row memory, but
this implementation preserves their declared roles and tie order directly.

Exact code-point comparison is case-sensitive and normalization-sensitive.
Returned positions are Python string indexes rather than UTF-8 byte offsets or
grapheme indexes. The function rejects surrogate code points and does not copy
the matching text; slice `left[start : start + length]` when a copy is needed.

The dynamic program handles two static strings and one longest match. It does
not enumerate every equal optimum, find non-contiguous subsequences, match
approximately, normalize text, compare by locale, accept more than two texts,
reuse work after edits, or scale beyond the bounded Cartesian product.

## Related Snippets

<!-- catalog:related:start -->
- [Find a Longest Common Integer Subsequence with Earliest Index-Pair Ties](find-a-longest-common-integer-subsequence-with-earliest-index-pair-ties.md)
- [Build a Canonical Suffix Array and Adjacent LCP Table for Bounded Unicode Text](build-a-canonical-suffix-array-and-adjacent-lcp-table-for-bounded-unicode-text.md)
- [Find All Overlapping Exact Text Matches with Knuth-Morris-Pratt](find-all-overlapping-exact-text-matches-with-knuth-morris-pratt.md)
<!-- catalog:related:end -->
