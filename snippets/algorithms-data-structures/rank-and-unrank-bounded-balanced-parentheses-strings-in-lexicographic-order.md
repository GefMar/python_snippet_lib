---
title: "Rank and Unrank Bounded Balanced-Parentheses Strings in Lexicographic Order"
snippet_type: algorithm
use_cases:
  - data-transformation
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - rank-and-unrank-index-combinations-in-itertools-combinations-order.md
  - rank-and-unrank-distinct-permutations-of-a-bounded-integer-multiset-in-lexicographic-order.md
  - return-the-next-lexicographic-permutation-of-bounded-integers.md
---

# Rank and Unrank Bounded Balanced-Parentheses Strings in Lexicographic Order

## Idea and Problem

Map every balanced-parentheses string of one bounded size to its zero-based lexicographic rank and recover the string at any valid rank.

A dynamic-programming table counts valid suffixes for each number of opening
and closing parentheses still available. While ranking, a closing parenthesis
skips every valid suffix that would have followed an opening parenthesis at
that position. Unranking compares the requested rank with the same skipped
block and chooses the next character without enumerating earlier strings.

## When to Use

Use these functions when balanced-parentheses words need compact stable
identifiers, deterministic test-case selection, or direct random-access
generation by ordinal. The contract uses one fixed pair count, zero-based
ranks, and ordinary ASCII lexicographic order, where `"(" < ")"`.

Use ordinary enumeration when every word must be visited. Use the combination
or permutation rankers for unconstrained selections and arrangements, and use
a grammar-specific parser when several bracket types or payload tokens are
part of the language.

## Implementation

```python
_MAX_BALANCED_PARENTHESES_PAIRS = 64


def _balanced_completion_counts(pair_count: int) -> list[list[int]]:
    counts = [[0] * (pair_count + 1) for _ in range(pair_count + 1)]
    counts[0][0] = 1

    for closes_remaining in range(pair_count + 1):
        for opens_remaining in range(pair_count + 1):
            if opens_remaining > closes_remaining:
                continue
            if opens_remaining == 0 and closes_remaining == 0:
                continue
            if opens_remaining:
                counts[opens_remaining][closes_remaining] += counts[
                    opens_remaining - 1
                ][closes_remaining]
            if closes_remaining > opens_remaining:
                counts[opens_remaining][closes_remaining] += counts[
                    opens_remaining
                ][closes_remaining - 1]
    return counts


def rank_balanced_parentheses(text: str) -> int:
    """Return the zero-based rank of one valid balanced-parentheses string."""
    if type(text) is not str:
        raise TypeError("text must be an exact string")
    if len(text) > 2 * _MAX_BALANCED_PARENTHESES_PAIRS:
        raise ValueError("text exceeds 64 parenthesis pairs")
    if len(text) % 2:
        raise ValueError("text length must be even")
    if any(character not in "()" for character in text):
        raise ValueError("text may contain only '(' and ')'")

    pair_count = len(text) // 2
    counts = _balanced_completion_counts(pair_count)
    opens_remaining = pair_count
    closes_remaining = pair_count
    rank = 0

    for character in text:
        if character == "(":
            if opens_remaining == 0:
                raise ValueError("text contains too many opening parentheses")
            opens_remaining -= 1
            continue

        if closes_remaining <= opens_remaining:
            raise ValueError("a text prefix closes before it opens")
        if opens_remaining:
            rank += counts[opens_remaining - 1][closes_remaining]
        closes_remaining -= 1

    if opens_remaining or closes_remaining:
        raise ValueError("text must have equal opening and closing counts")
    return rank


def unrank_balanced_parentheses(pair_count: int, rank: int) -> str:
    """Return the balanced-parentheses string at one zero-based rank."""
    if type(pair_count) is not int:
        raise TypeError("pair_count must be an exact integer")
    if not 0 <= pair_count <= _MAX_BALANCED_PARENTHESES_PAIRS:
        raise ValueError("pair_count is outside 0..64")
    if type(rank) is not int:
        raise TypeError("rank must be an exact integer")

    counts = _balanced_completion_counts(pair_count)
    total_count = counts[pair_count][pair_count]
    if not 0 <= rank < total_count:
        raise ValueError("rank is outside the balanced-parentheses space")

    opens_remaining = pair_count
    closes_remaining = pair_count
    characters: list[str] = []

    while opens_remaining or closes_remaining:
        open_count = (
            counts[opens_remaining - 1][closes_remaining]
            if opens_remaining
            else 0
        )
        if rank < open_count:
            characters.append("(")
            opens_remaining -= 1
            continue

        rank -= open_count
        if closes_remaining <= opens_remaining:
            raise AssertionError("a valid rank must admit an opening parenthesis")
        characters.append(")")
        closes_remaining -= 1

    return "".join(characters)
```

## Example

```python
def balanced_parentheses_by_filter(pair_count: int) -> tuple[str, ...]:
    from itertools import product

    words: list[str] = []
    for characters in product("()", repeat=2 * pair_count):
        balance = 0
        valid = True
        for character in characters:
            balance += 1 if character == "(" else -1
            if balance < 0:
                valid = False
                break
        if valid and balance == 0:
            words.append("".join(characters))
    return tuple(sorted(words))


def exercise_every_word_through_eight_pairs() -> int:
    from math import comb

    checked = 0
    for pair_count in range(9):
        words = balanced_parentheses_by_filter(pair_count)
        catalan_count = comb(2 * pair_count, pair_count) // (pair_count + 1)
        assert len(words) == catalan_count
        for rank, word in enumerate(words):
            assert rank_balanced_parentheses(word) == rank
            assert unrank_balanced_parentheses(pair_count, rank) == word
            checked += 1
    return checked


boundary_counts = _balanced_completion_counts(_MAX_BALANCED_PARENTHESES_PAIRS)
boundary_total = boundary_counts[-1][-1]
boundary_first = "(" * 64 + ")" * 64
boundary_last = "()" * 64


class TextSubclass(str):
    pass


rejected = 0
for invalid_text in (
    TextSubclass("()"),
    "(()",
    "())(",
    "(a)",
    "(" * 65 + ")" * 65,
):
    try:
        rank_balanced_parentheses(invalid_text)
    except (TypeError, ValueError):
        rejected += 1

for pair_count, rank in (
    (True, 0),
    (-1, 0),
    (65, 0),
    (1, True),
    (1, -1),
    (1, 1),
):
    try:
        unrank_balanced_parentheses(pair_count, rank)
    except (TypeError, ValueError):
        rejected += 1

assert (
    exercise_every_word_through_eight_pairs() == 2_056
    and rank_balanced_parentheses("") == 0
    and unrank_balanced_parentheses(0, 0) == ""
    and rank_balanced_parentheses(boundary_first) == 0
    and rank_balanced_parentheses(boundary_last) == boundary_total - 1
    and unrank_balanced_parentheses(64, 0) == boundary_first
    and unrank_balanced_parentheses(64, boundary_total - 1) == boundary_last
    and rejected == 11
)
```

## Trade-offs and Limitations

For `N` pairs, table construction performs `O(N**2)` exact-integer additions
and stores `O(N**2)` Python integer references. Ranking or unranking then
visits exactly `2 * N` characters. Catalan counts grow exponentially, so the
bit lengths and arithmetic cost of table entries and ranks are not constant.

Ranks are zero-based among strings with exactly the same pair count. The empty
string is the sole zero-pair word and has rank zero. Lexicographic order is the
ordinary Python string order for the two ASCII characters, so an opening
parenthesis precedes a closing parenthesis.

The functions reject malformed prefixes, unmatched parentheses, other
characters, subclasses and out-of-range ranks. They do not handle multiple
bracket types, quotient words by rotation, enumerate the language, sample
uniformly, rank strings of mixed sizes together, or retain a mutable index.

## Related Snippets

<!-- catalog:related:start -->
- [Rank and Unrank Index Combinations in itertools.combinations Order](rank-and-unrank-index-combinations-in-itertools-combinations-order.md)
- [Rank and Unrank Distinct Permutations of a Bounded Integer Multiset in Lexicographic Order](rank-and-unrank-distinct-permutations-of-a-bounded-integer-multiset-in-lexicographic-order.md)
- [Return the Next Lexicographic Permutation of Bounded Integers](return-the-next-lexicographic-permutation-of-bounded-integers.md)
<!-- catalog:related:end -->
