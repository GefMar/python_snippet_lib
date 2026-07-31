---
title: "Find the Lexicographically First Shortest Word Distinguishing Two States of a Bounded Complete DFA"
snippet_type: testing-technique
use_cases:
  - testing
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - ../algorithms-data-structures/minimize-a-bounded-complete-dfa-into-a-canonical-reachable-form.md
  - find-a-shortest-invariant-violation-in-a-bounded-deterministic-state-model.md
---

# Find the Lexicographically First Shortest Word Distinguishing Two States of a Bounded Complete DFA

## Idea and Problem

Produce the smallest reproducible input word that proves two states of one complete DFA accept different suffix languages.

Two states are immediately distinguishable when one is accepting and the other
is not, in which case the empty word is the witness. Otherwise, consuming one
symbol moves both states and creates another ordered state pair. Breadth-first
search over this product graph finds a shortest mismatch, while expanding the
declared alphabet in order selects the lexicographically first word among
equal lengths.

Remembering one predecessor per discovered pair reconstructs the witness
without storing a complete word in every queue entry. Exhausting at most
`states²` pairs proves language equivalence for the selected states.

## When to Use

Use this search in bounded automata tests when a Boolean inequality is not
enough and a compact counterexample should appear in a failure report. It is
especially useful when reviewing DFA transformations, minimizers, generated
transition tables, or state-merging decisions.

Use an automata library for partial or nondeterministic machines, epsilon
transitions, symbolic alphabets, regular-expression compilation, very large
state spaces, or minimization itself. This function compares any two declared
states, including states unreachable from some external start state.

## Implementation

```python
from collections import deque
from dataclasses import dataclass
from itertools import product
from random import Random

_MAX_WITNESS_STATES = 64
_MAX_WITNESS_SYMBOLS = 16

StatePair = tuple[int, int]


@dataclass(frozen=True, slots=True)
class DFAStateDistinction:
    word: tuple[str, ...] | None
    final_states: StatePair | None
    final_accepting: tuple[bool, bool] | None
    explored_pairs: int


def _validate_witness_dfa(
    alphabet: tuple[str, ...],
    transitions: tuple[tuple[int, ...], ...],
    accepting: tuple[bool, ...],
    left_state: int,
    right_state: int,
) -> None:
    if type(alphabet) is not tuple:
        raise TypeError("alphabet must be an exact tuple")
    if not 1 <= len(alphabet) <= _MAX_WITNESS_SYMBOLS:
        raise ValueError("alphabet size is outside 1..16")
    for symbol_index, symbol in enumerate(alphabet):
        if type(symbol) is not str:
            raise TypeError(f"alphabet[{symbol_index}] must be an exact string")
        if len(symbol) != 1 or not "!" <= symbol <= "~":
            raise ValueError("symbols must be printable non-space ASCII characters")
        if symbol_index and alphabet[symbol_index - 1] >= symbol:
            raise ValueError("alphabet must be strictly increasing")

    if type(transitions) is not tuple:
        raise TypeError("transitions must be an exact tuple")
    state_count = len(transitions)
    if not 1 <= state_count <= _MAX_WITNESS_STATES:
        raise ValueError("state count is outside 1..64")
    for state, row in enumerate(transitions):
        if type(row) is not tuple:
            raise TypeError(f"transitions[{state}] must be an exact tuple")
        if len(row) != len(alphabet):
            raise ValueError(f"transitions[{state}] must be complete")
        for symbol_index, target in enumerate(row):
            if type(target) is not int:
                raise TypeError(f"transitions[{state}][{symbol_index}] must be an exact integer")
            if not 0 <= target < state_count:
                raise ValueError(f"transitions[{state}][{symbol_index}] is out of range")

    if type(accepting) is not tuple:
        raise TypeError("accepting must be an exact tuple")
    if len(accepting) != state_count:
        raise ValueError("accepting must contain one flag per state")
    if any(type(flag) is not bool for flag in accepting):
        raise TypeError("accepting flags must be exact booleans")

    if type(left_state) is not int or type(right_state) is not int:
        raise TypeError("state indexes must be exact integers")
    if not 0 <= left_state < state_count or not 0 <= right_state < state_count:
        raise ValueError("state index is outside the declared state range")


def _reconstruct_word(
    pair: StatePair,
    predecessor: dict[StatePair, tuple[StatePair, str] | None],
) -> tuple[str, ...]:
    reversed_symbols: list[str] = []
    current = pair
    while predecessor[current] is not None:
        previous, symbol = predecessor[current]
        reversed_symbols.append(symbol)
        current = previous
    reversed_symbols.reverse()
    return tuple(reversed_symbols)


def shortest_dfa_state_distinguishing_word(
    alphabet: tuple[str, ...],
    transitions: tuple[tuple[int, ...], ...],
    accepting: tuple[bool, ...],
    left_state: int,
    right_state: int,
) -> DFAStateDistinction:
    """Return a shortlex-first witness, or evidence that the states agree."""
    _validate_witness_dfa(alphabet, transitions, accepting, left_state, right_state)
    initial = (left_state, right_state)
    if accepting[left_state] != accepting[right_state]:
        return DFAStateDistinction(
            word=(),
            final_states=initial,
            final_accepting=(accepting[left_state], accepting[right_state]),
            explored_pairs=1,
        )
    if left_state == right_state:
        return DFAStateDistinction(None, None, None, 1)

    predecessor: dict[StatePair, tuple[StatePair, str] | None] = {initial: None}
    pending = deque([initial])
    while pending:
        left, right = pending.popleft()
        for symbol_index, symbol in enumerate(alphabet):
            next_pair = (
                transitions[left][symbol_index],
                transitions[right][symbol_index],
            )
            if next_pair in predecessor:
                continue
            predecessor[next_pair] = ((left, right), symbol)
            if accepting[next_pair[0]] != accepting[next_pair[1]]:
                return DFAStateDistinction(
                    word=_reconstruct_word(next_pair, predecessor),
                    final_states=next_pair,
                    final_accepting=(accepting[next_pair[0]], accepting[next_pair[1]]),
                    explored_pairs=len(predecessor),
                )
            pending.append(next_pair)

    return DFAStateDistinction(None, None, None, len(predecessor))
```

## Example

```python

def accepts_from(
    transitions: tuple[tuple[int, ...], ...],
    accepting: tuple[bool, ...],
    state: int,
    word: tuple[int, ...],
) -> bool:
    for symbol_index in word:
        state = transitions[state][symbol_index]
    return accepting[state]


def shortlex_oracle(
    alphabet_size: int,
    transitions: tuple[tuple[int, ...], ...],
    accepting: tuple[bool, ...],
    left: int,
    right: int,
) -> tuple[int, ...] | None:
    for length in range(len(transitions) ** 2):
        for word in product(range(alphabet_size), repeat=length):
            if accepts_from(transitions, accepting, left, word) != accepts_from(
                transitions,
                accepting,
                right,
                word,
            ):
                return word
    return None


def equivalent_by_partition_refinement(
    transitions: tuple[tuple[int, ...], ...],
    accepting: tuple[bool, ...],
    left: int,
    right: int,
) -> bool:
    classes = tuple(int(flag) for flag in accepting)
    while True:
        ids: dict[tuple[int, ...], int] = {}
        refined: list[int] = []
        for state, row in enumerate(transitions):
            signature = (classes[state], *(classes[target] for target in row))
            refined.append(ids.setdefault(signature, len(ids)))
        next_classes = tuple(refined)
        if next_classes == classes:
            return classes[left] == classes[right]
        classes = next_classes


alphabet = ("a", "b")
transitions = (
    (1, 0),
    (1, 2),
    (2, 2),
)
accepting = (False, False, True)
assert shortest_dfa_state_distinguishing_word(
    alphabet,
    transitions,
    accepting,
    0,
    1,
) == DFAStateDistinction(
    word=("b",),
    final_states=(0, 2),
    final_accepting=(False, True),
    explored_pairs=3,
)
assert shortest_dfa_state_distinguishing_word(alphabet, transitions, accepting, 2, 2).word is None

rng = Random(0xDFA)
checked = 0
for _ in range(5_000):
    state_count = rng.randrange(1, 4)
    symbol_count = rng.randrange(1, 3)
    alphabet = tuple(chr(ord("a") + index) for index in range(symbol_count))
    transitions = tuple(
        tuple(rng.randrange(state_count) for _ in alphabet) for _ in range(state_count)
    )
    accepting = tuple(bool(rng.randrange(2)) for _ in range(state_count))
    left = rng.randrange(state_count)
    right = rng.randrange(state_count)
    expected_indexes = shortlex_oracle(
        symbol_count,
        transitions,
        accepting,
        left,
        right,
    )
    actual = shortest_dfa_state_distinguishing_word(
        alphabet,
        transitions,
        accepting,
        left,
        right,
    )
    expected_word = (
        None if expected_indexes is None else tuple(alphabet[index] for index in expected_indexes)
    )
    assert actual.word == expected_word
    if actual.word is None:
        assert equivalent_by_partition_refinement(transitions, accepting, left, right)
    checked += 1

assert checked == 5_000
```

## Trade-offs and Limitations

For `S` states and `A` symbols, at most `S²` ordered pairs are discovered.
Search takes `O(S² * A)` time and `O(S²)` predecessor and queue storage. The
64-state, 16-symbol limits keep both the proof of equivalence and witness
reconstruction bounded.

The empty tuple is a real witness when the selected states initially disagree.
`word=None` means no suffix word distinguishes them, not that search stopped
early. Alphabet ordering is part of the contract: shortest length wins first,
then ordinary tuple order over the declared symbols.

This routine validates one complete deterministic automaton but does not prune
unreachable states, minimize it, compile regular expressions, handle partial
transitions, or generalize to NFAs and epsilon transitions.

## Related Snippets

<!-- catalog:related:start -->
- [Minimize a Bounded Complete DFA into a Canonical Reachable Form](../algorithms-data-structures/minimize-a-bounded-complete-dfa-into-a-canonical-reachable-form.md)
- [Find a Shortest Invariant Violation in a Bounded Deterministic State Model](find-a-shortest-invariant-violation-in-a-bounded-deterministic-state-model.md)
<!-- catalog:related:end -->
