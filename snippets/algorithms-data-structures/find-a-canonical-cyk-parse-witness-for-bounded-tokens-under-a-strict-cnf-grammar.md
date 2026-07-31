---
title: "Find a Canonical CYK Parse Witness for Bounded Tokens under a Strict CNF Grammar"
snippet_type: algorithm
use_cases:
  - parsing
  - data-transformation
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - plan-a-minimum-cost-parenthesization-for-a-bounded-matrix-chain.md
  - ../configuration-serialization/parse-a-bounded-nested-bracket-tree.md
  - ../configuration-serialization/evaluate-a-bounded-boolean-rule-tree-from-closed-json.md
---

# Find a Canonical CYK Parse Witness for Bounded Tokens under a Strict CNF Grammar

## Idea and Problem

Decide whether a bounded token tuple belongs to a strict Chomsky-normal-form grammar and retain one reproducible parse tree as evidence.

CYK fills a chart for every nonempty half-open token span. Terminal productions
seed spans of width one. For wider spans, a binary production succeeds when
its left and right nonterminals already derive the two sides of some split.

An ambiguous grammar can admit several trees. Choosing the smallest split and
then the lexicographically smallest child-nonterminal pair at every node makes
the returned witness independent of production declaration order. Child chart
entries recursively obey the same rule.

## When to Use

Use this reference parser for small token sequences and a grammar that is
already in the exact `A -> token` or `A -> B C` form. Immutable nodes with
explicit spans are useful when acceptance alone is insufficient for tests,
diagnostics, or downstream tree traversal.

Use a parser generator when the source grammar needs precedence, associativity,
semantic actions, recovery, or useful syntax errors. Convert a general context-
free grammar to CNF in a separately tested step; conversion choices can change
tree shape and should not be hidden inside this helper.

## Implementation

```python
from dataclasses import dataclass
from functools import cache
from random import Random

_MAX_CYK_NONTERMINALS = 64
_MAX_CYK_PRODUCTIONS = 256
_MAX_CYK_TOKENS = 64
_MAX_CYK_SYMBOL_BYTES = 64
_MAX_CYK_TOKEN_BYTES = 64
_MAX_CYK_INPUT_BYTES = 4_096


@dataclass(frozen=True, slots=True)
class CykParseNode:
    symbol: str
    start: int
    end: int
    token: str | None
    children: tuple["CykParseNode", ...]


def canonical_cyk_parse(
    nonterminals: tuple[str, ...],
    terminal_productions: tuple[tuple[str, str], ...],
    binary_productions: tuple[tuple[str, str, str], ...],
    start_symbol: str,
    tokens: tuple[str, ...],
) -> CykParseNode | None:
    """Return one canonical strict-CNF parse witness, or None."""
    if type(nonterminals) is not tuple:
        raise TypeError("nonterminals must be an exact tuple")
    if not 1 <= len(nonterminals) <= _MAX_CYK_NONTERMINALS:
        raise ValueError("nonterminal count is outside 1..64")
    for symbol in nonterminals:
        if type(symbol) is not str:
            raise TypeError("nonterminals must be exact strings")
        if not 1 <= len(symbol.encode("utf-8")) <= _MAX_CYK_SYMBOL_BYTES:
            raise ValueError("nonterminal UTF-8 length is outside 1..64 bytes")
    if len(set(nonterminals)) != len(nonterminals):
        raise ValueError("nonterminals must be unique")
    declared = set(nonterminals)
    if type(start_symbol) is not str:
        raise TypeError("start_symbol must be an exact string")
    if start_symbol not in declared:
        raise ValueError("start_symbol must be a declared nonterminal")

    if type(tokens) is not tuple:
        raise TypeError("tokens must be an exact tuple")
    if not 1 <= len(tokens) <= _MAX_CYK_TOKENS:
        raise ValueError("token count is outside 1..64")
    input_bytes = 0
    for token in tokens:
        if type(token) is not str:
            raise TypeError("tokens must be exact strings")
        token_bytes = len(token.encode("utf-8"))
        if not 1 <= token_bytes <= _MAX_CYK_TOKEN_BYTES:
            raise ValueError("token UTF-8 length is outside 1..64 bytes")
        input_bytes += token_bytes
    if input_bytes > _MAX_CYK_INPUT_BYTES:
        raise ValueError("token tuple exceeds 4096 UTF-8 bytes")

    if type(terminal_productions) is not tuple:
        raise TypeError("terminal_productions must be an exact tuple")
    if type(binary_productions) is not tuple:
        raise TypeError("binary_productions must be an exact tuple")
    production_count = len(terminal_productions) + len(binary_productions)
    if not 1 <= production_count <= _MAX_CYK_PRODUCTIONS:
        raise ValueError("production count is outside 1..256")

    checked_terminal: list[tuple[str, str]] = []
    for rule in terminal_productions:
        if type(rule) is not tuple or len(rule) != 2:
            raise TypeError("terminal productions must be exact pairs")
        parent, token = rule
        if parent not in declared:
            raise ValueError("terminal-production parent is undeclared")
        if type(token) is not str:
            raise TypeError("terminal-production tokens must be exact strings")
        if not 1 <= len(token.encode("utf-8")) <= _MAX_CYK_TOKEN_BYTES:
            raise ValueError("terminal-production token is outside 1..64 UTF-8 bytes")
        checked_terminal.append((parent, token))
    if len(set(checked_terminal)) != len(checked_terminal):
        raise ValueError("terminal productions must be duplicate-free")

    checked_binary: list[tuple[str, str, str]] = []
    for rule in binary_productions:
        if type(rule) is not tuple or len(rule) != 3:
            raise TypeError("binary productions must be exact triples")
        parent, left_child, right_child = rule
        if parent not in declared or left_child not in declared or right_child not in declared:
            raise ValueError("binary-production symbols must all be declared")
        checked_binary.append((parent, left_child, right_child))
    if len(set(checked_binary)) != len(checked_binary):
        raise ValueError("binary productions must be duplicate-free")

    terminal_parents: dict[str, list[str]] = {}
    for parent, token in checked_terminal:
        terminal_parents.setdefault(token, []).append(parent)
    for parents in terminal_parents.values():
        parents.sort()
    ordered_binary = sorted(
        checked_binary,
        key=lambda rule: (rule[1], rule[2], rule[0]),
    )

    token_count = len(tokens)
    chart: list[list[dict[str, CykParseNode]]] = [
        [dict() for _ in range(token_count + 1)] for _ in range(token_count)
    ]
    for index, token in enumerate(tokens):
        for parent in terminal_parents.get(token, ()):
            chart[index][index + 1][parent] = CykParseNode(
                symbol=parent,
                start=index,
                end=index + 1,
                token=token,
                children=(),
            )

    for width in range(2, token_count + 1):
        for left in range(token_count - width + 1):
            right = left + width
            cell = chart[left][right]
            for split in range(left + 1, right):
                left_cell = chart[left][split]
                right_cell = chart[split][right]
                for parent, left_symbol, right_symbol in ordered_binary:
                    if parent in cell:
                        continue
                    left_node = left_cell.get(left_symbol)
                    right_node = right_cell.get(right_symbol)
                    if left_node is not None and right_node is not None:
                        cell[parent] = CykParseNode(
                            symbol=parent,
                            start=left,
                            end=right,
                            token=None,
                            children=(left_node, right_node),
                        )
    return chart[0][token_count].get(start_symbol)
```

## Example

```python
def recursive_oracle(
    nonterminals: tuple[str, ...],
    terminal_productions: tuple[tuple[str, str], ...],
    binary_productions: tuple[tuple[str, str, str], ...],
    start_symbol: str,
    tokens: tuple[str, ...],
) -> CykParseNode | None:
    terminal_rules = set(terminal_productions)
    children_by_parent = {
        parent: tuple(
            sorted(
                (left, right)
                for candidate, left, right in binary_productions
                if candidate == parent
            )
        )
        for parent in nonterminals
    }

    @cache
    def derive(symbol: str, left: int, right: int) -> CykParseNode | None:
        if right - left == 1 and (symbol, tokens[left]) in terminal_rules:
            return CykParseNode(symbol, left, right, tokens[left], ())
        for split in range(left + 1, right):
            for left_symbol, right_symbol in children_by_parent[symbol]:
                left_node = derive(left_symbol, left, split)
                if left_node is None:
                    continue
                right_node = derive(right_symbol, split, right)
                if right_node is not None:
                    return CykParseNode(
                        symbol,
                        left,
                        right,
                        None,
                        (left_node, right_node),
                    )
        return None

    return derive(start_symbol, 0, len(tokens))


def yielded_tokens(node: CykParseNode) -> tuple[str, ...]:
    if node.token is not None:
        return (node.token,)
    return tuple(token for child in node.children for token in yielded_tokens(child))


grammar_nonterminals = ("S", "A", "B")
grammar_terminals = (("A", "a"), ("B", "a"))
grammar_binary = (("S", "B", "A"), ("S", "A", "B"))
tree = canonical_cyk_parse(
    grammar_nonterminals,
    grammar_terminals,
    grammar_binary,
    "S",
    ("a", "a"),
)
assert tree is not None
assert tuple(child.symbol for child in tree.children) == ("A", "B")
assert yielded_tokens(tree) == ("a", "a")

rng = Random(0xC7_00)
checked = 0
for _ in range(2_000):
    symbol_count = rng.randrange(1, 6)
    nonterminals = tuple(chr(ord("A") + index) for index in range(symbol_count))
    terminal_productions = tuple(
        sorted(
            {
                (parent, token)
                for parent in nonterminals
                for token in ("a", "b")
                if rng.randrange(3) == 0
            }
        )
    )
    binary_productions = tuple(
        sorted(
            {
                (parent, left, right)
                for parent in nonterminals
                for left in nonterminals
                for right in nonterminals
                if rng.randrange(8) == 0
            }
        )
    )
    if not terminal_productions and not binary_productions:
        terminal_productions = ((nonterminals[0], "a"),)
    tokens = tuple(rng.choice(("a", "b")) for _ in range(rng.randrange(1, 7)))
    start_symbol = rng.choice(nonterminals)
    actual = canonical_cyk_parse(
        nonterminals,
        terminal_productions,
        binary_productions,
        start_symbol,
        tokens,
    )
    expected = recursive_oracle(
        nonterminals,
        terminal_productions,
        binary_productions,
        start_symbol,
        tokens,
    )
    assert actual == expected
    if actual is not None:
        assert yielded_tokens(actual) == tokens
    checked += 1

assert checked == 2_000
```

## Trade-offs and Limitations

With `N` tokens, `P` total productions, and `P₂` binary productions, ordering
rules and scanning the direct chart cost
`O(P log P + N³(P₂ + 1) + NP₁)` time. The added one covers span/split loops
even when `P₂` is zero. The chart stores up to `O(N²V)` nodes for `V`
nonterminals, alongside `O(P)` indexed rule state. This prioritizes a visible
recurrence and deterministic witness over indexed production joins. Each node
shares already-built immutable children, but dense ambiguous grammars can
still approach the chart bound.

The helper accepts only nonempty input and an already strict-CNF grammar. It
does not perform epsilon or unit derivations, grammar conversion, semantic
actions, precedence handling, recovery, probabilistic scoring, or enumeration
of all parses. Lexicographic ties use Python string order over declared
nonterminal names.

## Related Snippets

<!-- catalog:related:start -->
- [Plan a Minimum-Cost Parenthesization for a Bounded Matrix Chain](plan-a-minimum-cost-parenthesization-for-a-bounded-matrix-chain.md)
- [Parse a Bounded Nested Bracket Tree](../configuration-serialization/parse-a-bounded-nested-bracket-tree.md)
- [Evaluate a Bounded Boolean Rule Tree from Closed JSON](../configuration-serialization/evaluate-a-bounded-boolean-rule-tree-from-closed-json.md)
<!-- catalog:related:end -->
