---
title: "Find the Earliest Lexicographically Smallest Rotation of Bounded Unicode Text with Booth's Algorithm"
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
  - find-the-leftmost-longest-exact-text-palindrome-with-manachers-algorithm.md
  - build-a-canonical-unicode-caseless-comparison-key.md
---

# Find the Earliest Lexicographically Smallest Rotation of Bounded Unicode Text with Booth's Algorithm

## Idea and Problem

Find the lexicographically smallest cyclic rotation of bounded Unicode text in linear time, choosing its earliest start when periodic rotations are equal.

Sorting every rotation materializes quadratic text. Booth's algorithm instead
keeps two possible starts in doubled text. At their first mismatch, the losing
start and every start skipped through the matching prefix can be discarded, so
the candidates advance without revisiting a quadratic number of characters.

## When to Use

Use this algorithm when rotations of one already decoded string represent the
same cycle and one deterministic representative is needed under exact Python
code-point ordering. Returning both the start and rotated text preserves the
connection to the original indexes while making the canonical value explicit.

Prefer direct rotation enumeration for tiny inputs when simplicity matters
more than worst-case work. Normalize or case-fold before calling only when that
transformation is part of the surrounding contract; use a grapheme or
locale-aware library when displayed characters or collation define ordering.

## Implementation

```python
_MAX_TEXT_CODE_POINTS = 65_536
_MAX_TEXT_UTF8_BYTES = 262_144


def _validated_rotation_text(value: object) -> str:
    if type(value) is not str:
        raise TypeError("text must be an exact string")
    if len(value) > _MAX_TEXT_CODE_POINTS:
        raise ValueError("text exceeds the code-point limit")
    try:
        encoded_size = len(value.encode("utf-8"))
    except UnicodeEncodeError:
        raise ValueError("text must be valid UTF-8 text") from None
    if encoded_size > _MAX_TEXT_UTF8_BYTES:
        raise ValueError("text exceeds the UTF-8 byte limit")
    return value


def find_earliest_smallest_rotation(text: str) -> tuple[int, str]:
    """Return the earliest start and smallest exact cyclic rotation."""
    checked = _validated_rotation_text(text)
    length = len(checked)
    if length == 0:
        return 0, ""

    doubled = checked + checked
    first = 0
    second = 1
    matched = 0

    while first < length and second < length and matched < length:
        first_character = doubled[first + matched]
        second_character = doubled[second + matched]
        if first_character == second_character:
            matched += 1
            continue

        if first_character > second_character:
            first += matched + 1
            if first <= second:
                first = second + 1
        else:
            second += matched + 1
            if second <= first:
                second = first + 1
        matched = 0

    start = min(first, second)
    return start, doubled[start : start + length]
```

## Example

```python
ordinary = find_earliest_smallest_rotation("caba")
periodic_tie = find_earliest_smallest_rotation("baba")
uniform_tie = find_earliest_smallest_rotation("aaaa")
astral = find_earliest_smallest_rotation("\U0001f642a\U0001f642")
empty = find_earliest_smallest_rotation("")

assert (ordinary, periodic_tie, uniform_tie, astral, empty) == (
    (1, "abac"),
    (1, "abab"),
    (0, "aaaa"),
    (1, "a\U0001f642\U0001f642"),
    (0, ""),
)
```

## Trade-offs and Limitations

Strict UTF-8 validation and Booth's scan take `O(n)` time for `n` Python code
points. The temporary UTF-8 encoding, doubled text, and returned rotation each
use bounded `O(n)` memory, while the candidate indexes use constant auxiliary
state. The returned rotation is a new string even when the earliest start is
zero.

Ordering is exact, case-sensitive, and normalization-sensitive. Python compares
Unicode code points rather than grapheme clusters or locale collation weights,
so canonically equivalent strings can produce different rotations and indexes.
The start is a code-point index, not a UTF-8 byte offset.

The algorithm handles one materialized string and returns one representative.
It does not stream, enumerate every equal minimum, normalize text, compare
approximately, or find palindromes and substring matches. Periodic ties select
the smallest original start index.

## Related Snippets

<!-- catalog:related:start -->
- [Find All Overlapping Exact Text Matches with Knuth-Morris-Pratt](find-all-overlapping-exact-text-matches-with-knuth-morris-pratt.md)
- [Find the Leftmost Longest Exact Text Palindrome with Manacher's Algorithm](find-the-leftmost-longest-exact-text-palindrome-with-manachers-algorithm.md)
- [Build a Canonical Unicode Caseless Comparison Key](build-a-canonical-unicode-caseless-comparison-key.md)
<!-- catalog:related:end -->
