---
title: "Factor Bounded Unicode Text into Lyndon Words with Duval's Algorithm"
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
  - build-a-canonical-suffix-array-and-adjacent-lcp-table-for-bounded-unicode-text.md
  - find-all-overlapping-exact-text-matches-with-knuth-morris-pratt.md
---

# Factor Bounded Unicode Text into Lyndon Words with Duval's Algorithm

## Idea and Problem

Split bounded Unicode text into its unique non-increasing sequence of lexicographically minimal primitive factors.

A Lyndon word is a non-empty string that is strictly smaller than each of its
non-trivial cyclic rotations. The Chen-Fox-Lyndon theorem gives every string
one factorization into Lyndon words in lexicographically non-increasing order.
Duval's algorithm finds the factor boundaries by comparing one candidate word
with the text that follows it and reusing any repeated candidate prefix.

Returning half-open spans preserves the exact positions and avoids copying the
factor strings. Consecutive equal factors remain separate because each span is
one member of the unique factorization.

## When to Use

Use this function when a fully materialized string needs a canonical
combinatorial decomposition for text-algorithm experiments, periodicity
analysis, or deterministic fixtures. Ordering follows Python's exact Unicode
code-point comparison, and the spans cover the original text from left to
right without gaps or overlaps.

The caller must want code-point semantics and the Chen-Fox-Lyndon
factorization specifically. Use direct slicing for a known delimiter-based
format, normalization before this function when canonical equivalence is part
of the contract, or a Unicode-aware library when grapheme clusters or locale
collation determine visible ordering.

## Implementation

```python
_MAX_LYNDON_CODE_POINTS = 65_536
_MAX_LYNDON_UTF8_BYTES = 262_144


def _validated_lyndon_text(value: object) -> str:
    if type(value) is not str:
        raise TypeError("text must be an exact string")
    if len(value) > _MAX_LYNDON_CODE_POINTS:
        raise ValueError("text exceeds the code-point limit")
    try:
        encoded_size = len(value.encode("utf-8"))
    except UnicodeEncodeError:
        raise ValueError("text must be valid UTF-8 text") from None
    if encoded_size > _MAX_LYNDON_UTF8_BYTES:
        raise ValueError("text exceeds the UTF-8 byte limit")
    return value


def factor_text_into_lyndon_spans(
    text: str,
) -> tuple[tuple[int, int], ...]:
    """Return the unique non-increasing Lyndon factor spans of text."""
    checked = _validated_lyndon_text(text)
    length = len(checked)
    spans: list[tuple[int, int]] = []
    factor_start = 0

    while factor_start < length:
        scan = factor_start + 1
        candidate_position = factor_start

        while scan < length and checked[candidate_position] <= checked[scan]:
            if checked[candidate_position] < checked[scan]:
                candidate_position = factor_start
            else:
                candidate_position += 1
            scan += 1

        factor_length = scan - candidate_position
        while factor_start <= candidate_position:
            factor_stop = factor_start + factor_length
            spans.append((factor_start, factor_stop))
            factor_start = factor_stop

    return tuple(spans)
```

## Example

```python
def is_lyndon_by_rotations(word: str) -> bool:
    return bool(word) and all(
        word < word[offset:] + word[:offset] for offset in range(1, len(word))
    )


def brute_lyndon_factor_spans(text: str) -> tuple[tuple[int, int], ...]:
    from itertools import pairwise

    if not text:
        return ()

    candidates: list[tuple[tuple[int, int], ...]] = []
    for cut_mask in range(1 << (len(text) - 1)):
        spans: list[tuple[int, int]] = []
        factor_start = 0
        for boundary in range(1, len(text) + 1):
            has_cut = boundary == len(text) or cut_mask & (1 << (boundary - 1))
            if has_cut:
                spans.append((factor_start, boundary))
                factor_start = boundary

        factors = tuple(text[start:stop] for start, stop in spans)
        if all(is_lyndon_by_rotations(factor) for factor in factors) and all(
            first >= second for first, second in pairwise(factors)
        ):
            candidates.append(tuple(spans))

    assert len(candidates) == 1
    return candidates[0]


def exercise_binary_texts() -> int:
    from itertools import product

    checked = 0
    for length in range(9):
        for characters in product("ab", repeat=length):
            text = "".join(characters)
            expected = brute_lyndon_factor_spans(text)
            actual = factor_text_into_lyndon_spans(text)
            assert actual == expected
            assert "".join(text[start:stop] for start, stop in actual) == text
            checked += 1
    return checked


def raises(error_type: type[Exception], operation) -> bool:
    try:
        operation()
    except error_type:
        return True
    return False


ordinary_text = "banana"
ordinary_spans = factor_text_into_lyndon_spans(ordinary_text)
ordinary_factors = tuple(ordinary_text[start:stop] for start, stop in ordinary_spans)

astral_text = "\U0001f642" * _MAX_LYNDON_CODE_POINTS
astral_spans = factor_text_into_lyndon_spans(astral_text)

assert (
    exercise_binary_texts(),
    ordinary_spans,
    ordinary_factors,
    factor_text_into_lyndon_spans("aaaa"),
    factor_text_into_lyndon_spans(""),
    len(astral_text.encode("utf-8")),
    len(astral_spans),
    astral_spans[0],
    astral_spans[-1],
    raises(TypeError, lambda: factor_text_into_lyndon_spans(b"abc")),
    raises(
        ValueError,
        lambda: factor_text_into_lyndon_spans("a" * (_MAX_LYNDON_CODE_POINTS + 1)),
    ),
    raises(ValueError, lambda: factor_text_into_lyndon_spans("\ud800")),
) == (
    511,
    ((0, 1), (1, 3), (3, 5), (5, 6)),
    ("b", "an", "an", "a"),
    ((0, 1), (1, 2), (2, 3), (3, 4)),
    (),
    262_144,
    65_536,
    (0, 1),
    (65_535, 65_536),
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

For `n` code points, UTF-8 validation and Duval's scan take `O(n)` time, and
the scan performs `O(n)` code-point comparisons. The returned tuple uses
`O(f)` spans for `f` factors. UTF-8 validation creates a bounded `O(n)`
temporary byte string, while the algorithm does not copy factor substrings.

Ordering is exact, case-sensitive, and normalization-sensitive. Python
compares Unicode code points rather than grapheme clusters or locale collation
weights, so canonically equivalent strings can have different boundaries. The
span endpoints are code-point indexes, not UTF-8 byte offsets.

The function handles one static string. It does not return every cyclic
rotation, normalize text, compare approximately, stream across chunks, or
maintain a factorization after edits. The factors are lexicographically
minimal primitive words, not linguistic words or delimiter-separated tokens.

## Related Snippets

<!-- catalog:related:start -->
- [Find the Earliest Lexicographically Smallest Rotation of Bounded Unicode Text with Booth's Algorithm](find-the-earliest-lexicographically-smallest-rotation-of-bounded-unicode-text-with-booths-algorithm.md)
- [Build a Canonical Suffix Array and Adjacent LCP Table for Bounded Unicode Text](build-a-canonical-suffix-array-and-adjacent-lcp-table-for-bounded-unicode-text.md)
- [Find All Overlapping Exact Text Matches with Knuth-Morris-Pratt](find-all-overlapping-exact-text-matches-with-knuth-morris-pratt.md)
<!-- catalog:related:end -->
