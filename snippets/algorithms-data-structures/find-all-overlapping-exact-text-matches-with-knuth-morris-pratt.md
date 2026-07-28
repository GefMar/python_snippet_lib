---
title: "Find All Overlapping Exact Text Matches with Knuth-Morris-Pratt"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - build-a-bounded-immutable-text-trie-for-longest-prefix-lookup.md
  - find-a-longest-common-integer-subsequence-with-earliest-index-pair-ties.md
  - build-a-canonical-unicode-caseless-comparison-key.md
---

# Find All Overlapping Exact Text Matches with Knuth-Morris-Pratt

## Idea and Problem

Find every overlapping occurrence of one exact text pattern in linear time without rescanning characters after a partial match fails.

Knuth-Morris-Pratt first records how much of the pattern remains useful after
each mismatch. The search then moves the pattern state through the text without
moving the text position backward. After a complete match, falling back through
the same prefix table preserves a suffix that can begin an overlapping match.

## When to Use

Use this algorithm when one bounded pattern must be matched exactly, every
overlap matters, and a visible `O(n + m)` scanning guarantee is more important
than delegating the search to a runtime-specific implementation. Returned
positions are Python string indexes, so they identify Unicode code points.

Prefer repeated `str.find()` calls for ordinary one-off application searches;
the built-in implementation is shorter and is usually faster in practice. Use
a regular-expression engine for pattern syntax, a multi-pattern algorithm for
many needles, or a streaming matcher when text cannot be held as one string.

## Implementation

```python
_MAX_TEXT_CHARACTERS = 65_536
_MAX_TEXT_BYTES = 262_144
_MAX_PATTERN_CHARACTERS = 4_096
_MAX_PATTERN_BYTES = 16_384


def _validated_utf8_string(
    value: object,
    *,
    field: str,
    max_characters: int,
    max_bytes: int,
    allow_empty: bool,
) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if not allow_empty and not value:
        raise ValueError(f"{field} must not be empty")
    if len(value) > max_characters:
        raise ValueError(f"{field} exceeds the supported character count")
    encoded_length = 0
    for character in value:
        code_point = ord(character)
        if code_point <= 0x7F:
            encoded_length += 1
        elif code_point <= 0x7FF:
            encoded_length += 2
        elif 0xD800 <= code_point <= 0xDFFF:
            raise ValueError(f"{field} must be valid UTF-8 text")
        elif code_point <= 0xFFFF:
            encoded_length += 3
        else:
            encoded_length += 4
        if encoded_length > max_bytes:
            raise ValueError(f"{field} exceeds the supported UTF-8 byte length")
    return value


def find_overlapping_exact_matches(text: str, pattern: str) -> tuple[int, ...]:
    """Return every code-point start index where pattern occurs in text."""
    checked_text = _validated_utf8_string(
        text,
        field="text",
        max_characters=_MAX_TEXT_CHARACTERS,
        max_bytes=_MAX_TEXT_BYTES,
        allow_empty=True,
    )
    checked_pattern = _validated_utf8_string(
        pattern,
        field="pattern",
        max_characters=_MAX_PATTERN_CHARACTERS,
        max_bytes=_MAX_PATTERN_BYTES,
        allow_empty=False,
    )

    prefix_lengths = [0] * len(checked_pattern)
    matched = 0
    for index in range(1, len(checked_pattern)):
        while matched and checked_pattern[index] != checked_pattern[matched]:
            matched = prefix_lengths[matched - 1]
        if checked_pattern[index] == checked_pattern[matched]:
            matched += 1
        prefix_lengths[index] = matched

    matches: list[int] = []
    matched = 0
    for index, character in enumerate(checked_text):
        while matched and character != checked_pattern[matched]:
            matched = prefix_lengths[matched - 1]
        if character == checked_pattern[matched]:
            matched += 1
        if matched == len(checked_pattern):
            matches.append(index - len(checked_pattern) + 1)
            matched = prefix_lengths[matched - 1]
    return tuple(matches)
```

## Example

```python
overlapping = find_overlapping_exact_matches("bananana", "ana")
repeated = find_overlapping_exact_matches("aaaaa", "aaa")
astral = find_overlapping_exact_matches(
    "\U0001f642a\U0001f642a\U0001f642",
    "\U0001f642a\U0001f642",
)
composed = find_overlapping_exact_matches("caf\u00e9", "\u00e9")
not_normalized = find_overlapping_exact_matches("cafe\u0301", "\u00e9")

try:
    find_overlapping_exact_matches("text", "")
except ValueError:
    empty_pattern_rejected = True
else:
    empty_pattern_rejected = False

assert (
    overlapping,
    repeated,
    astral,
    composed,
    not_normalized,
    find_overlapping_exact_matches("", "x"),
    empty_pattern_rejected,
) == ((1, 3, 5), (0, 1, 2), (0, 2), (3,), (), (), True)
```

## Trade-offs and Limitations

Validation, prefix construction, scanning, and result materialization take
`O(n + m + k)` time for `n` text code points, `m` pattern code points, and `k`
matches. The prefix table needs `O(m)` working memory; the temporary match list
and returned tuple need `O(k)` result memory and briefly coexist at return.

Matching is case-sensitive and compares exact Python Unicode code points. It
does not normalize text, case-fold, identify grapheme clusters, return byte
offsets, or accept an empty pattern. The prefix table is rebuilt on every call,
and a highly repetitive input can produce a result almost as large as the
text. Regular expressions, wildcards, approximate matching, multiple patterns,
and chunked streams are outside this function's contract.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Bounded Immutable Text Trie for Longest-Prefix Lookup](build-a-bounded-immutable-text-trie-for-longest-prefix-lookup.md)
- [Find a Longest Common Integer Subsequence with Earliest Index-Pair Ties](find-a-longest-common-integer-subsequence-with-earliest-index-pair-ties.md)
- [Build a Canonical Unicode Caseless Comparison Key](build-a-canonical-unicode-caseless-comparison-key.md)
<!-- catalog:related:end -->
