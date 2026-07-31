---
title: "Wrap Bounded ASCII Words with Minimum Squared Raggedness"
snippet_type: algorithm
use_cases:
  - data-transformation
  - testing
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - partition-bounded-non-negative-weights-into-exactly-k-contiguous-groups-with-minimum-peak-load.md
  - plan-a-minimum-cost-parenthesization-for-a-bounded-matrix-chain.md
  - ../data-processing/limit-text-lines-across-arbitrary-chunks.md
---

# Wrap Bounded ASCII Words with Minimum Squared Raggedness

## Idea and Problem

Choose line breaks globally instead of greedily when uneven non-final lines should carry an explicit squared penalty.

Each line joins a contiguous group of words with one ASCII space. Its unused
width is squared so a very short line costs more than several mildly uneven
lines with the same total slack. The final line is exempt, matching layouts
where filling the paragraph's last line is not desirable.

A suffix dynamic program compares every feasible next break. Equal penalties
prefer fewer lines and then the lexicographically smallest complete sequence of
exclusive break stops, making the layout reproducible.

## When to Use

Use this planner for bounded ASCII labels, generated reports, fixtures, or
other text whose tokenization is already fixed and where global raggedness is
more important than greedy filling. It is also useful as a reference oracle
for another wrapping implementation with the same objective.

Use `textwrap` for conventional prose policies, and use a display-width-aware
layout library for terminals, fonts, tabs, combining marks, or East Asian wide
characters. Use a richer typesetting algorithm when hyphenation, justification,
forced breaks, indentation, or stretchable spaces affect the objective.

## Implementation

```python
from dataclasses import dataclass

_MAX_WORD_COUNT = 256
_MAX_WORD_LENGTH = 64
_MAX_TOTAL_WORD_CHARACTERS = 4_096
_MAX_LINE_WIDTH = 256


@dataclass(frozen=True, slots=True)
class RaggedWrap:
    lines: tuple[str, ...]
    break_stops: tuple[int, ...]
    penalty: int


def minimum_raggedness_wrap(
    words: tuple[str, ...],
    *,
    line_width: int,
) -> RaggedWrap:
    """Return the canonical minimum-squared-raggedness wrapping."""
    if type(words) is not tuple:
        raise TypeError("words must be an exact tuple")
    if not 1 <= len(words) <= _MAX_WORD_COUNT:
        raise ValueError("word count is outside the supported range")
    if type(line_width) is not int:
        raise TypeError("line_width must be an exact non-boolean integer")
    if not 1 <= line_width <= _MAX_LINE_WIDTH:
        raise ValueError("line_width is outside the supported range")

    total_characters = 0
    for index, word in enumerate(words):
        if type(word) is not str:
            raise TypeError(f"words[{index}] must be an exact string")
        if not 1 <= len(word) <= _MAX_WORD_LENGTH:
            raise ValueError(f"words[{index}] length is outside the supported range")
        if any(not "!" <= character <= "~" for character in word):
            raise ValueError(f"words[{index}] must contain printable non-space ASCII")
        if len(word) > line_width:
            raise ValueError(f"words[{index}] does not fit within line_width")
        total_characters += len(word)
        if total_characters > _MAX_TOTAL_WORD_CHARACTERS:
            raise ValueError("total word characters exceed the supported limit")

    word_count = len(words)
    best: list[tuple[int, int, tuple[int, ...]] | None] = [None] * (word_count + 1)
    best[word_count] = (0, 0, ())

    for start in range(word_count - 1, -1, -1):
        occupied = 0
        for stop in range(start + 1, word_count + 1):
            occupied += len(words[stop - 1])
            if stop > start + 1:
                occupied += 1
            if occupied > line_width:
                break

            suffix = best[stop]
            if suffix is None:
                raise AssertionError("every validated suffix must be wrappable")
            unused = line_width - occupied
            line_penalty = 0 if stop == word_count else unused * unused
            candidate = (
                line_penalty + suffix[0],
                1 + suffix[1],
                (stop, *suffix[2]),
            )
            if best[start] is None or candidate < best[start]:
                best[start] = candidate

    optimum = best[0]
    if optimum is None:
        raise AssertionError("validated words must have a wrapping")
    break_stops = optimum[2]
    line_starts = (0, *break_stops[:-1])
    lines = tuple(
        " ".join(words[start:stop]) for start, stop in zip(line_starts, break_stops, strict=True)
    )
    return RaggedWrap(
        lines=lines,
        break_stops=break_stops,
        penalty=optimum[0],
    )
```

## Example

```python
def exhaustive_wrap_oracle(
    words: tuple[str, ...],
    line_width: int,
) -> RaggedWrap:
    candidates: list[tuple[int, int, tuple[int, ...], tuple[str, ...]]] = []
    for mask in range(1 << (len(words) - 1)):
        stops = (
            *(index + 1 for index in range(len(words) - 1) if mask & (1 << index)),
            len(words),
        )
        starts = (0, *stops[:-1])
        lines = tuple(
            " ".join(words[start:stop]) for start, stop in zip(starts, stops, strict=True)
        )
        if any(len(line) > line_width for line in lines):
            continue
        penalty = sum((line_width - len(line)) ** 2 for line in lines[:-1])
        candidates.append((penalty, len(lines), stops, lines))

    penalty, _, stops, lines = min(candidates)
    return RaggedWrap(lines=lines, break_stops=stops, penalty=penalty)


def verify_small_wraps() -> int:
    from itertools import product

    tokens = ("a", "bb", "ccc")
    checked = 0
    for word_count in range(1, 7):
        for words in product(tokens, repeat=word_count):
            for width in range(1, 9):
                if max(map(len, words)) > width:
                    continue
                assert minimum_raggedness_wrap(
                    words,
                    line_width=width,
                ) == exhaustive_wrap_oracle(words, width)
                checked += 1
    return checked


wrapped = minimum_raggedness_wrap(
    ("alpha", "beta", "gamma", "delta"),
    line_width=11,
)
exact_fit = minimum_raggedness_wrap(("aa", "bb", "cc"), line_width=5)


def rejected(words: object, width: object) -> bool:
    try:
        minimum_raggedness_wrap(words, line_width=width)
    except (TypeError, ValueError):
        return True
    return False


invalid_calls = (
    ((), 4),
    (("long",), 3),
    (("two words",), 10),
    (("café",), 8),
    (("ok",), True),
)

assert (
    wrapped,
    exact_fit,
    verify_small_wraps(),
    sum(rejected(words, width) for words, width in invalid_calls),
) == (
    RaggedWrap(("alpha beta", "gamma delta"), (2, 4), 1),
    RaggedWrap(("aa bb", "cc"), (2, 3), 0),
    6_684,
    5,
)
```

## Trade-offs and Limitations

For `n` words, the dynamic program evaluates `O(n**2)` candidate lines. Each
candidate stores and compares a break-stop witness of length at most `n`, so
the straightforward bounded implementation uses `O(n**3)` worst-case tuple
construction/comparison work and `O(n**2)` retained index references. The
256-word cap keeps that explicit trade-off predictable.

Squared slack penalizes conspicuously short non-final lines, but it is only one
layout policy. The final line always contributes zero. Equal penalties prefer
fewer lines, then earlier break stops by tuple order. Returned lines contain
words joined by one ASCII space; the input tuple and its strings are not
mutated.

Width means printable ASCII characters, not bytes in an arbitrary encoding or
display cells in a terminal or font. The function does not tokenize a
paragraph, retain caller whitespace, split a word, hyphenate, justify, indent,
accept forced breaks, stream output, or optimize any alternative penalty.

## Related Snippets

<!-- catalog:related:start -->
- [Partition Bounded Non-Negative Weights into Exact-K Contiguous Groups by Minimum Peak Load](partition-bounded-non-negative-weights-into-exactly-k-contiguous-groups-with-minimum-peak-load.md)
- [Plan a Minimum-Cost Parenthesization for a Bounded Matrix Chain](plan-a-minimum-cost-parenthesization-for-a-bounded-matrix-chain.md)
- [Limit Text Lines Across Arbitrary Chunks](../data-processing/limit-text-lines-across-arbitrary-chunks.md)
<!-- catalog:related:end -->
