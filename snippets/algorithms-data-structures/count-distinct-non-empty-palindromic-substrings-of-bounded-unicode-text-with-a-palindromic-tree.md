---
title: "Count Distinct Non-Empty Palindromic Substrings of Bounded Unicode Text with a Palindromic Tree"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - find-the-leftmost-longest-exact-text-palindrome-with-manachers-algorithm.md
  - count-distinct-non-empty-substrings-of-bounded-unicode-text-with-a-suffix-automaton.md
  - compute-the-longest-palindromic-subsequence-length-of-bounded-unicode-text.md
---

# Count Distinct Non-Empty Palindromic Substrings of Bounded Unicode Text with a Palindromic Tree

## Idea and Problem

Count the distinct non-empty contiguous palindromes in one bounded Unicode string without materializing the substring values.

A palindromic tree, also called an eertree, has one node for every distinct
non-empty palindromic substring plus two structural roots of lengths `-1`
and `0`. While the text is scanned, suffix links move from the longest
palindromic suffix to shorter candidates until the new code point can extend
one on both sides.

An absent transition creates exactly one new palindrome node. An existing
transition means the same palindrome value occurred before. The required
count is therefore the number of nodes other than the two roots, regardless
of how many positions produce each value.

## When to Use

Use this function when one complete bounded string needs an exact count of its
different palindromic substrings without retaining their slices or occurrence
positions. It is useful for repetition summaries, compact fixture comparison,
and validating tiny reference implementations against a scalable structure.

Use Manacher's algorithm when only one longest palindrome is required. Use a
general distinct-substring index when non-palindromic values matter too.
Normalize or segment text before this function when canonical equivalence,
case-insensitive matching, or grapheme clusters define equality.

## Implementation

```python
_MAX_PALINDROMIC_TREE_CODE_POINTS = 65_536
_MAX_PALINDROMIC_TREE_UTF8_BYTES = 262_144
_ODD_ROOT = 0
_EVEN_ROOT = 1


def count_distinct_non_empty_palindromic_substrings(text: str) -> int:
    """Return the exact count of distinct non-empty code-point palindromes."""
    if type(text) is not str:
        raise TypeError("text must be an exact string")
    if len(text) > _MAX_PALINDROMIC_TREE_CODE_POINTS:
        raise ValueError("text exceeds the code-point limit")
    try:
        encoded_size = len(text.encode("utf-8"))
    except UnicodeEncodeError:
        raise ValueError("text must not contain lone surrogate code points") from None
    if encoded_size > _MAX_PALINDROMIC_TREE_UTF8_BYTES:
        raise ValueError("text exceeds the UTF-8 byte limit")

    palindrome_lengths = [-1, 0]
    suffix_links = [_ODD_ROOT, _ODD_ROOT]
    transitions: list[dict[str, int]] = [{}, {}]
    longest_suffix = _EVEN_ROOT

    for position, character in enumerate(text):
        extendable = longest_suffix
        while True:
            candidate_length = palindrome_lengths[extendable]
            left_position = position - candidate_length - 1
            if left_position >= 0 and text[left_position] == character:
                break
            extendable = suffix_links[extendable]

        existing = transitions[extendable].get(character)
        if existing is not None:
            longest_suffix = existing
            continue

        new_node = len(palindrome_lengths)
        new_length = palindrome_lengths[extendable] + 2
        palindrome_lengths.append(new_length)
        suffix_links.append(_EVEN_ROOT)
        transitions.append({})
        transitions[extendable][character] = new_node

        if new_length == 1:
            longest_suffix = new_node
            continue

        fallback = suffix_links[extendable]
        while True:
            fallback_length = palindrome_lengths[fallback]
            left_position = position - fallback_length - 1
            if left_position >= 0 and text[left_position] == character:
                break
            fallback = suffix_links[fallback]
        suffix_links[new_node] = transitions[fallback][character]
        longest_suffix = new_node

    return len(palindrome_lengths) - 2
```

## Example

```python
def count_palindromes_by_materialized_set(text: str) -> int:
    return len(
        {
            candidate
            for start in range(len(text))
            for stop in range(start + 1, len(text) + 1)
            if (candidate := text[start:stop]) == candidate[::-1]
        }
    )


def exercise_short_texts() -> int:
    from itertools import product

    checked = 0
    for length in range(9):
        for characters in product("abc", repeat=length):
            text = "".join(characters)
            assert count_distinct_non_empty_palindromic_substrings(
                text
            ) == count_palindromes_by_materialized_set(text)
            checked += 1
    return checked


uniform_count = count_distinct_non_empty_palindromic_substrings(
    "a" * _MAX_PALINDROMIC_TREE_CODE_POINTS
)
astral_character = chr(0x1F642)
astral_count = count_distinct_non_empty_palindromic_substrings(
    astral_character * _MAX_PALINDROMIC_TREE_CODE_POINTS
)
distinct_text = "".join(chr(0x1000 + offset) for offset in range(2_048))


class TextSubclass(str):
    pass


rejected = 0
for invalid_text in (
    TextSubclass("aba"),
    b"aba",
    chr(0xD800),
    "x" * (_MAX_PALINDROMIC_TREE_CODE_POINTS + 1),
):
    try:
        count_distinct_non_empty_palindromic_substrings(invalid_text)
    except (TypeError, ValueError):
        rejected += 1

assert (
    exercise_short_texts(),
    count_distinct_non_empty_palindromic_substrings(""),
    count_distinct_non_empty_palindromic_substrings("banana"),
    count_distinct_non_empty_palindromic_substrings("ababa"),
    count_distinct_non_empty_palindromic_substrings("aaaa"),
    count_distinct_non_empty_palindromic_substrings("e" + chr(0x0301) + "e"),
    count_distinct_non_empty_palindromic_substrings(astral_character + "a" + astral_character),
    uniform_count,
    astral_count,
    count_distinct_non_empty_palindromic_substrings(distinct_text),
    rejected,
) == (9_841, 0, 6, 5, 4, 3, 3, 65_536, 65_536, 2_048, 4)
```

## Trade-offs and Limitations

For `N` code points, the tree contains at most `N + 2` nodes and at most
`N` stored transitions. Suffix-link traversal plus expected constant-time
dictionary access gives expected `O(N)` construction time and `O(N)`
memory. Python hash-table behavior is not an unconditional worst-case
guarantee. UTF-8 validation adds linear work and a temporary encoding of at
most 262,144 bytes.

The count uses exact Python code-point sequences. Comparison is case-sensitive
and normalization-sensitive, so canonically equivalent spellings and visually
identical grapheme clusters can remain distinct. The empty string is excluded,
and repeated occurrences of the same palindromic value contribute only once.
The returned Python integer is exact.

The function returns no substring values, positions, occurrence counts,
longest-palindrome result, suffix links, or reusable tree object. It does not
count palindromic subsequences, normalize Unicode, segment graphemes, accept
bytes, combine several strings, process chunks, or update a retained index.

## Related Snippets

<!-- catalog:related:start -->
- [Find the Leftmost Longest Exact Text Palindrome with Manacher's Algorithm](find-the-leftmost-longest-exact-text-palindrome-with-manachers-algorithm.md)
- [Count Distinct Non-Empty Substrings of Bounded Unicode Text with a Suffix Automaton](count-distinct-non-empty-substrings-of-bounded-unicode-text-with-a-suffix-automaton.md)
- [Compute the Longest Palindromic Subsequence Length of Bounded Unicode Text](compute-the-longest-palindromic-subsequence-length-of-bounded-unicode-text.md)
<!-- catalog:related:end -->
