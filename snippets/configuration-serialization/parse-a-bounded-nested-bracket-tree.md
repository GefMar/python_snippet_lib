---
title: "Parse a Bounded Nested Bracket Tree"
snippet_type: algorithm
use_cases:
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - expand-bounded-nested-brace-alternatives.md
  - ../data-processing/split-quoted-and-bracketed-log-fields.md
  - ../python-language/walk-a-tree-recursively-with-yield-from.md
---

# Parse a Bounded Nested Bracket Tree

## Idea and Problem

Parse one small nested bracket document into immutable nodes while preserving the order of text and child-node parts.

The exact grammar is `document := node` and
`node := "[" name ":" (text | node)* "]"`. Names are short ASCII identifiers;
text is printable ASCII without brackets. Full-input consumption and explicit
source, node, and nesting limits turn malformed or oversized input into one
bounded validation failure.

## When to Use

Use this algorithm only when an existing compact format has nested named
sections and text may appear before, between, or after children. Agree on the
grammar and budgets before accepting input. Prefer JSON or another maintained
serialization format when the producer can be changed, especially when text
needs quoting, escaping, Unicode, schema evolution, or streaming.

## Implementation

```python
import re
from dataclasses import dataclass


_NODE_NAME = re.compile(r"[A-Za-z][A-Za-z0-9_-]{0,31}", re.ASCII)


@dataclass(frozen=True, slots=True)
class TextPart:
    value: str


@dataclass(frozen=True, slots=True)
class BracketNode:
    name: str
    parts: tuple["TextPart | BracketNode", ...]


def _checked_limit(value: int, *, name: str, hard_maximum: int) -> int:
    if isinstance(value, bool) or not isinstance(value, int):
        raise TypeError(f"{name} must be an integer")
    if not 1 <= value <= hard_maximum:
        raise ValueError(f"{name} is outside its supported range")
    return value


class _BracketParser:
    __slots__ = ("index", "max_depth", "max_nodes", "nodes", "source")

    def __init__(self, source: str, *, max_nodes: int, max_depth: int) -> None:
        self.source = source
        self.index = 0
        self.nodes = 0
        self.max_nodes = max_nodes
        self.max_depth = max_depth

    def parse_node(self, *, depth: int) -> BracketNode:
        if depth > self.max_depth:
            raise ValueError("document exceeds the nesting limit")
        if self.index >= len(self.source) or self.source[self.index] != "[":
            raise ValueError("expected a node")
        self.index += 1

        name_start = self.index
        while (
            self.index < len(self.source)
            and self.source[self.index] not in ":[]"
        ):
            self.index += 1
        if self.index >= len(self.source) or self.source[self.index] != ":":
            raise ValueError("node header is invalid")
        name = self.source[name_start : self.index]
        if _NODE_NAME.fullmatch(name) is None:
            raise ValueError("node name is invalid")
        self.index += 1

        self.nodes += 1
        if self.nodes > self.max_nodes:
            raise ValueError("document exceeds the node limit")

        parts: list[TextPart | BracketNode] = []
        text_start = self.index
        while self.index < len(self.source):
            character = self.source[self.index]
            if character == "[":
                if text_start < self.index:
                    parts.append(TextPart(self.source[text_start : self.index]))
                parts.append(self.parse_node(depth=depth + 1))
                text_start = self.index
            elif character == "]":
                if text_start < self.index:
                    parts.append(TextPart(self.source[text_start : self.index]))
                self.index += 1
                return BracketNode(name, tuple(parts))
            else:
                self.index += 1
        raise ValueError("node is not closed")


def parse_bracket_tree(
    source: str,
    *,
    max_source_length: int = 4096,
    max_nodes: int = 256,
    max_depth: int = 32,
) -> BracketNode:
    if not isinstance(source, str):
        raise TypeError("source must be text")
    source_limit = _checked_limit(
        max_source_length,
        name="max_source_length",
        hard_maximum=100_000,
    )
    node_limit = _checked_limit(
        max_nodes,
        name="max_nodes",
        hard_maximum=10_000,
    )
    depth_limit = _checked_limit(
        max_depth,
        name="max_depth",
        hard_maximum=64,
    )
    if not source or len(source) > source_limit:
        raise ValueError("source length is outside the accepted range")
    if any(not 32 <= ord(character) <= 126 for character in source):
        raise ValueError("source must contain printable ASCII only")

    parser = _BracketParser(
        source,
        max_nodes=node_limit,
        max_depth=depth_limit,
    )
    root = parser.parse_node(depth=1)
    if parser.index != len(source):
        raise ValueError("document contains trailing content")
    return root
```

## Example

```python
tree = parse_bracket_tree(
    "[panel:before[item:first][item:second]after]",
)
expected = BracketNode(
    "panel",
    (
        TextPart("before"),
        BracketNode("item", (TextPart("first"),)),
        BracketNode("item", (TextPart("second"),)),
        TextPart("after"),
    ),
)


def is_rejected(source: str, *, max_nodes: int = 8) -> bool:
    try:
        parse_bracket_tree(source, max_nodes=max_nodes)
    except ValueError:
        return True
    return False


assert (
    tree,
    parse_bracket_tree("[empty:]"),
    is_rejected("[root:missing"),
    is_rejected("[root:]trailing"),
    is_rejected("[:unnamed]"),
    is_rejected("[root:[a:][b:]]", max_nodes=2),
) == (
    expected,
    BracketNode("empty", ()),
    True,
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

Brackets are always structural and cannot appear in text because the grammar
has no escaping or quoting. Node names and text are ASCII-only, whitespace in
names is invalid, and an unmatched `]` closes the current node rather than
becoming data. Recursive parsing retains a hard depth ceiling of 64; accepted
memory still grows with source length and node count. The result preserves
part order but performs no schema validation on names or content. This is not
a general log parser, streaming parser, or replacement for a standard format.

## Related Snippets

<!-- catalog:related:start -->
- [Expand Bounded Nested Brace Alternatives](expand-bounded-nested-brace-alternatives.md)
- [Split Quoted and Bracketed Log Fields](../data-processing/split-quoted-and-bracketed-log-fields.md)
- [Walk a Tree Recursively with yield from](../python-language/walk-a-tree-recursively-with-yield-from.md)
<!-- catalog:related:end -->
