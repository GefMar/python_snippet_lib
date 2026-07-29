---
title: "Find All Exact Multi-Pattern Text Matches with Aho-Corasick"
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
  - build-a-bounded-immutable-text-trie-for-longest-prefix-lookup.md
  - find-every-anagram-window-in-bounded-unicode-text.md
---

# Find All Exact Multi-Pattern Text Matches with Aho-Corasick

## Idea and Problem

Find every overlapping occurrence of several exact patterns in one bounded text without running an independent search for each pattern.

Aho-Corasick stores the patterns in one trie. Failure links preserve the
longest useful suffix after a transition is missing, while output links visit
shorter terminal suffixes without scanning unrelated trie ancestors. One pass
through the text can therefore report matches for patterns that overlap or end
at the same position.

Each result keeps the pattern's declaration index. Results are ordered by
exclusive stop position and then start position, so longer matching suffixes at
one stop appear first and the output is independent of dictionary iteration.

## When to Use

Use this algorithm when a fixed, bounded set of distinct literal patterns must
be searched together in one already decoded string, every overlap matters, and
the caller needs to identify which declared pattern matched. It is especially
useful when the combined trie shares prefixes or the text is large enough that
restarting a single-pattern matcher for every pattern is undesirable.

Use repeated `str.find()` or the single-pattern KMP snippet when there are only
one or two needles and simplicity matters more than shared scanning. Choose a
regular-expression engine for pattern syntax, or a streaming implementation
when the complete text is not available at once.

## Implementation

```python
from collections import deque

_MAX_TEXT_CODE_POINTS = 65_536
_MAX_TEXT_UTF8_BYTES = 262_144
_MAX_PATTERN_COUNT = 4_096
_MAX_MATCH_COUNT = 100_000
_NO_OUTPUT = -1


def _validated_aho_text(value: object, *, field: str, allow_empty: bool) -> tuple[str, int]:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if not allow_empty and not value:
        raise ValueError(f"{field} must not be empty")
    if len(value) > _MAX_TEXT_CODE_POINTS:
        raise ValueError(f"{field} exceeds the code-point limit")
    try:
        encoded_size = len(value.encode("utf-8"))
    except UnicodeEncodeError:
        raise ValueError(f"{field} must be valid UTF-8 text") from None
    if encoded_size > _MAX_TEXT_UTF8_BYTES:
        raise ValueError(f"{field} exceeds the UTF-8 byte limit")
    return value, encoded_size


def find_multi_pattern_matches(
    text: str,
    patterns: tuple[str, ...],
) -> tuple[tuple[int, int, int], ...]:
    """Return exact (start, stop, pattern_index) matches in stable order."""
    checked_text, _ = _validated_aho_text(text, field="text", allow_empty=True)
    if type(patterns) is not tuple:
        raise TypeError("patterns must be an exact tuple")
    if not 1 <= len(patterns) <= _MAX_PATTERN_COUNT:
        raise ValueError("pattern count is outside the supported range")

    checked_patterns: list[str] = []
    seen_patterns: set[str] = set()
    total_code_points = 0
    total_utf8_bytes = 0
    for index, pattern in enumerate(patterns):
        checked, encoded_size = _validated_aho_text(
            pattern,
            field=f"patterns[{index}]",
            allow_empty=False,
        )
        if checked in seen_patterns:
            raise ValueError(f"patterns[{index}] duplicates an earlier pattern")
        seen_patterns.add(checked)
        checked_patterns.append(checked)
        total_code_points += len(checked)
        total_utf8_bytes += encoded_size
        if total_code_points > _MAX_TEXT_CODE_POINTS:
            raise ValueError("combined patterns exceed the code-point limit")
        if total_utf8_bytes > _MAX_TEXT_UTF8_BYTES:
            raise ValueError("combined patterns exceed the UTF-8 byte limit")

    transitions: list[dict[str, int]] = [{}]
    failures = [0]
    output_links = [_NO_OUTPUT]
    terminal_patterns = [_NO_OUTPUT]

    for pattern_index, pattern in enumerate(checked_patterns):
        state = 0
        for character in pattern:
            next_state = transitions[state].get(character)
            if next_state is None:
                next_state = len(transitions)
                transitions[state][character] = next_state
                transitions.append({})
                failures.append(0)
                output_links.append(_NO_OUTPUT)
                terminal_patterns.append(_NO_OUTPUT)
            state = next_state
        terminal_patterns[state] = pattern_index

    pending: deque[int] = deque(transitions[0].values())
    while pending:
        state = pending.popleft()
        for character, next_state in transitions[state].items():
            fallback = failures[state]
            while fallback and character not in transitions[fallback]:
                fallback = failures[fallback]
            failures[next_state] = transitions[fallback].get(character, 0)

            failure_state = failures[next_state]
            if terminal_patterns[failure_state] != _NO_OUTPUT:
                output_links[next_state] = failure_state
            else:
                output_links[next_state] = output_links[failure_state]
            pending.append(next_state)

    matches: list[tuple[int, int, int]] = []
    state = 0
    for stop, character in enumerate(checked_text, start=1):
        while state and character not in transitions[state]:
            state = failures[state]
        state = transitions[state].get(character, 0)

        output_state = state
        if terminal_patterns[output_state] == _NO_OUTPUT:
            output_state = output_links[output_state]
        while output_state != _NO_OUTPUT:
            pattern_index = terminal_patterns[output_state]
            start = stop - len(checked_patterns[pattern_index])
            if len(matches) == _MAX_MATCH_COUNT:
                raise ValueError("match count exceeds the supported limit")
            matches.append((start, stop, pattern_index))
            output_state = output_links[output_state]

    return tuple(matches)
```

## Example

```python
def naive_multi_pattern_matches(
    text: str,
    patterns: tuple[str, ...],
) -> tuple[tuple[int, int, int], ...]:
    matches: list[tuple[int, int, int]] = []
    for stop in range(1, len(text) + 1):
        at_stop: list[tuple[int, int, int]] = []
        for pattern_index, pattern in enumerate(patterns):
            start = stop - len(pattern)
            if start >= 0 and text.startswith(pattern, start):
                at_stop.append((start, stop, pattern_index))
        matches.extend(sorted(at_stop))
    return tuple(matches)


def exercise_small_multi_pattern_spaces() -> int:
    from itertools import combinations, product

    pool = tuple(
        "".join(characters) for length in range(1, 3) for characters in product("ab", repeat=length)
    )
    texts = tuple(
        "".join(characters) for length in range(6) for characters in product("ab", repeat=length)
    )

    checked = 0
    for pattern_count in range(1, 4):
        for selected in combinations(pool, pattern_count):
            declarations = (selected,) if pattern_count == 1 else (selected, selected[::-1])
            for patterns in declarations:
                for text in texts:
                    assert find_multi_pattern_matches(
                        text,
                        patterns,
                    ) == naive_multi_pattern_matches(text, patterns)
                    checked += 1
    return checked


ordinary = find_multi_pattern_matches("ababa", ("aba", "ba", "a"))
failure_chain = find_multi_pattern_matches(
    "ahishers",
    ("he", "she", "his", "hers"),
)

value_errors = 0
for invalid_patterns in ((), ("a", "a"), ("",)):
    try:
        find_multi_pattern_matches("text", invalid_patterns)
    except ValueError:
        value_errors += 1

try:
    find_multi_pattern_matches("a" * 50_001, ("a", "aa"))
except ValueError:
    value_errors += 1

type_errors = 0
try:
    find_multi_pattern_matches("text", ["text"])
except TypeError:
    type_errors += 1

try:
    find_multi_pattern_matches(123, ("123",))
except TypeError:
    type_errors += 1

assert (
    exercise_small_multi_pattern_spaces(),
    ordinary,
    failure_chain,
    value_errors,
    type_errors,
) == (
    4_788,
    (
        (0, 1, 2),
        (0, 3, 0),
        (1, 3, 1),
        (2, 3, 2),
        (2, 5, 0),
        (3, 5, 1),
        (4, 5, 2),
    ),
    ((1, 4, 2), (3, 6, 1), (4, 6, 0), (4, 8, 3)),
    4,
    2,
)
```

## Trade-offs and Limitations

The trie uses `O(P)` states for `P` combined pattern code points. With expected
constant-time dictionary access, text scanning is amortized `O(T + M)` for
`T` text code points and `M` emitted matches; output links avoid visiting
non-terminal suffixes. Sparse failure-link construction stores `O(P)` data but
can follow several failure links while assigning one trie edge, so its strict
worst case is weaker than a dense fixed-alphabet automaton.

The result itself uses `O(M)` memory and is deliberately capped. The complete
patterns, trie, text and results must coexist in memory. Rebuild the automaton
for every call only when reuse is unimportant; a dedicated object is more
appropriate when one pattern set serves many texts.

Matching is exact Python code-point equality. The function does not normalize,
case-fold, interpret grapheme clusters, support empty or duplicate patterns,
parse regular expressions, match approximately, update patterns dynamically,
or preserve streaming state between chunks. Invalid UTF-8 surrogate code
points are rejected.

## Related Snippets

<!-- catalog:related:start -->
- [Find All Overlapping Exact Text Matches with Knuth-Morris-Pratt](find-all-overlapping-exact-text-matches-with-knuth-morris-pratt.md)
- [Build a Bounded Immutable Text Trie for Longest-Prefix Lookup](build-a-bounded-immutable-text-trie-for-longest-prefix-lookup.md)
- [Find Every Anagram Window in Bounded Unicode Text](find-every-anagram-window-in-bounded-unicode-text.md)
<!-- catalog:related:end -->
