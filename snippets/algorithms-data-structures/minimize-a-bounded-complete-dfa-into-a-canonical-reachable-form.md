---
title: "Minimize a Bounded Complete DFA into a Canonical Reachable Form"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - testing
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - compute-the-reflexive-transitive-closure-of-a-bounded-directed-graph-with-integer-bitsets.md
  - partition-a-bounded-directed-graph-into-deterministic-strongly-connected-components.md
  - ../testing-tooling/find-a-shortest-invariant-violation-in-a-bounded-deterministic-state-model.md
---

# Minimize a Bounded Complete DFA into a Canonical Reachable Form

## Idea and Problem

Reduce a complete deterministic finite automaton to one canonical representation of its reachable language.

First discard states that cannot be reached from the declared start. Moore
refinement then repeatedly separates states whose acceptance or symbol
transitions can be distinguished. The stable blocks are exactly the
language-equivalence classes, so their quotient is minimal.

Minimal state identifiers can still depend on input numbering. A final
breadth-first traversal of the quotient visits symbols in alphabet order and
therefore numbers states by their shortlex-smallest access words. State
renaming, unreachable junk, and duplicated equivalent states cannot affect the
returned representation.

## When to Use

Use this algorithm for one small, fully materialized, complete DFA when stable
structural equality is useful for fixtures, generated-state review, cache keys,
or reference tests. Symbols must already be a closed sorted ASCII alphabet,
and transition rows must contain exactly one target for every symbol.

Use an automata library for regular-expression compilation, partial or
nondeterministic machines, epsilon transitions, symbolic alphabets, very large
state sets, or repeated incremental minimization. Preserve a separate mapping
outside this function when callers must relate canonical states back to source
state names.

## Implementation

```python
from collections import deque
from dataclasses import dataclass

_MAX_DFA_STATES = 64
_MAX_DFA_SYMBOLS = 16


@dataclass(frozen=True, slots=True)
class CanonicalDFA:
    alphabet: tuple[str, ...]
    transitions: tuple[tuple[int, ...], ...]
    accepting: tuple[bool, ...]


def _validate_complete_dfa(
    alphabet: object,
    transitions: object,
    accepting: object,
    start_state: object,
) -> None:
    if type(alphabet) is not tuple:
        raise TypeError("alphabet must be an exact tuple")
    if not 1 <= len(alphabet) <= _MAX_DFA_SYMBOLS:
        raise ValueError("alphabet size is outside the supported range")
    for index, symbol in enumerate(alphabet):
        if type(symbol) is not str:
            raise TypeError(f"alphabet[{index}] must be an exact string")
        if len(symbol) != 1 or not "!" <= symbol <= "~":
            raise ValueError(f"alphabet[{index}] must be one printable non-space ASCII symbol")
        if index and alphabet[index - 1] >= symbol:
            raise ValueError("alphabet must be strictly increasing")

    if type(transitions) is not tuple:
        raise TypeError("transitions must be an exact tuple")
    state_count = len(transitions)
    if not 1 <= state_count <= _MAX_DFA_STATES:
        raise ValueError("state count is outside the supported range")

    if type(accepting) is not tuple:
        raise TypeError("accepting must be an exact tuple")
    if len(accepting) != state_count:
        raise ValueError("accepting must contain one flag per state")
    for state, flag in enumerate(accepting):
        if type(flag) is not bool:
            raise TypeError(f"accepting[{state}] must be an exact boolean")

    if type(start_state) is not int:
        raise TypeError("start_state must be an exact integer")
    if not 0 <= start_state < state_count:
        raise ValueError("start_state is outside the state range")

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


def _reachable_states(
    transitions: tuple[tuple[int, ...], ...],
    start_state: int,
) -> tuple[int, ...]:
    seen = [False] * len(transitions)
    seen[start_state] = True
    queue = deque([start_state])
    ordered: list[int] = []

    while queue:
        state = queue.popleft()
        ordered.append(state)
        for target in transitions[state]:
            if not seen[target]:
                seen[target] = True
                queue.append(target)
    return tuple(ordered)


def _refine_blocks(
    transitions: tuple[tuple[int, ...], ...],
    accepting: tuple[bool, ...],
    reachable: tuple[int, ...],
) -> tuple[tuple[int, ...], int]:
    block_by_state = [-1] * len(transitions)
    for state in reachable:
        block_by_state[state] = int(accepting[state])

    while True:
        block_by_signature: dict[tuple[int, ...], int] = {}
        next_block_by_state = [-1] * len(transitions)
        for state in reachable:
            signature = (
                block_by_state[state],
                *(block_by_state[target] for target in transitions[state]),
            )
            block = block_by_signature.get(signature)
            if block is None:
                block = len(block_by_signature)
                block_by_signature[signature] = block
            next_block_by_state[state] = block

        if all(next_block_by_state[state] == block_by_state[state] for state in reachable):
            return tuple(next_block_by_state), len(block_by_signature)
        block_by_state = next_block_by_state


def minimize_complete_dfa(
    alphabet: tuple[str, ...],
    transitions: tuple[tuple[int, ...], ...],
    accepting: tuple[bool, ...],
    *,
    start_state: int,
) -> CanonicalDFA:
    """Return the minimal reachable DFA in canonical state order."""
    _validate_complete_dfa(alphabet, transitions, accepting, start_state)
    reachable = _reachable_states(transitions, start_state)
    block_by_state, block_count = _refine_blocks(transitions, accepting, reachable)

    representatives = [-1] * block_count
    for state in reachable:
        block = block_by_state[state]
        if representatives[block] == -1:
            representatives[block] = state

    block_transitions = tuple(
        tuple(block_by_state[target] for target in transitions[representative])
        for representative in representatives
    )
    block_accepting = tuple(accepting[representative] for representative in representatives)

    start_block = block_by_state[start_state]
    canonical_by_block = [-1] * block_count
    canonical_by_block[start_block] = 0
    ordered_blocks = [start_block]
    for block in ordered_blocks:
        for target in block_transitions[block]:
            if canonical_by_block[target] == -1:
                canonical_by_block[target] = len(ordered_blocks)
                ordered_blocks.append(target)
    if len(ordered_blocks) != block_count:
        raise AssertionError("every quotient state must remain reachable")

    return CanonicalDFA(
        alphabet=alphabet,
        transitions=tuple(
            tuple(canonical_by_block[target] for target in block_transitions[block])
            for block in ordered_blocks
        ),
        accepting=tuple(block_accepting[block] for block in ordered_blocks),
    )
```

## Example

```python
def product_language_equivalent(
    transitions: tuple[tuple[int, ...], ...],
    accepting: tuple[bool, ...],
    start_state: int,
    canonical: CanonicalDFA,
) -> bool:
    pending = deque([(start_state, 0)])
    seen = {(start_state, 0)}
    while pending:
        left, right = pending.popleft()
        if accepting[left] != canonical.accepting[right]:
            return False
        for symbol_index in range(len(canonical.alphabet)):
            pair = (
                transitions[left][symbol_index],
                canonical.transitions[right][symbol_index],
            )
            if pair not in seen:
                seen.add(pair)
                pending.append(pair)
    return True


def states_are_distinguishable(dfa: CanonicalDFA, left: int, right: int) -> bool:
    pending = deque([(left, right)])
    seen = {(left, right)}
    while pending:
        first, second = pending.popleft()
        if dfa.accepting[first] != dfa.accepting[second]:
            return True
        for symbol_index in range(len(dfa.alphabet)):
            pair = (
                dfa.transitions[first][symbol_index],
                dfa.transitions[second][symbol_index],
            )
            if pair not in seen:
                seen.add(pair)
                pending.append(pair)
    return False


def is_pairwise_minimal(dfa: CanonicalDFA) -> bool:
    return all(
        states_are_distinguishable(dfa, left, right)
        for left in range(len(dfa.transitions))
        for right in range(left + 1, len(dfa.transitions))
    )


def rename_states(
    transitions: tuple[tuple[int, ...], ...],
    accepting: tuple[bool, ...],
    start_state: int,
    new_state_by_old: tuple[int, ...],
) -> tuple[tuple[tuple[int, ...], ...], tuple[bool, ...], int]:
    renamed_transitions: list[tuple[int, ...]] = [()] * len(transitions)
    renamed_accepting = [False] * len(transitions)
    for old_state, new_state in enumerate(new_state_by_old):
        renamed_transitions[new_state] = tuple(
            new_state_by_old[target] for target in transitions[old_state]
        )
        renamed_accepting[new_state] = accepting[old_state]
    return (
        tuple(renamed_transitions),
        tuple(renamed_accepting),
        new_state_by_old[start_state],
    )


def verify_every_two_state_dfa() -> int:
    from itertools import product

    alphabet = ("a", "b")
    checked = 0
    for targets in product(range(2), repeat=4):
        transitions = (targets[:2], targets[2:])
        for accepting in product((False, True), repeat=2):
            for start_state in range(2):
                canonical = minimize_complete_dfa(
                    alphabet,
                    transitions,
                    accepting,
                    start_state=start_state,
                )
                assert product_language_equivalent(
                    transitions,
                    accepting,
                    start_state,
                    canonical,
                )
                assert is_pairwise_minimal(canonical)

                renamed_transitions, renamed_accepting, renamed_start = rename_states(
                    transitions,
                    accepting,
                    start_state,
                    (1, 0),
                )
                assert (
                    minimize_complete_dfa(
                        alphabet,
                        renamed_transitions,
                        renamed_accepting,
                        start_state=renamed_start,
                    )
                    == canonical
                )
                checked += 1
    return checked


def every_state_renaming(state_count: int):
    from itertools import permutations

    return permutations(range(state_count))


alphabet = ("0", "1")
base = minimize_complete_dfa(
    alphabet,
    ((0, 1), (0, 1)),
    (False, True),
    start_state=0,
)
extended_transitions = (
    (0, 1),
    (2, 1),
    (0, 1),
    (3, 3),
)
extended_accepting = (False, True, False, True)
extended = minimize_complete_dfa(
    alphabet,
    extended_transitions,
    extended_accepting,
    start_state=0,
)

renamed_variants_match = all(
    minimize_complete_dfa(
        alphabet,
        renamed_transitions,
        renamed_accepting,
        start_state=renamed_start,
    )
    == base
    for renamed_transitions, renamed_accepting, renamed_start in (
        rename_states(
            extended_transitions,
            extended_accepting,
            0,
            new_state_by_old,
        )
        for new_state_by_old in every_state_renaming(4)
    )
)

maximum_alphabet = tuple(chr(ord("!") + index) for index in range(_MAX_DFA_SYMBOLS))
maximum_transitions = tuple(
    tuple((state + symbol_index) % _MAX_DFA_STATES for symbol_index in range(_MAX_DFA_SYMBOLS))
    for state in range(_MAX_DFA_STATES)
)
maximum = minimize_complete_dfa(
    maximum_alphabet,
    maximum_transitions,
    (False,) * _MAX_DFA_STATES,
    start_state=0,
)


def rejected(
    candidate_alphabet: object,
    candidate_transitions: object,
    candidate_accepting: object,
    candidate_start: object,
) -> bool:
    try:
        minimize_complete_dfa(
            candidate_alphabet,
            candidate_transitions,
            candidate_accepting,
            start_state=candidate_start,
        )
    except (TypeError, ValueError):
        return True
    return False


invalid_calls = (
    (("b", "a"), ((0, 0),), (False,), 0),
    (("a",), ((0, 0),), (False,), 0),
    (("a",), ((True,),), (False,), 0),
    (("a",), ((0,),), (False,), True),
    (("a",), tuple((0,) for _ in range(_MAX_DFA_STATES + 1)), (False,) * 65, 0),
)

assert (
    base,
    extended,
    renamed_variants_match,
    verify_every_two_state_dfa(),
    maximum,
    sum(rejected(*call) for call in invalid_calls),
) == (
    CanonicalDFA(alphabet, ((0, 1), (0, 1)), (False, True)),
    base,
    True,
    128,
    CanonicalDFA(maximum_alphabet, ((0,) * _MAX_DFA_SYMBOLS,), (False,)),
    len(invalid_calls),
)
```

## Trade-offs and Limitations

For `n` reachable states and `a` symbols, each refinement round builds `O(n)`
signatures of length `a + 1`. At most `n` rounds are needed, giving expected
`O(a * n**2)` dictionary work and `O(a * n)` working references. Reachability,
quotient construction, and canonical breadth-first numbering each take
`O(a * n)` time. The input and returned transition matrices occupy
`O(a * n)` references.

Refinement begins with acceptance of the empty word. After each round, states
remain together only when they also agree after one more leading symbol, so a
stable partition represents equality on every input word. The final ordered
breadth-first traversal assigns identifiers by shortlex-smallest access words;
it does not expose input state identifiers or a projection back to them.

The alphabet order is part of the contract and must already be canonical. The
function does not infer an implicit rejecting sink, preserve unreachable
states, return distinguishing words, accept empty alphabets, minimize
nondeterministic or weighted machines, parse regular expressions, generate
code, or update an existing quotient incrementally.

## Related Snippets

<!-- catalog:related:start -->
- [Compute the Reflexive Transitive Closure of a Bounded Directed Graph with Integer Bitsets](compute-the-reflexive-transitive-closure-of-a-bounded-directed-graph-with-integer-bitsets.md)
- [Partition a Bounded Directed Graph into Deterministic Strongly Connected Components](partition-a-bounded-directed-graph-into-deterministic-strongly-connected-components.md)
- [Find a Shortest Invariant Violation in a Bounded Deterministic State Model](../testing-tooling/find-a-shortest-invariant-violation-in-a-bounded-deterministic-state-model.md)
<!-- catalog:related:end -->
