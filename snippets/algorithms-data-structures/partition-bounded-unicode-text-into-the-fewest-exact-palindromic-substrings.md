---
title: "Partition Bounded Unicode Text into the Fewest Exact Palindromic Substrings"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - find-the-leftmost-longest-exact-text-palindrome-with-manachers-algorithm.md
  - count-distinct-non-empty-palindromic-substrings-of-bounded-unicode-text-with-a-palindromic-tree.md
  - compute-the-longest-palindromic-subsequence-length-of-bounded-unicode-text.md
---

# Partition Bounded Unicode Text into the Fewest Exact Palindromic Substrings

## Idea and Problem

Cover an entire bounded string with the fewest contiguous palindromes while resolving equally short partitions deterministically.

A palindrome lookup table answers whether any half-open substring is valid.
A suffix dynamic program then chooses the minimum number of pieces from each
position. Trying candidate ends in increasing order makes the first selected
end the smallest possible one; applying the same rule to the remaining suffix
produces the lexicographically smallest complete tuple of end offsets.

## When to Use

Use this algorithm when every code point must belong to exactly one contiguous
piece, each piece must read identically in both directions, and reproducible
ties matter. Returning both half-open spans and substrings is convenient for
small text-segmentation diagnostics, exact test fixtures, and later processing
that needs either indexes or detached values.

Equality is Python code-point equality. Normalize or segment text before this
function when canonical Unicode equivalence, grapheme clusters, locale rules,
or case-insensitive matching define the real requirement. For only the longest
single palindrome, use a linear longest-palindrome algorithm instead.

## Implementation

```python
from collections.abc import Callable
from dataclasses import dataclass

_MAX_TEXT_CODE_POINTS = 1_024
_MAX_TEXT_UTF8_BYTES = 4_096


@dataclass(frozen=True, slots=True)
class PalindromePartition:
    spans: tuple[tuple[int, int], ...]
    parts: tuple[str, ...]


def partition_into_fewest_palindromes(text: str) -> PalindromePartition:
    """Return a canonical minimum-piece exact-palindrome partition."""
    if type(text) is not str:
        raise TypeError("text must be an exact string")
    if len(text) > _MAX_TEXT_CODE_POINTS:
        raise ValueError("text exceeds the code-point limit")
    try:
        encoded_size = len(text.encode("utf-8"))
    except UnicodeEncodeError:
        raise ValueError("text must contain valid UTF-8 code points") from None
    if encoded_size > _MAX_TEXT_UTF8_BYTES:
        raise ValueError("text exceeds the UTF-8 byte limit")
    if not text:
        return PalindromePartition(spans=(), parts=())

    length = len(text)
    is_palindrome = [bytearray(length) for _ in range(length)]
    for start in range(length - 1, -1, -1):
        for stop in range(start + 1, length + 1):
            matches_inside = stop - start <= 2 or bool(is_palindrome[start + 1][stop - 2])
            if text[start] == text[stop - 1] and matches_inside:
                is_palindrome[start][stop - 1] = 1

    minimum_parts = [0] * (length + 1)
    next_stop = [length] * length
    for start in range(length - 1, -1, -1):
        best_count = length + 1
        chosen_stop = -1
        for stop in range(start + 1, length + 1):
            if not is_palindrome[start][stop - 1]:
                continue
            candidate_count = 1 + minimum_parts[stop]
            if candidate_count < best_count:
                best_count = candidate_count
                chosen_stop = stop
        minimum_parts[start] = best_count
        next_stop[start] = chosen_stop

    spans: list[tuple[int, int]] = []
    parts: list[str] = []
    start = 0
    while start < length:
        stop = next_stop[start]
        spans.append((start, stop))
        parts.append(text[start:stop])
        start = stop

    return PalindromePartition(spans=tuple(spans), parts=tuple(parts))
```

## Example

```python

def _oracle_end_offsets(text: str) -> tuple[int, ...]:
    if not text:
        return ()
    best: tuple[int, tuple[int, ...]] | None = None
    for cut_mask in range(1 << (len(text) - 1)):
        ends_list = [
            position for position in range(1, len(text)) if cut_mask & (1 << (position - 1))
        ]
        ends_list.append(len(text))
        ends = tuple(ends_list)
        start = 0
        valid = True
        for stop in ends:
            part = text[start:stop]
            if part != part[::-1]:
                valid = False
                break
            start = stop
        if valid:
            candidate = (len(ends), ends)
            if best is None or candidate < best:
                best = candidate
    assert best is not None
    return best[1]


def _raises(
    error_type: type[BaseException],
    function: Callable[..., object],
    *args: object,
) -> bool:
    try:
        function(*args)
    except error_type:
        return True
    return False


first = partition_into_fewest_palindromes("aab")
assert first == PalindromePartition(
    spans=((0, 2), (2, 3)),
    parts=("aa", "b"),
)

# Both ("a", "bab") and ("aba", "b") use two pieces; end 1 wins.
tied = partition_into_fewest_palindromes("abab")
assert tied.parts == ("a", "bab")
assert partition_into_fewest_palindromes("") == PalindromePartition((), ())
assert partition_into_fewest_palindromes("noon").parts == ("noon",)

checked = 0
for size in range(9):
    for encoded_text in range(1 << size):
        text = "".join("b" if encoded_text & (1 << index) else "a" for index in range(size))
        result = partition_into_fewest_palindromes(text)
        assert tuple(stop for _, stop in result.spans) == _oracle_end_offsets(text)
        assert "".join(result.parts) == text
        assert all(part == part[::-1] for part in result.parts)
        checked += 1


class TextSubclass(str):
    pass


assert _raises(TypeError, partition_into_fewest_palindromes, TextSubclass("a"))
assert _raises(ValueError, partition_into_fewest_palindromes, "x" * 1_025)
assert _raises(ValueError, partition_into_fewest_palindromes, "\ud800")
assert checked == 511
```

## Trade-offs and Limitations

For `n` code points, table construction and suffix optimization take `O(n²)`
time. The byte-backed palindrome table uses `O(n²)` bytes; the DP arrays,
spans, and detached substrings add `O(n)` bounded storage. This is appropriate
for the explicit 1,024-code-point limit, but not for large or streaming text.

The tie rule minimizes the tuple of exclusive code-point ends only after
minimizing piece count. It does not minimize substring values or byte offsets.
The result is case-sensitive and normalization-sensitive, rejects lone
surrogates, and does not enumerate every optimum, operate on grapheme clusters,
or perform approximate palindrome matching.

## Related Snippets

<!-- catalog:related:start -->
- [Find the Leftmost Longest Exact Text Palindrome with Manacher's Algorithm](find-the-leftmost-longest-exact-text-palindrome-with-manachers-algorithm.md)
- [Count Distinct Non-Empty Palindromic Substrings of Bounded Unicode Text with a Palindromic Tree](count-distinct-non-empty-palindromic-substrings-of-bounded-unicode-text-with-a-palindromic-tree.md)
- [Compute the Longest Palindromic Subsequence Length of Bounded Unicode Text](compute-the-longest-palindromic-subsequence-length-of-bounded-unicode-text.md)
<!-- catalog:related:end -->
