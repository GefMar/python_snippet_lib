---
title: "Build a Canonical Suffix Array and Adjacent LCP Table for Bounded Unicode Text"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - find-the-earliest-lexicographically-smallest-rotation-of-bounded-unicode-text-with-booths-algorithm.md
  - find-all-overlapping-exact-text-matches-with-knuth-morris-pratt.md
  - find-the-leftmost-longest-common-substring-of-two-bounded-texts.md
---

# Build a Canonical Suffix Array and Adjacent LCP Table for Bounded Unicode Text

## Idea and Problem

Order every suffix of one bounded Unicode string and record how many leading code points each suffix shares with its predecessor.

A suffix array stores start indexes instead of copying all suffix strings.
Prefix doubling first ranks one-character prefixes, then repeatedly sorts by
pairs of ranks for prefixes twice as long. Once every suffix has a distinct
rank, the order is the same as direct lexicographic suffix comparison.

Kasai's scan then derives the adjacent longest-common-prefix table. Moving from
one text position to the next can reduce the reusable prefix by at most one, so
it avoids comparing every adjacent suffix from the beginning.

## When to Use

Use this representation when one complete, bounded text needs reusable suffix
order for repeated-substring analysis, lexicographic range searches, or an
independently inspectable text index. The two returned tuples are aligned:
`lcp_with_previous[i]` compares suffix `suffix_order[i]` with the preceding
suffix, and its first entry is zero.

Prefer direct suffix slicing and sorting for tiny inputs when compact code is
more important than avoiding quadratic copied text. Use a specialized suffix
array library, suffix tree, compressed index, or external-memory design for
very large corpora, online updates, or production search infrastructure.

## Implementation

```python
_MAX_SUFFIX_TEXT_CODE_POINTS = 65_536
_MAX_SUFFIX_TEXT_UTF8_BYTES = 262_144


def _validated_suffix_text(value: object) -> str:
    if type(value) is not str:
        raise TypeError("text must be an exact string")
    if len(value) > _MAX_SUFFIX_TEXT_CODE_POINTS:
        raise ValueError("text exceeds the code-point limit")
    try:
        encoded_size = len(value.encode("utf-8"))
    except UnicodeEncodeError:
        raise ValueError("text must be valid UTF-8 text") from None
    if encoded_size > _MAX_SUFFIX_TEXT_UTF8_BYTES:
        raise ValueError("text exceeds the UTF-8 byte limit")
    return value


def build_suffix_array_and_lcp(
    text: str,
) -> tuple[tuple[int, ...], tuple[int, ...]]:
    """Return suffix starts in lexical order and adjacent LCP lengths."""
    checked = _validated_suffix_text(text)
    length = len(checked)
    if length == 0:
        return (), ()

    order = sorted(range(length), key=checked.__getitem__)
    ranks = [0] * length
    class_index = 0
    ranks[order[0]] = class_index
    for position in range(1, length):
        if checked[order[position]] != checked[order[position - 1]]:
            class_index += 1
        ranks[order[position]] = class_index

    width = 1
    while width < length and class_index + 1 < length:
        order.sort(
            key=lambda start: (
                ranks[start],
                ranks[start + width] if start + width < length else -1,
            )
        )

        next_ranks = [0] * length
        next_class = 0
        previous_start = order[0]
        previous_key = (
            ranks[previous_start],
            ranks[previous_start + width] if previous_start + width < length else -1,
        )
        for start in order[1:]:
            key = (
                ranks[start],
                ranks[start + width] if start + width < length else -1,
            )
            if key != previous_key:
                next_class += 1
            next_ranks[start] = next_class
            previous_key = key

        ranks = next_ranks
        class_index = next_class
        width *= 2

    positions = [0] * length
    for position, start in enumerate(order):
        positions[start] = position

    lcp_with_previous = [0] * length
    common = 0
    for start in range(length):
        position = positions[start]
        if position == 0:
            common = 0
            continue
        previous_start = order[position - 1]
        while (
            start + common < length
            and previous_start + common < length
            and checked[start + common] == checked[previous_start + common]
        ):
            common += 1
        lcp_with_previous[position] = common
        if common:
            common -= 1

    return tuple(order), tuple(lcp_with_previous)
```

## Example

```python
def direct_suffix_array_and_lcp(text: str) -> tuple[tuple[int, ...], tuple[int, ...]]:
    order = tuple(sorted(range(len(text)), key=lambda start: text[start:]))
    lcp = [0] * len(order)
    for position in range(1, len(order)):
        left = order[position - 1]
        right = order[position]
        while (
            left + lcp[position] < len(text)
            and right + lcp[position] < len(text)
            and text[left + lcp[position]] == text[right + lcp[position]]
        ):
            lcp[position] += 1
    return order, tuple(lcp)


def exercise_small_suffix_arrays() -> int:
    from itertools import product

    checked = 0
    for length in range(9):
        for characters in product("ab", repeat=length):
            text = "".join(characters)
            assert build_suffix_array_and_lcp(text) == direct_suffix_array_and_lcp(text)
            checked += 1
    return checked


banana = build_suffix_array_and_lcp("banana")
astral = build_suffix_array_and_lcp("\U0001f642a\U0001f642")
uniform_order, uniform_lcp = build_suffix_array_and_lcp("a" * _MAX_SUFFIX_TEXT_CODE_POINTS)

value_errors = 0
for invalid_text in ("a" * (_MAX_SUFFIX_TEXT_CODE_POINTS + 1), "\ud800"):
    try:
        build_suffix_array_and_lcp(invalid_text)
    except ValueError:
        value_errors += 1

type_errors = 0
try:
    build_suffix_array_and_lcp(b"text")
except TypeError:
    type_errors += 1

assert (
    exercise_small_suffix_arrays(),
    banana,
    astral,
    uniform_order[:3],
    uniform_order[-3:],
    uniform_lcp[:3],
    uniform_lcp[-3:],
    value_errors,
    type_errors,
) == (
    511,
    ((5, 3, 1, 0, 4, 2), (0, 1, 3, 0, 0, 2)),
    ((1, 2, 0), (0, 0, 1)),
    (65_535, 65_534, 65_533),
    (2, 1, 0),
    (0, 1, 2),
    (65_533, 65_534, 65_535),
    2,
    1,
)
```

## Trade-offs and Limitations

This prefix-doubling implementation performs `O(log n)` comparison sorts of
`n` integer-rank pairs, for `O(n log^2 n)` comparison time and `O(n)` working
memory. Kasai's LCP pass adds `O(n)` time and memory. It avoids materializing
all suffix strings, but Python tuple sorting, rank-list allocation, and the two
returned tuples still have meaningful constants.

Ordering uses exact Python Unicode code-point order. It is not locale-aware,
normalization-insensitive, case-insensitive, grapheme-aware, or based on UTF-8
byte offsets. The function rejects surrogate code points and does not retain a
copy of the source text, so callers must keep that text to interpret indexes.

The result is a static index for one bounded string. It does not search a
pattern by itself, build a suffix tree, support incremental edits, compress the
index, combine multiple documents, or provide the stronger asymptotic and
memory guarantees of specialized suffix-array construction algorithms.

## Related Snippets

<!-- catalog:related:start -->
- [Find the Earliest Lexicographically Smallest Rotation of Bounded Unicode Text with Booth's Algorithm](find-the-earliest-lexicographically-smallest-rotation-of-bounded-unicode-text-with-booths-algorithm.md)
- [Find All Overlapping Exact Text Matches with Knuth-Morris-Pratt](find-all-overlapping-exact-text-matches-with-knuth-morris-pratt.md)
- [Find the Leftmost Longest Common Substring of Two Bounded Texts](find-the-leftmost-longest-common-substring-of-two-bounded-texts.md)
<!-- catalog:related:end -->
