---
title: "Compute the Z-Array of Bounded Unicode Text"
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
  - build-a-canonical-suffix-array-and-adjacent-lcp-table-for-bounded-unicode-text.md
  - find-the-earliest-lexicographically-smallest-rotation-of-bounded-unicode-text-with-booths-algorithm.md
---

# Compute the Z-Array of Bounded Unicode Text

## Idea and Problem

Compute how many leading code points of one text match at every possible suffix start.

For position `i`, the Z-value is the longest exact match between the whole
text and the suffix beginning at `i`. The first value is defined as the full
text length. A half-open window records the rightmost prefix match found so
far, allowing positions inside that window to reuse an earlier Z-value before
performing any new comparisons.

## When to Use

Use the Z-array when one bounded string needs all of its prefix-versus-suffix
match lengths. It is useful for exact border analysis, repetition checks, and
algorithms that reduce literal pattern matching to prefix matches in one
combined string.

Use `str` operations for an ordinary one-off lookup. Use a KMP matcher when
the direct result should be pattern occurrences, or a suffix array when
arbitrary suffix ordering and adjacent longest-common-prefix queries are the
main objective. Choose a streaming algorithm when the complete text cannot be
held in memory.

## Implementation

```python
_MAX_Z_CODE_POINTS = 65_536
_MAX_Z_UTF8_BYTES = 262_144


def compute_z_array(text: str) -> tuple[int, ...]:
    """Return exact prefix-match lengths, defining the first value as len(text)."""
    if type(text) is not str:
        raise TypeError("text must be an exact string")
    if len(text) > _MAX_Z_CODE_POINTS:
        raise ValueError("text exceeds the code-point limit")
    try:
        encoded_size = len(text.encode("utf-8"))
    except UnicodeEncodeError:
        raise ValueError("text must not contain lone surrogate code points") from None
    if encoded_size > _MAX_Z_UTF8_BYTES:
        raise ValueError("text exceeds the UTF-8 byte limit")

    length = len(text)
    if length == 0:
        return ()

    matches = [0] * length
    matches[0] = length
    window_start = 0
    window_stop = 0

    for start in range(1, length):
        if start < window_stop:
            matches[start] = min(
                window_stop - start,
                matches[start - window_start],
            )

        while (
            start + matches[start] < length
            and text[matches[start]] == text[start + matches[start]]
        ):
            matches[start] += 1

        candidate_stop = start + matches[start]
        if candidate_stop > window_stop:
            window_start = start
            window_stop = candidate_stop

    return tuple(matches)
```

## Example

```python
def compute_z_array_naively(text: str) -> tuple[int, ...]:
    answer = []
    for start in range(len(text)):
        matched = 0
        while start + matched < len(text) and text[matched] == text[start + matched]:
            matched += 1
        answer.append(matched)
    return tuple(answer)


def exercise_short_texts() -> int:
    from itertools import product

    checked = 0
    for length in range(9):
        for characters in product("ab", repeat=length):
            text = "".join(characters)
            assert compute_z_array(text) == compute_z_array_naively(text)
            checked += 1
    return checked


uniform = compute_z_array("a" * _MAX_Z_CODE_POINTS)
astral_boundary = compute_z_array("\U0001f642" * _MAX_Z_CODE_POINTS)
unicode_text = "\U0001f642a\U0001f642a\U0001f642"


class TextSubclass(str):
    pass


rejected = 0
for invalid_text in (
    TextSubclass("aba"),
    "\ud800",
    "x" * (_MAX_Z_CODE_POINTS + 1),
):
    try:
        compute_z_array(invalid_text)
    except (TypeError, ValueError):
        rejected += 1

assert (
    exercise_short_texts(),
    compute_z_array(""),
    compute_z_array("a"),
    compute_z_array("abacaba"),
    compute_z_array(unicode_text),
    (len(uniform), uniform[0], uniform[-1]),
    (len(astral_boundary), astral_boundary[0], astral_boundary[-1]),
    rejected,
) == (
    511,
    (),
    (1,),
    (7, 0, 1, 0, 3, 0, 1),
    (5, 0, 3, 0, 1),
    (_MAX_Z_CODE_POINTS, _MAX_Z_CODE_POINTS, 1),
    (_MAX_Z_CODE_POINTS, _MAX_Z_CODE_POINTS, 1),
    3,
)
```

## Trade-offs and Limitations

For `N` code points, the window scan takes `O(N)` comparisons and the returned
tuple uses `O(N)` integer references. The mutable list and immutable tuple
briefly coexist at return. UTF-8 validation takes `O(N)` time and creates a
temporary encoding of at most 262,144 bytes.

The result uses Python code-point indexes and exact, case-sensitive,
normalization-sensitive comparison. Input is limited to 65,536 code points and
262,144 UTF-8 bytes. The empty input returns `()`, while every non-empty result
has `z[0] == len(text)`. Lone surrogate code points are rejected; grapheme
clusters and canonically equivalent spellings are not coalesced.

This function computes the table only. It does not join a pattern and text,
choose or escape a separator, return match positions, retain the source text,
normalize Unicode, accept bytes, process chunks, or update after edits.

## Related Snippets

<!-- catalog:related:start -->
- [Find All Overlapping Exact Text Matches with Knuth-Morris-Pratt](find-all-overlapping-exact-text-matches-with-knuth-morris-pratt.md)
- [Build a Canonical Suffix Array and Adjacent LCP Table for Bounded Unicode Text](build-a-canonical-suffix-array-and-adjacent-lcp-table-for-bounded-unicode-text.md)
- [Find the Earliest Lexicographically Smallest Rotation of Bounded Unicode Text with Booth's Algorithm](find-the-earliest-lexicographically-smallest-rotation-of-bounded-unicode-text-with-booths-algorithm.md)
<!-- catalog:related:end -->
