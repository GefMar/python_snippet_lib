---
title: "Generate a Minimal Cyclic Test Sequence Covering Every Fixed-Length Symbol Word"
snippet_type: testing-technique
use_cases:
  - data-transformation
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - audit-a-bounded-test-matrix-for-complete-pairwise-coverage.md
  - ../algorithms-data-structures/rank-and-unrank-index-combinations-in-itertools-combinations-order.md
  - ../algorithms-data-structures/find-the-earliest-lexicographically-smallest-rotation-of-bounded-unicode-text-with-booths-algorithm.md
---

# Generate a Minimal Cyclic Test Sequence Covering Every Fixed-Length Symbol Word

## Idea and Problem

Generate the lexicographically smallest shortest cycle whose fixed-length windows cover every word over one bounded ordered alphabet.

A cycle of length `k**order` has exactly as many starting positions as there
are possible words of that length. The prefer-min Fredricksen-Kessler-Maiorana
construction concatenates ordered aperiodic prefixes so every word occurs once
without searching the exponentially larger space of candidate cycles.

## When to Use

Use this construction when a cyclic test driver must exercise every possible
fixed-length history over a small closed alphabet. It can compactly cover state
machine inputs, rolling decoders, or transition fixtures when wraparound is an
intentional part of the test contract. Declared alphabet order, rather than
Unicode code-point order, determines the canonical result.

Use an ordinary Cartesian product when each word must be an independent test
case. Use a constraint-aware generator when some adjacent symbols are invalid,
and use randomized generation when exhaustive fixed-order coverage is not the
goal.

## Implementation

```python
_MAX_ALPHABET_SYMBOLS = 32
_MAX_WORD_ORDER = 12
_MAX_CYCLE_LENGTH = 100_000


def generate_minimal_cyclic_word_cover(
    alphabet: tuple[str, ...],
    order: int,
) -> tuple[str, ...]:
    """Return the canonical shortest cycle covering every word once."""
    if type(alphabet) is not tuple:
        raise TypeError("alphabet must be an exact tuple")
    if not 1 <= len(alphabet) <= _MAX_ALPHABET_SYMBOLS:
        raise ValueError("alphabet size is outside the supported range")

    seen: set[str] = set()
    for index, symbol in enumerate(alphabet):
        if type(symbol) is not str:
            raise TypeError(f"alphabet[{index}] must be an exact string")
        if len(symbol) != 1:
            raise ValueError(f"alphabet[{index}] must contain one code point")
        try:
            symbol.encode("utf-8")
        except UnicodeEncodeError:
            raise ValueError(f"alphabet[{index}] must be valid UTF-8 text") from None
        if symbol in seen:
            raise ValueError(f"alphabet[{index}] is duplicated")
        seen.add(symbol)

    if type(order) is not int:
        raise TypeError("order must be an exact non-boolean integer")
    if not 1 <= order <= _MAX_WORD_ORDER:
        raise ValueError("order is outside the supported range")

    cycle_length = len(alphabet) ** order
    if cycle_length > _MAX_CYCLE_LENGTH:
        raise ValueError("cycle length exceeds the supported limit")

    work = [0] * (order + 1)
    cycle_indices: list[int] = []

    def visit(position: int, period: int) -> None:
        if position > order:
            if order % period == 0:
                cycle_indices.extend(work[1 : period + 1])
            return

        work[position] = work[position - period]
        visit(position + 1, period)
        for symbol_index in range(work[position - period] + 1, len(alphabet)):
            work[position] = symbol_index
            visit(position + 1, position)

    visit(1, 1)
    return tuple(alphabet[index] for index in cycle_indices)
```

## Example

```python
def cyclic_words(
    sequence: tuple[str, ...],
    order: int,
) -> tuple[tuple[str, ...], ...]:
    return tuple(
        tuple(sequence[(start + offset) % len(sequence)] for offset in range(order))
        for start in range(len(sequence))
    )


def cartesian_words(
    alphabet: tuple[str, ...],
    order: int,
) -> set[tuple[str, ...]]:
    from itertools import product

    return set(product(alphabet, repeat=order))


def brute_minimal_cycle(
    alphabet: tuple[str, ...],
    order: int,
) -> tuple[str, ...]:
    from itertools import product

    expected_words = cartesian_words(alphabet, order)
    cycle_length = len(alphabet) ** order
    alphabet_positions = {symbol: index for index, symbol in enumerate(alphabet)}

    valid_cycles = (
        candidate
        for candidate in product(alphabet, repeat=cycle_length)
        if set(cyclic_words(candidate, order)) == expected_words
    )
    return min(
        valid_cycles,
        key=lambda candidate: tuple(alphabet_positions[symbol] for symbol in candidate),
    )


declared_binary = ("b", "a")
binary = generate_minimal_cyclic_word_cover(declared_binary, 3)
ternary = generate_minimal_cyclic_word_cover(("a", "b", "c"), 2)
unary = generate_minimal_cyclic_word_cover(("x",), 4)

try:
    generate_minimal_cyclic_word_cover(("x", "x"), 2)
except ValueError:
    duplicate_rejected = True
else:
    duplicate_rejected = False

assert (
    binary,
    binary == brute_minimal_cycle(declared_binary, 3),
    ternary,
    ternary == brute_minimal_cycle(("a", "b", "c"), 2),
    unary,
    set(cyclic_words(binary, 3)) == cartesian_words(declared_binary, 3),
    duplicate_rejected,
) == (
    ("b", "b", "b", "a", "b", "a", "a", "a"),
    True,
    ("a", "a", "b", "a", "c", "b", "b", "c", "c"),
    True,
    ("x",),
    True,
    True,
)
```

## Trade-offs and Limitations

For alphabet size `k`, the returned cycle contains exactly `k**order` symbol
references. Validation and prefer-min generation take `O(order + k**order)`
time. The work array and recursion use `O(order)` state, within the declared
`O(k * order + order)` bound, while the temporary index list and returned tuple
use output-proportional memory and briefly coexist.

Output grows exponentially even though generation has constant amortized work
per emitted symbol. The explicit cap therefore matters more than the linear
generation bound. A one-symbol alphabet returns one symbol at every order; its
longer words are observed by repeatedly wrapping over that single position.

Symbols are exact, normalization-sensitive Unicode code points. The result is
cyclic and does not append the first `order - 1` symbols for linear scanning.
It does not support constrained words, random generation, configurable cycle
orientation, approximate coverage, weighted symbols, or unbounded spaces.

## Related Snippets

<!-- catalog:related:start -->
- [Audit a Bounded Test Matrix for Complete Pairwise Coverage](audit-a-bounded-test-matrix-for-complete-pairwise-coverage.md)
- [Rank and Unrank Index Combinations in itertools.combinations Order](../algorithms-data-structures/rank-and-unrank-index-combinations-in-itertools-combinations-order.md)
- [Find the Earliest Lexicographically Smallest Rotation of Bounded Unicode Text with Booth's Algorithm](../algorithms-data-structures/find-the-earliest-lexicographically-smallest-rotation-of-bounded-unicode-text-with-booths-algorithm.md)
<!-- catalog:related:end -->
