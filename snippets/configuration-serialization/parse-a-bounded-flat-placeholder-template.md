---
title: "Parse a Bounded Flat Placeholder Template"
snippet_type: algorithm
use_cases:
  - configuration
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - render-fixed-date-placeholders-from-an-explicit-anchor.md
  - substitute-typed-values-into-a-json-like-template.md
  - reject-unknown-options-with-conservative-typo-suggestions.md
---

# Parse a Bounded Flat Placeholder Template

## Idea and Problem

Parse a small text template into immutable literal and placeholder tokens while accepting only declared flat ASCII field names.

The grammar is `template := (literal | "{{" | "}}" | placeholder)*`,
`placeholder := "{" name "}"`, where `name` is an ASCII letter or underscore
followed by at most 63 ASCII letters, digits, or underscores. Doubled braces
become literal brace characters. A single brace starts or ends structure, so
attributes, indexes, conversions, format specifications, and malformed fields
cannot acquire special behavior.

## When to Use

Use this parser when a controlled configuration format needs flat named slots
but rendering or code generation happens in a separate stage. Supply the exact
allowed-name tuple from a reviewed schema, retain the returned token order, and
decide separately whether repeated placeholders or unused declarations are
appropriate for the caller.

Choose a maintained template language when producers need conditionals, loops,
pluralization, locale rules, nested lookups, expressions, or source locations.
This deliberately narrow grammar is suitable only for small bounded templates.

## Implementation

```python
import re
from dataclasses import dataclass
from typing import TypeAlias


_MAX_TEMPLATE_BYTES = 16_384
_MAX_DECLARED_NAMES = 128
_MAX_TOKENS = 1_024
_FIELD_NAME = re.compile(r"[A-Za-z_][A-Za-z0-9_]{0,63}", re.ASCII)


class FlatTemplateError(ValueError):
    def __init__(self, position: int) -> None:
        self.position = position
        super().__init__(f"invalid flat template at position {position}")


@dataclass(frozen=True, slots=True)
class LiteralToken:
    value: str


@dataclass(frozen=True, slots=True)
class PlaceholderToken:
    name: str


TemplateToken: TypeAlias = LiteralToken | PlaceholderToken


def _fail(position: int) -> None:
    raise FlatTemplateError(position)


def _append_token(
    tokens: list[TemplateToken],
    token: TemplateToken,
    *,
    position: int,
) -> None:
    if len(tokens) >= _MAX_TOKENS:
        _fail(position)
    tokens.append(token)


def _declared_name_set(declared_names: tuple[str, ...]) -> frozenset[str]:
    if type(declared_names) is not tuple:
        raise TypeError("declared_names must be a tuple")
    if len(declared_names) > _MAX_DECLARED_NAMES:
        raise ValueError("declared_names exceed the supported count")

    names: set[str] = set()
    for name in declared_names:
        if type(name) is not str:
            raise TypeError("declared names must be text")
        if _FIELD_NAME.fullmatch(name) is None:
            raise ValueError("declared_names contains an invalid field name")
        if name in names:
            raise ValueError("declared_names must be unique")
        names.add(name)
    return frozenset(names)


def parse_flat_placeholder_template(
    template: str,
    declared_names: tuple[str, ...],
) -> tuple[TemplateToken, ...]:
    if type(template) is not str:
        raise TypeError("template must be text")
    if len(template) > _MAX_TEMPLATE_BYTES:
        _fail(_MAX_TEMPLATE_BYTES)
    try:
        template_size = len(template.encode("utf-8"))
    except UnicodeEncodeError as error:
        _fail(error.start)
    if template_size > _MAX_TEMPLATE_BYTES:
        _fail(len(template))
    allowed = _declared_name_set(declared_names)

    tokens: list[TemplateToken] = []
    literal: list[str] = []

    def flush_literal(position: int) -> None:
        if literal:
            _append_token(
                tokens,
                LiteralToken("".join(literal)),
                position=position,
            )
            literal.clear()

    index = 0
    while index < len(template):
        character = template[index]
        if character == "{":
            if index + 1 < len(template) and template[index + 1] == "{":
                literal.append("{")
                index += 2
                continue

            flush_literal(index)
            close = template.find("}", index + 1)
            if close == -1:
                _fail(index)
            name = template[index + 1 : close]
            if _FIELD_NAME.fullmatch(name) is None or name not in allowed:
                _fail(index)
            _append_token(
                tokens,
                PlaceholderToken(name),
                position=index,
            )
            index = close + 1
            continue

        if character == "}":
            if index + 1 < len(template) and template[index + 1] == "}":
                literal.append("}")
                index += 2
                continue
            _fail(index)

        literal.append(character)
        index += 1

    flush_literal(len(template))
    return tuple(tokens)
```

## Example

```python
tokens = parse_flat_placeholder_template(
    "Hello, café {{user}}: {customer} / {customer}; count={count}.",
    ("customer", "count"),
)

invalid_templates = (
    "{customer.name}",
    "{items[0]}",
    "{customer!r}",
    "{count:03d}",
    "{count:}",
    "{unknown}",
    "unclosed {customer",
    "single } brace",
)
rejected_positions = []
for invalid in invalid_templates:
    try:
        parse_flat_placeholder_template(
            invalid,
            ("customer", "count", "items"),
        )
    except FlatTemplateError as error:
        rejected_positions.append(error.position)

try:
    tokens[1].name = "changed"
except (AttributeError, TypeError):
    immutable = True
else:
    immutable = False

assert (tokens, rejected_positions, immutable) == (
    (
        LiteralToken("Hello, café {user}: "),
        PlaceholderToken("customer"),
        LiteralToken(" / "),
        PlaceholderToken("customer"),
        LiteralToken("; count="),
        PlaceholderToken("count"),
        LiteralToken("."),
    ),
    [0, 0, 0, 0, 0, 0, 9, 7],
    True,
)
```

## Trade-offs and Limitations

Parsing is linear in the bounded template length and allocates the decoded
literal text plus at most 1,024 frozen tokens. Doubled braces are represented
by their single literal character, so the token stream preserves their meaning
but cannot reconstruct whether an ordinary literal segment used brace escapes.
Repeated fields remain repeated tokens, and unused declared names are allowed.

Field matching is exact and case-sensitive. The parser provides no values,
rendering, defaults, type annotations, format operations, Unicode field names,
attribute or index access, comments, code generation, or pluralization. A
declaration tuple is caller-owned schema input: invalid or duplicate declarations
raise ordinary type or value errors, while every template-content failure uses
`FlatTemplateError` without echoing the supplied text.

## Related Snippets

<!-- catalog:related:start -->
- [Render Fixed Date Placeholders from an Explicit Anchor](render-fixed-date-placeholders-from-an-explicit-anchor.md)
- [Substitute Typed Values into a JSON-Like Template](substitute-typed-values-into-a-json-like-template.md)
- [Reject Unknown Options with Conservative Typo Suggestions](reject-unknown-options-with-conservative-typo-suggestions.md)
<!-- catalog:related:end -->
