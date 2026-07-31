---
title: "Compile and Match a Bounded Postfix Regular Expression with a Thompson Epsilon-NFA"
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
  - determinize-a-bounded-epsilon-nfa-into-a-canonical-complete-dfa-under-a-state-cap.md
  - minimize-a-bounded-complete-dfa-into-a-canonical-reachable-form.md
  - find-all-overlapping-exact-text-matches-with-knuth-morris-pratt.md
---

# Compile and Match a Bounded Postfix Regular Expression with a Thompson Epsilon-NFA

## Idea and Problem

Compile a small typed postfix regular expression into an explicit epsilon-NFA and use it for exact full-string matching.

Each literal creates a two-state fragment. Concatenation joins two fragments,
alternation adds a shared entry and exit, and Kleene star adds an entry/exit
pair with the required skip and repeat epsilon transitions. A postfix stream
removes precedence and escaping ambiguity: every operator consumes its operands
from one fragment stack.

State IDs follow construction order, and the returned transition tuples are
sorted, so the same typed token stream always produces the same frozen
automaton. Matching repeatedly expands epsilon closure before consuming the
next Unicode scalar.

## When to Use

Use this construction when a bounded tool or teaching example already has a
typed postfix expression and needs the automaton to remain visible, reusable,
and independently inspectable. It supports literal Unicode scalars,
concatenation, alternation, and zero-or-more repetition with full-string
semantics.

Use a maintained regular-expression engine for textual patterns, rich syntax,
large inputs, searching, or production denial-of-service defenses. Use the
catalog's determinization algorithm when a complete DFA is required after this
construction.

## Implementation

```python
from dataclasses import dataclass

_MAX_POSTFIX_TOKENS = 256
_MAX_NFA_STATES = 513
_MAX_NFA_TRANSITIONS = 1_024
_MAX_TEXT_CODE_POINTS = 4_096
_MAX_TEXT_UTF8_BYTES = 16_384

NfaTransition = tuple[int, str | None, int]


@dataclass(frozen=True, slots=True)
class Literal:
    value: str


@dataclass(frozen=True, slots=True)
class Concat:
    pass


@dataclass(frozen=True, slots=True)
class Alternate:
    pass


@dataclass(frozen=True, slots=True)
class Star:
    pass


PostfixToken = Literal | Concat | Alternate | Star


@dataclass(frozen=True, slots=True)
class ThompsonNfa:
    state_count: int
    start_state: int
    accept_state: int
    transitions: tuple[NfaTransition, ...]


@dataclass(frozen=True, slots=True)
class _Fragment:
    start: int
    accept: int


def _validate_scalar(value: object, *, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if len(value) != 1 or 0xD800 <= ord(value) <= 0xDFFF:
        raise ValueError(f"{field} must contain exactly one Unicode scalar")
    return value


def _transition_key(transition: NfaTransition) -> tuple[int, bool, str, int]:
    source, symbol, target = transition
    return source, symbol is not None, symbol or "", target


def compile_postfix_regex(tokens: tuple[PostfixToken, ...]) -> ThompsonNfa:
    """Compile the closed postfix grammar into a deterministic frozen NFA."""
    if type(tokens) is not tuple:
        raise TypeError("tokens must be an exact tuple")
    if not 1 <= len(tokens) <= _MAX_POSTFIX_TOKENS:
        raise ValueError("token count is outside 1..256")

    state_count = 0
    transitions: list[NfaTransition] = []
    fragments: list[_Fragment] = []

    def new_state() -> int:
        nonlocal state_count
        if state_count >= _MAX_NFA_STATES:
            raise ValueError("compiled NFA exceeds 513 states")
        state = state_count
        state_count += 1
        return state

    def add_transition(source: int, symbol: str | None, target: int) -> None:
        if len(transitions) >= _MAX_NFA_TRANSITIONS:
            raise ValueError("compiled NFA exceeds 1,024 transitions")
        transitions.append((source, symbol, target))

    for index, token in enumerate(tokens):
        if type(token) is Literal:
            symbol = _validate_scalar(token.value, field=f"tokens[{index}].value")
            start = new_state()
            accept = new_state()
            add_transition(start, symbol, accept)
            fragments.append(_Fragment(start, accept))
        elif type(token) is Concat:
            if len(fragments) < 2:
                raise ValueError(f"tokens[{index}] has too few operands")
            right = fragments.pop()
            left = fragments.pop()
            add_transition(left.accept, None, right.start)
            fragments.append(_Fragment(left.start, right.accept))
        elif type(token) is Alternate:
            if len(fragments) < 2:
                raise ValueError(f"tokens[{index}] has too few operands")
            right = fragments.pop()
            left = fragments.pop()
            start = new_state()
            accept = new_state()
            add_transition(start, None, left.start)
            add_transition(start, None, right.start)
            add_transition(left.accept, None, accept)
            add_transition(right.accept, None, accept)
            fragments.append(_Fragment(start, accept))
        elif type(token) is Star:
            if not fragments:
                raise ValueError(f"tokens[{index}] has too few operands")
            repeated = fragments.pop()
            start = new_state()
            accept = new_state()
            add_transition(start, None, repeated.start)
            add_transition(start, None, accept)
            add_transition(repeated.accept, None, repeated.start)
            add_transition(repeated.accept, None, accept)
            fragments.append(_Fragment(start, accept))
        else:
            raise TypeError(f"tokens[{index}] has an unknown token type")

    if len(fragments) != 1:
        raise ValueError("postfix expression must leave exactly one fragment")
    fragment = fragments[0]
    return ThompsonNfa(
        state_count=state_count,
        start_state=fragment.start,
        accept_state=fragment.accept,
        transitions=tuple(sorted(transitions, key=_transition_key)),
    )


def fullmatch_thompson(nfa: ThompsonNfa, text: str) -> bool:
    """Return whether text is a full match for one bounded Thompson NFA."""
    if type(nfa) is not ThompsonNfa:
        raise TypeError("nfa must be an exact ThompsonNfa")
    if type(text) is not str:
        raise TypeError("text must be an exact string")
    if len(text) > _MAX_TEXT_CODE_POINTS:
        raise ValueError("text exceeds 4,096 code points")
    try:
        encoded_text = text.encode("utf-8")
    except UnicodeEncodeError:
        raise ValueError("text must contain only Unicode scalars") from None
    if len(encoded_text) > _MAX_TEXT_UTF8_BYTES:
        raise ValueError("text exceeds 16,384 UTF-8 bytes")

    if type(nfa.state_count) is not int or not 1 <= nfa.state_count <= _MAX_NFA_STATES:
        raise ValueError("nfa.state_count is outside 1..513")
    if type(nfa.start_state) is not int or type(nfa.accept_state) is not int:
        raise TypeError("NFA state IDs must be exact integers")
    if not 0 <= nfa.start_state < nfa.state_count:
        raise ValueError("nfa.start_state is outside the NFA")
    if not 0 <= nfa.accept_state < nfa.state_count:
        raise ValueError("nfa.accept_state is outside the NFA")
    if type(nfa.transitions) is not tuple:
        raise TypeError("nfa.transitions must be an exact tuple")
    if len(nfa.transitions) > _MAX_NFA_TRANSITIONS:
        raise ValueError("NFA exceeds 1,024 transitions")

    epsilon_targets: list[list[int]] = [[] for _ in range(nfa.state_count)]
    symbol_targets: list[dict[str, list[int]]] = [{} for _ in range(nfa.state_count)]
    seen: set[NfaTransition] = set()
    previous_key: tuple[int, bool, str, int] | None = None
    for index, transition in enumerate(nfa.transitions):
        if type(transition) is not tuple or len(transition) != 3:
            raise TypeError(f"nfa.transitions[{index}] must be an exact triple")
        source, symbol, target = transition
        if type(source) is not int or type(target) is not int:
            raise TypeError(f"nfa.transitions[{index}] endpoints must be exact integers")
        if not 0 <= source < nfa.state_count or not 0 <= target < nfa.state_count:
            raise ValueError(f"nfa.transitions[{index}] endpoint is outside the NFA")
        if symbol is not None:
            symbol = _validate_scalar(symbol, field=f"nfa.transitions[{index}] symbol")
        normalized = (source, symbol, target)
        key = _transition_key(normalized)
        if previous_key is not None and key <= previous_key:
            raise ValueError("NFA transitions must be unique and strictly sorted")
        previous_key = key
        if normalized in seen:
            raise ValueError(f"nfa.transitions[{index}] is duplicated")
        seen.add(normalized)
        if symbol is None:
            epsilon_targets[source].append(target)
        else:
            symbol_targets[source].setdefault(symbol, []).append(target)

    def epsilon_closure(initial: set[int]) -> set[int]:
        closure = set(initial)
        pending = list(initial)
        while pending:
            source = pending.pop()
            for target in epsilon_targets[source]:
                if target not in closure:
                    closure.add(target)
                    pending.append(target)
        return closure

    active = epsilon_closure({nfa.start_state})
    for symbol in text:
        moved: set[int] = set()
        for source in active:
            moved.update(symbol_targets[source].get(symbol, ()))
        active = epsilon_closure(moved)
        if not active:
            return False
    return nfa.accept_state in active
```

## Example

```python
import re
from itertools import product

Ast = tuple[object, ...]


def to_postfix(node: Ast) -> tuple[PostfixToken, ...]:
    kind = node[0]
    if kind == "literal":
        return (Literal(node[1]),)
    if kind == "concat":
        return (*to_postfix(node[1]), *to_postfix(node[2]), Concat())
    if kind == "alternate":
        return (*to_postfix(node[1]), *to_postfix(node[2]), Alternate())
    if kind == "star":
        return (*to_postfix(node[1]), Star())
    raise AssertionError("unknown test AST")


def to_python_pattern(node: Ast) -> str:
    kind = node[0]
    if kind == "literal":
        return re.escape(node[1])
    if kind == "concat":
        return f"(?:{to_python_pattern(node[1])})(?:{to_python_pattern(node[2])})"
    if kind == "alternate":
        return f"(?:{to_python_pattern(node[1])}|{to_python_pattern(node[2])})"
    if kind == "star":
        return f"(?:{to_python_pattern(node[1])})*"
    raise AssertionError("unknown test AST")


literals: tuple[Ast, ...] = (
    ("literal", "a"),
    ("literal", "b"),
    ("literal", "é"),
)
asts = [*literals]
for left, right in product(literals, repeat=2):
    asts.append(("concat", left, right))
    asts.append(("alternate", left, right))
asts.extend(("star", node) for node in tuple(asts))

texts = [
    "".join(symbols) for length in range(6) for symbols in product(("a", "b", "é"), repeat=length)
]
checked_matches = 0
for ast in asts:
    nfa = compile_postfix_regex(to_postfix(ast))
    python_pattern = to_python_pattern(ast)
    for candidate_text in texts:
        assert fullmatch_thompson(nfa, candidate_text) == (
            re.fullmatch(python_pattern, candidate_text) is not None
        )
        checked_matches += 1

concatenated = compile_postfix_regex((Literal("a"), Literal("b"), Concat()))
largest_tokens: list[PostfixToken] = [Literal("a")]
for index in range(127):
    largest_tokens.extend((Literal("b" if index % 2 else "a"), Alternate()))
largest_tokens.append(Star())
largest_nfa = compile_postfix_regex(tuple(largest_tokens))

malformed_rejections = 0
for malformed in (
    (Concat(),),
    (Literal("a"), Literal("b")),
    (Literal(""),),
    (Literal("ab"),),
):
    try:
        compile_postfix_regex(malformed)
    except (TypeError, ValueError):
        malformed_rejections += 1

assert (
    concatenated.state_count,
    concatenated.transitions,
    fullmatch_thompson(concatenated, "ab"),
    fullmatch_thompson(concatenated, "a"),
    len(largest_tokens),
    largest_nfa.state_count,
    malformed_rejections,
    checked_matches,
) == (
    4,
    ((0, "a", 1), (1, None, 2), (2, "b", 3)),
    True,
    False,
    256,
    512,
    4,
    15_288,
)
```

## Trade-offs and Limitations

Compilation takes `O(P log P)` time because the `O(P)` produced transitions
are sorted, and it retains `O(P)` states and transitions. Including initial
validation and closure, full matching takes `O((T + 1) * (S + E))` time in the
conservative worst case. It uses `O(S + E + T)` temporary memory because it
indexes the automaton, maintains closure state, and materializes bounded UTF-8
text during validation. The 256-token grammar cannot quite reach the separate
513-state defensive cap; its largest valid construction has 512 states.

Construction-order IDs and sorted transitions are reproducible for the exact
typed postfix stream, not a canonical representation of its language.
Equivalent expressions can produce different NFAs, and Thompson construction
does not minimize or determinize them. Repeated matching recompiles nothing if
the caller retains the returned frozen NFA.

The grammar has no textual lexer or infix syntax, epsilon token, wildcard,
character class, bounded repetition, capture, anchor, backreference,
lookaround, search, or substitution behavior. It performs exact full-string
matching only and is not an unbounded or hardened regular-expression service.

## Related Snippets

<!-- catalog:related:start -->
- [Determinize a Bounded Epsilon-NFA into a Canonical Complete DFA Under a State Cap](determinize-a-bounded-epsilon-nfa-into-a-canonical-complete-dfa-under-a-state-cap.md)
- [Minimize a Bounded Complete DFA into a Canonical Reachable Form](minimize-a-bounded-complete-dfa-into-a-canonical-reachable-form.md)
- [Find All Overlapping Exact Text Matches with Knuth-Morris-Pratt](find-all-overlapping-exact-text-matches-with-knuth-morris-pratt.md)
<!-- catalog:related:end -->
