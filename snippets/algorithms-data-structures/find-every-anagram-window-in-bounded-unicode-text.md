---
title: "Find Every Anagram Window in Bounded Unicode Text"
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
  - compute-full-window-trailing-maxima-with-a-monotonic-index-deque.md
  - find-exact-heavy-hitters-with-verified-misra-gries-candidates.md
---

# Find Every Anagram Window in Bounded Unicode Text

## Idea and Problem

Find every fixed-width text window whose Unicode code-point multiset exactly matches one non-empty pattern.

An anagram preserves character multiplicities rather than character order. A
frequency-difference map records the counts still needed from the pattern and
subtracts the counts present in the current text window. The window is an
anagram exactly when every difference is zero.

Updating only the character that leaves the window and the character that
enters it avoids recounting every position. A separate count of non-zero map
entries makes each match check constant expected time without comparing two
complete frequency maps.

## When to Use

Use this algorithm when a bounded, fully materialized Unicode string must be
searched for every contiguous rearrangement of one exact non-empty pattern.
It fits letter-inventory searches, order-insensitive symbol-window checks, and
small deterministic validation tasks where overlapping matches must be
retained in ascending start order.

Both inputs are exact, case-sensitive and normalization-sensitive Python
strings. Each input may contain at most 65,536 code points and at most 262,144
UTF-8 bytes, and surrogate code points are rejected. An empty text or a pattern
longer than the text produces no matches; an empty pattern is rejected because
it would make every boundary an ambiguous zero-width match.

## Implementation

```python
_MAX_TEXT_CODE_POINTS = 65_536
_MAX_TEXT_UTF8_BYTES = 262_144


def _validate_bounded_exact_text(value: str, *, name: str) -> None:
    if type(value) is not str:
        raise TypeError(f"{name} must be an exact string")
    if len(value) > _MAX_TEXT_CODE_POINTS:
        raise ValueError(f"{name} exceeds the code-point limit")
    if any(0xD800 <= ord(character) <= 0xDFFF for character in value):
        raise ValueError(f"{name} must not contain surrogate code points")
    if len(value.encode("utf-8")) > _MAX_TEXT_UTF8_BYTES:
        raise ValueError(f"{name} exceeds the UTF-8 byte limit")


def find_anagram_window_starts(text: str, pattern: str) -> tuple[int, ...]:
    """Return every start whose fixed-width window is an anagram of pattern."""
    _validate_bounded_exact_text(text, name="text")
    _validate_bounded_exact_text(pattern, name="pattern")
    if not pattern:
        raise ValueError("pattern must be non-empty")
    if len(pattern) > len(text):
        return ()

    differences: dict[str, int] = {}
    non_zero_keys = 0

    def adjust(character: str, amount: int) -> None:
        nonlocal non_zero_keys

        previous = differences.get(character, 0)
        current = previous + amount
        if previous == 0 and current != 0:
            non_zero_keys += 1
        elif previous != 0 and current == 0:
            non_zero_keys -= 1

        if current == 0:
            differences.pop(character, None)
        else:
            differences[character] = current

    window_width = len(pattern)
    for character in pattern:
        adjust(character, 1)
    for character in text[:window_width]:
        adjust(character, -1)

    matches: list[int] = []
    if non_zero_keys == 0:
        matches.append(0)

    for stop in range(window_width, len(text)):
        adjust(text[stop - window_width], 1)
        adjust(text[stop], -1)
        if non_zero_keys == 0:
            matches.append(stop - window_width + 1)

    return tuple(matches)
```

## Example

```python
def anagram_starts_by_counting(text: str, pattern: str) -> tuple[int, ...]:
    from collections import Counter

    expected_counts = Counter(pattern)
    width = len(pattern)
    return tuple(
        start
        for start in range(len(text) - width + 1)
        if Counter(text[start : start + width]) == expected_counts
    )


def exercise_small_anagram_windows() -> int:
    from itertools import product

    checked = 0
    for text_length in range(8):
        for text_characters in product("ab", repeat=text_length):
            text = "".join(text_characters)
            for pattern_length in range(1, 5):
                for pattern_characters in product("ab", repeat=pattern_length):
                    pattern = "".join(pattern_characters)
                    assert find_anagram_window_starts(
                        text,
                        pattern,
                    ) == anagram_starts_by_counting(text, pattern)
                    checked += 1
    return checked


def raises(error_type: type[Exception], operation) -> bool:
    try:
        operation()
    except error_type:
        return True
    return False


maximum_text = "\U0001f600" * _MAX_TEXT_CODE_POINTS

assert (
    exercise_small_anagram_windows(),
    find_anagram_window_starts("cbaebabacd", "abc"),
    find_anagram_window_starts("aaaaa", "aa"),
    find_anagram_window_starts("abababa", "aab"),
    find_anagram_window_starts("", "a"),
    find_anagram_window_starts("abc", "abcd"),
    find_anagram_window_starts(maximum_text, maximum_text),
    raises(ValueError, lambda: find_anagram_window_starts("abc", "")),
    raises(ValueError, lambda: find_anagram_window_starts("\ud800", "a")),
    raises(ValueError, lambda: find_anagram_window_starts("a" * 65_537, "a")),
    raises(TypeError, lambda: find_anagram_window_starts("abc", True)),
) == (
    7_650,
    (0, 6),
    (0, 1, 2, 3),
    (0, 2, 4),
    (),
    (),
    (0,),
    True,
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

Validation and UTF-8 size measurement scan both inputs before matching and
temporarily materialize their UTF-8 encodings. Frequency initialization and
the sliding scan use expected `O(P + T)` time for pattern length `P` and text
length `T`. The difference map uses `O(U)` entries for the distinct code points
encountered, and the returned tuple uses `O(M)` slots for `M` matches.

Dictionary access is expected constant time rather than a worst-case
constant-time guarantee. The implementation stores only non-zero differences,
but a large diverse alphabet can still make the map proportional to the input.
Highly repetitive text can produce a match at almost every start, so output
materialization can dominate the working state even though the scan is linear.

Matching uses exact Python string code points. It does not normalize Unicode,
case-fold text, combine grapheme clusters, honor locale-specific equivalence,
accept wildcards, tolerate substitutions, or find variable-width
rearrangements. It also does not retain state across streamed chunks or support
changing the pattern during a scan.

## Related Snippets

<!-- catalog:related:start -->
- [Find All Overlapping Exact Text Matches with Knuth-Morris-Pratt](find-all-overlapping-exact-text-matches-with-knuth-morris-pratt.md)
- [Compute Full-Window Trailing Maxima with a Monotonic Index Deque](compute-full-window-trailing-maxima-with-a-monotonic-index-deque.md)
- [Find Exact Heavy Hitters with Verified Misra-Gries Candidates](find-exact-heavy-hitters-with-verified-misra-gries-candidates.md)
<!-- catalog:related:end -->
