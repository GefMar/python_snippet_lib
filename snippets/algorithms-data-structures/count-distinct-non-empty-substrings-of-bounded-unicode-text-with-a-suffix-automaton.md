---
title: "Count Distinct Non-Empty Substrings of Bounded Unicode Text with a Suffix Automaton"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - build-a-canonical-suffix-array-and-adjacent-lcp-table-for-bounded-unicode-text.md
  - find-the-leftmost-longest-common-substring-of-two-bounded-texts.md
  - compute-the-z-array-of-bounded-unicode-text.md
---

# Count Distinct Non-Empty Substrings of Bounded Unicode Text with a Suffix Automaton

## Idea and Problem

Count the distinct non-empty contiguous code-point sequences in one bounded Unicode string without materializing those substrings.

A suffix automaton compactly represents the end-position classes of every
substring seen while scanning the text. Each state stores the greatest length
in its class, a suffix link to the class containing its longest proper suffix,
and transitions for the next code point.

Every non-root state contributes exactly the lengths strictly above its suffix
link's maximum. Summing those differences counts each distinct non-empty
substring once, even when the same text occurs at several positions.

## When to Use

Use this function when one complete, bounded string needs an exact distinct-
substring count without retaining the substrings themselves. It is useful for
measuring repetition, comparing compact text fixtures, and validating a small
string-processing implementation against a materialized reference set.

Use a suffix array and adjacent LCP values when lexicographic suffix order is
also needed. Materialize a set only for tiny inputs or reference checks. Use a
specialized text index for multiple documents, incremental updates, occurrence
queries, or inputs too large for Python dictionaries per automaton state.

## Implementation

```python
_MAX_AUTOMATON_CODE_POINTS = 65_536
_MAX_AUTOMATON_UTF8_BYTES = 262_144


def count_distinct_non_empty_substrings(text: str) -> int:
    """Return the exact number of distinct non-empty code-point substrings."""
    if type(text) is not str:
        raise TypeError("text must be an exact string")
    if len(text) > _MAX_AUTOMATON_CODE_POINTS:
        raise ValueError("text exceeds the code-point limit")
    try:
        encoded_size = len(text.encode("utf-8"))
    except UnicodeEncodeError:
        raise ValueError("text must not contain lone surrogate code points") from None
    if encoded_size > _MAX_AUTOMATON_UTF8_BYTES:
        raise ValueError("text exceeds the UTF-8 byte limit")

    if len(text) < 2:
        return len(text)

    maximum_lengths = [0]
    suffix_links = [-1]
    transitions: list[dict[str, int]] = [{}]
    last_state = 0

    for character in text:
        current_state = len(maximum_lengths)
        maximum_lengths.append(maximum_lengths[last_state] + 1)
        suffix_links.append(0)
        transitions.append({})

        predecessor = last_state
        while predecessor != -1 and character not in transitions[predecessor]:
            transitions[predecessor][character] = current_state
            predecessor = suffix_links[predecessor]

        if predecessor == -1:
            suffix_links[current_state] = 0
        else:
            next_state = transitions[predecessor][character]
            if maximum_lengths[predecessor] + 1 == maximum_lengths[next_state]:
                suffix_links[current_state] = next_state
            else:
                clone_state = len(maximum_lengths)
                maximum_lengths.append(maximum_lengths[predecessor] + 1)
                suffix_links.append(suffix_links[next_state])
                transitions.append(transitions[next_state].copy())

                while (
                    predecessor != -1
                    and transitions[predecessor].get(character) == next_state
                ):
                    transitions[predecessor][character] = clone_state
                    predecessor = suffix_links[predecessor]

                suffix_links[next_state] = clone_state
                suffix_links[current_state] = clone_state

        last_state = current_state

    return sum(
        maximum_lengths[state] - maximum_lengths[suffix_links[state]]
        for state in range(1, len(maximum_lengths))
    )
```

## Example

```python
def count_distinct_substrings_by_set(text: str) -> int:
    return len(
        {
            text[start:stop]
            for start in range(len(text))
            for stop in range(start + 1, len(text) + 1)
        }
    )


def exercise_short_texts() -> int:
    from itertools import product

    checked = 0
    for length in range(8):
        for characters in product("abc", repeat=length):
            text = "".join(characters)
            assert count_distinct_non_empty_substrings(
                text
            ) == count_distinct_substrings_by_set(text)
            checked += 1
    return checked


uniform_count = count_distinct_non_empty_substrings(
    "a" * _MAX_AUTOMATON_CODE_POINTS
)
astral_count = count_distinct_non_empty_substrings(
    "\U0001f642" * _MAX_AUTOMATON_CODE_POINTS
)
distinct_text = "".join(chr(0x1000 + offset) for offset in range(2_048))


class TextSubclass(str):
    pass


rejected = 0
for invalid_text in (
    TextSubclass("aba"),
    "\ud800",
    "x" * (_MAX_AUTOMATON_CODE_POINTS + 1),
):
    try:
        count_distinct_non_empty_substrings(invalid_text)
    except (TypeError, ValueError):
        rejected += 1

assert (
    exercise_short_texts(),
    count_distinct_non_empty_substrings(""),
    count_distinct_non_empty_substrings("banana"),
    count_distinct_non_empty_substrings("abab"),
    count_distinct_non_empty_substrings("e\u0301"),
    count_distinct_non_empty_substrings("\U0001f642a\U0001f642"),
    uniform_count,
    astral_count,
    count_distinct_non_empty_substrings(distinct_text),
    rejected,
) == (3_280, 0, 15, 7, 3, 5, 65_536, 65_536, 2_098_176, 3)
```

## Trade-offs and Limitations

Empty and one-code-point inputs return directly. For `N >= 2`, the automaton
has at most `2 * N - 1` states and a linear number of transition entries.
Construction performs `O(N)` dictionary operations in aggregate and takes
expected amortized `O(N)` time; this is not an unconditional worst-case time
guarantee for Python hash tables. It uses `O(N)` states and transitions. UTF-8
validation adds `O(N)` work and a temporary encoding of at most 262,144 bytes.

The result counts exact Python code-point sequences. Comparison is
case-sensitive and normalization-sensitive, so grapheme clusters and
canonically equivalent spellings are not coalesced. The empty string is not
counted. The returned Python integer is exact, although its bit width and the
cost of arithmetic are not fixed-width promises.

The function returns no substrings, positions, occurrence counts, repeated-
substring statistics, or automaton object. It does not normalize Unicode,
segment graphemes, accept bytes, combine several texts, process chunks, or
update an existing index after edits.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Canonical Suffix Array and Adjacent LCP Table for Bounded Unicode Text](build-a-canonical-suffix-array-and-adjacent-lcp-table-for-bounded-unicode-text.md)
- [Find the Leftmost Longest Common Substring of Two Bounded Texts](find-the-leftmost-longest-common-substring-of-two-bounded-texts.md)
- [Compute the Z-Array of Bounded Unicode Text](compute-the-z-array-of-bounded-unicode-text.md)
<!-- catalog:related:end -->
