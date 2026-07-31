---
title: "Determinize a Bounded Epsilon-NFA into a Canonical Complete DFA Under a State Cap"
snippet_type: algorithm
use_cases:
  - data-transformation
  - parsing
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - minimize-a-bounded-complete-dfa-into-a-canonical-reachable-form.md
  - ../testing-tooling/find-the-lexicographically-first-shortest-word-distinguishing-two-states-of-a-bounded-complete-dfa.md
  - find-a-canonical-cyk-parse-witness-for-bounded-tokens-under-a-strict-cnf-grammar.md
---

# Determinize a Bounded Epsilon-NFA into a Canonical Complete DFA Under a State Cap

## Idea and Problem

Convert one small nondeterministic finite automaton into an equivalent complete deterministic automaton without hiding the exponential subset expansion.

Each DFA state represents the epsilon closure of a set of NFA states. Starting
from the closure of the declared start state, subset construction follows every
symbol and closes the result again. Breadth-first discovery and declared symbol
order give every reachable subset a reproducible integer ID.

A reachable empty subset is retained as the rejecting sink that makes the
transition table complete. Discovery stops with a dedicated error before a
1,025th DFA state can escape the configured bound.

## When to Use

Use this construction for bounded automata when deterministic transition
tables simplify simulation, validation, or a later minimization step. The
canonical state IDs are helpful for stable fixtures and independent structural
comparisons.

Do not use it as an unbounded regular-expression engine. NFA determinization
can produce exponentially many subsets even when the input graph is small.
Keep an NFA simulator when only a few words need evaluation, and apply a
separate algorithm when DFA minimization is required.

## Implementation

```python
from dataclasses import dataclass
from itertools import product

_MAX_NFA_STATES = 32
_MAX_NFA_SYMBOLS = 8
_MAX_NFA_TRANSITIONS = 1_024
_MAX_REACHABLE_DFA_STATES = 1_024
_MAX_SYMBOL_BYTES = 32

NfaTransition = tuple[int, str | None, int]


@dataclass(frozen=True, slots=True)
class CanonicalDfa:
    alphabet: tuple[str, ...]
    state_subsets: tuple[tuple[int, ...], ...]
    transitions: tuple[tuple[int, ...], ...]
    accepting_states: tuple[int, ...]


class DfaStateLimitError(ValueError):
    def __init__(self, limit: int) -> None:
        self.limit = limit
        super().__init__(f"determinization exceeds {limit} reachable DFA states")


def determinize_epsilon_nfa(
    state_count: int,
    alphabet: tuple[str, ...],
    start_state: int,
    accepting_states: tuple[int, ...],
    transitions: tuple[NfaTransition, ...],
) -> CanonicalDfa:
    """Return the canonical reachable complete DFA under the fixed state cap."""
    if type(state_count) is not int:
        raise TypeError("state_count must be an exact integer")
    if not 1 <= state_count <= _MAX_NFA_STATES:
        raise ValueError("state_count is outside 1..32")
    if type(alphabet) is not tuple:
        raise TypeError("alphabet must be an exact tuple")
    if not 1 <= len(alphabet) <= _MAX_NFA_SYMBOLS:
        raise ValueError("alphabet size is outside 1..8")

    symbol_indexes: dict[str, int] = {}
    for symbol_index, symbol in enumerate(alphabet):
        if type(symbol) is not str:
            raise TypeError(f"alphabet[{symbol_index}] must be an exact string")
        try:
            encoded_symbol = symbol.encode("utf-8")
        except UnicodeEncodeError:
            raise ValueError(f"alphabet[{symbol_index}] must be valid UTF-8") from None
        if not 1 <= len(encoded_symbol) <= _MAX_SYMBOL_BYTES:
            raise ValueError(f"alphabet[{symbol_index}] is outside the byte limit")
        if symbol in symbol_indexes:
            raise ValueError(f"alphabet[{symbol_index}] duplicates an earlier symbol")
        symbol_indexes[symbol] = symbol_index

    if type(start_state) is not int:
        raise TypeError("start_state must be an exact integer")
    if not 0 <= start_state < state_count:
        raise ValueError("start_state is outside the NFA")
    if type(accepting_states) is not tuple:
        raise TypeError("accepting_states must be an exact tuple")
    accepting_mask = 0
    for index, state in enumerate(accepting_states):
        if type(state) is not int:
            raise TypeError(f"accepting_states[{index}] must be an exact integer")
        if not 0 <= state < state_count:
            raise ValueError(f"accepting_states[{index}] is outside the NFA")
        state_bit = 1 << state
        if accepting_mask & state_bit:
            raise ValueError(f"accepting_states[{index}] is duplicated")
        accepting_mask |= state_bit

    if type(transitions) is not tuple:
        raise TypeError("transitions must be an exact tuple")
    if len(transitions) > _MAX_NFA_TRANSITIONS:
        raise ValueError("transition count exceeds 1,024")

    epsilon_targets = [0] * state_count
    symbol_targets = [[0] * len(alphabet) for _ in range(state_count)]
    seen_transitions: set[NfaTransition] = set()
    for index, transition in enumerate(transitions):
        if type(transition) is not tuple or len(transition) != 3:
            raise TypeError(f"transitions[{index}] must be an exact triple")
        source, symbol, target = transition
        if type(source) is not int or type(target) is not int:
            raise TypeError(f"transitions[{index}] endpoints must be exact integers")
        if not 0 <= source < state_count or not 0 <= target < state_count:
            raise ValueError(f"transitions[{index}] endpoint is outside the NFA")
        if symbol is not None and type(symbol) is not str:
            raise TypeError(f"transitions[{index}] symbol must be a string or None")
        if symbol is not None and symbol not in symbol_indexes:
            raise ValueError(f"transitions[{index}] uses an undeclared symbol")
        if transition in seen_transitions:
            raise ValueError(f"transitions[{index}] is duplicated")
        seen_transitions.add(transition)

        if symbol is None:
            epsilon_targets[source] |= 1 << target
        else:
            symbol_targets[source][symbol_indexes[symbol]] |= 1 << target

    epsilon_closures: list[int] = []
    for initial_state in range(state_count):
        closure = 1 << initial_state
        frontier = closure
        while frontier:
            state_bit = frontier & -frontier
            frontier ^= state_bit
            state = state_bit.bit_length() - 1
            discovered = epsilon_targets[state] & ~closure
            closure |= discovered
            frontier |= discovered
        epsilon_closures.append(closure)

    def close_subset(subset: int) -> int:
        closure = 0
        remaining = subset
        while remaining:
            state_bit = remaining & -remaining
            remaining ^= state_bit
            closure |= epsilon_closures[state_bit.bit_length() - 1]
        return closure

    start_subset = epsilon_closures[start_state]
    subsets = [start_subset]
    subset_ids = {start_subset: 0}
    dense_transitions: list[tuple[int, ...]] = []
    state_index = 0
    while state_index < len(subsets):
        subset = subsets[state_index]
        row: list[int] = []
        for symbol_index in range(len(alphabet)):
            moved = 0
            remaining = subset
            while remaining:
                state_bit = remaining & -remaining
                remaining ^= state_bit
                moved |= symbol_targets[state_bit.bit_length() - 1][symbol_index]
            target_subset = close_subset(moved)
            target_id = subset_ids.get(target_subset)
            if target_id is None:
                if len(subsets) >= _MAX_REACHABLE_DFA_STATES:
                    raise DfaStateLimitError(_MAX_REACHABLE_DFA_STATES)
                target_id = len(subsets)
                subset_ids[target_subset] = target_id
                subsets.append(target_subset)
            row.append(target_id)
        dense_transitions.append(tuple(row))
        state_index += 1

    state_subsets = tuple(
        tuple(state for state in range(state_count) if subset >> state & 1)
        for subset in subsets
    )
    dfa_accepting = tuple(
        dfa_state
        for dfa_state, subset in enumerate(subsets)
        if subset & accepting_mask
    )
    return CanonicalDfa(
        alphabet,
        state_subsets,
        tuple(dense_transitions),
        dfa_accepting,
    )
```

## Example

```python
def reference_determinize(
    state_count: int,
    alphabet: tuple[str, ...],
    start_state: int,
    accepting_states: tuple[int, ...],
    transitions: tuple[NfaTransition, ...],
) -> CanonicalDfa:
    epsilon_edges = {state: set() for state in range(state_count)}
    symbol_edges = {
        (state, symbol): set() for state in range(state_count) for symbol in alphabet
    }
    for source, symbol, target in transitions:
        if symbol is None:
            epsilon_edges[source].add(target)
        else:
            symbol_edges[source, symbol].add(target)

    def closure(states: frozenset[int]) -> frozenset[int]:
        result = set(states)
        stack = list(states)
        while stack:
            for target in epsilon_edges[stack.pop()]:
                if target not in result:
                    result.add(target)
                    stack.append(target)
        return frozenset(result)

    subsets = [closure(frozenset((start_state,)))]
    subset_ids = {subsets[0]: 0}
    rows: list[tuple[int, ...]] = []
    for subset in subsets:
        row: list[int] = []
        for symbol in alphabet:
            moved = frozenset(
                target
                for state in subset
                for target in symbol_edges[state, symbol]
            )
            target_subset = closure(moved)
            if target_subset not in subset_ids:
                subset_ids[target_subset] = len(subsets)
                subsets.append(target_subset)
            row.append(subset_ids[target_subset])
        rows.append(tuple(row))
    accepting = set(accepting_states)
    return CanonicalDfa(
        alphabet,
        tuple(tuple(sorted(subset)) for subset in subsets),
        tuple(rows),
        tuple(index for index, subset in enumerate(subsets) if subset & accepting),
    )


def nfa_accepts(
    start_state: int,
    accepting_states: tuple[int, ...],
    transitions: tuple[NfaTransition, ...],
    word: tuple[str, ...],
) -> bool:
    def closure(states: set[int]) -> set[int]:
        result = set(states)
        stack = list(states)
        while stack:
            source = stack.pop()
            for edge_source, symbol, target in transitions:
                if edge_source == source and symbol is None and target not in result:
                    result.add(target)
                    stack.append(target)
        return result

    current = closure({start_state})
    for symbol in word:
        current = closure(
            {
                target
                for source, edge_symbol, target in transitions
                if source in current and edge_symbol == symbol
            }
        )
    return bool(current & set(accepting_states))


def accepts(dfa: CanonicalDfa, word: tuple[str, ...]) -> bool:
    symbol_indexes = {symbol: index for index, symbol in enumerate(dfa.alphabet)}
    state = 0
    for symbol in word:
        state = dfa.transitions[state][symbol_indexes[symbol]]
    return state in dfa.accepting_states


alphabet = ("a", "b")
transitions = (
    (0, None, 1),
    (0, None, 2),
    (1, "a", 1),
    (1, "b", 3),
    (2, "b", 2),
    (2, "a", 3),
    (3, None, 1),
)
dfa = determinize_epsilon_nfa(4, alphabet, 0, (3,), transitions)
assert dfa == reference_determinize(4, alphabet, 0, (3,), transitions)
assert dfa == determinize_epsilon_nfa(
    4,
    alphabet,
    0,
    (3,),
    tuple(reversed(transitions)),
)

for length in range(7):
    for word in product(alphabet, repeat=length):
        assert accepts(dfa, word) == nfa_accepts(0, (3,), transitions, word)


def nth_symbol_from_end_nfa(distance: int) -> tuple[int, tuple[NfaTransition, ...]]:
    edges: list[NfaTransition] = [(0, "0", 0), (0, "1", 0), (0, "1", 1)]
    for state in range(1, distance):
        edges.extend(((state, "0", state + 1), (state, "1", state + 1)))
    return distance + 1, tuple(edges)


state_count, cap_edges = nth_symbol_from_end_nfa(10)
at_cap = determinize_epsilon_nfa(state_count, ("0", "1"), 0, (10,), cap_edges)
state_count, over_cap_edges = nth_symbol_from_end_nfa(11)
try:
    determinize_epsilon_nfa(state_count, ("0", "1"), 0, (11,), over_cap_edges)
except DfaStateLimitError as error:
    over_cap_rejected = error.limit == 1_024
else:
    over_cap_rejected = False

assert len(at_cap.state_subsets) == 1_024 and over_cap_rejected
```

## Trade-offs and Limitations

For `V` NFA states, `E` transitions, `A` symbols, and `D` reachable DFA
subsets, the conservative bound is `O(V * (V + E) + D * A * V)` time and
`O(V + E + D * (V + A))` memory. Integer bitsets keep the bounded production
implementation compact, but do not change the possible exponential growth in
`D`.

The state cap is a fail-closed resource contract, not a language result. The
function creates only reachable complete-DFA states and includes the empty
sink only when a symbol transition reaches it. It does not minimize the DFA,
parse a regular expression, preserve NFA transition identities, produce a
proof of equivalence, handle weighted automata or transducers, or stream an
unbounded state graph.

## Related Snippets

<!-- catalog:related:start -->
- [Minimize a Bounded Complete DFA into a Canonical Reachable Form](minimize-a-bounded-complete-dfa-into-a-canonical-reachable-form.md)
- [Find the Lexicographically First Shortest Word Distinguishing Two States of a Bounded Complete DFA](../testing-tooling/find-the-lexicographically-first-shortest-word-distinguishing-two-states-of-a-bounded-complete-dfa.md)
- [Find a Canonical CYK Parse Witness for Bounded Tokens under a Strict CNF Grammar](find-a-canonical-cyk-parse-witness-for-bounded-tokens-under-a-strict-cnf-grammar.md)
<!-- catalog:related:end -->
