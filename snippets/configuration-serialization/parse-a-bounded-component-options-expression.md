---
title: "Parse a Bounded Component Options Expression"
snippet_type: algorithm
use_cases:
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - parse-a-bounded-nested-bracket-tree.md
  - parse-explicit-decimal-and-binary-byte-sizes.md
  - reject-unknown-options-with-conservative-typo-suggestions.md
---

# Parse a Bounded Component Options Expression

## Idea and Problem

Parse a small ordered component list with optional integer options under one explicit bounded ASCII grammar.

The grammar is `component ("," component)*`. A component is a canonical
lowercase identifier followed by either nothing or one non-empty parenthesized
list of `key=unsigned_decimal` options. Spaces and tabs are allowed only around
tokens. The result retains component order, option order, identifier spelling,
and decimal spelling in frozen tuples.

## When to Use

Use this parser when an existing configuration boundary needs a deliberately
narrow human-editable expression and every accepted producer can follow the
same grammar. It is appropriate only for bounded option lists whose values are
non-negative integers.

Prefer JSON, TOML, or another maintained format when values need strings,
nesting, quoting, Unicode, comments, schema evolution, or streaming. Keep
semantic validation of recognized component and option names separate from
this syntax parser.

## Implementation

```python
import re
from dataclasses import dataclass


_MAX_SOURCE_BYTES = 4_096
_MAX_COMPONENTS = 64
_MAX_OPTIONS_PER_COMPONENT = 32
_MAX_TOTAL_OPTIONS = 256
_MAX_IDENTIFIER_LENGTH = 64
_MAX_DECIMAL_DIGITS = 12
_MAX_OPTION_VALUE = 1_000_000_000
_IDENTIFIER = re.compile(r"[a-z][a-z0-9]*(?:-[a-z0-9]+)*", re.ASCII)


class ComponentExpressionError(ValueError):
    def __init__(self, position: int) -> None:
        self.position = position
        super().__init__(f"invalid component expression at position {position}")


def _fail(position: int) -> None:
    raise ComponentExpressionError(position)


@dataclass(frozen=True, slots=True)
class ComponentOption:
    key: str
    value: int
    decimal: str


@dataclass(frozen=True, slots=True)
class ComponentSpec:
    name: str
    options: tuple[ComponentOption, ...]


class _ComponentParser:
    __slots__ = ("index", "source", "total_options")

    def __init__(self, source: str) -> None:
        self.source = source
        self.index = 0
        self.total_options = 0

    def _skip_space(self) -> None:
        while self.index < len(self.source) and self.source[self.index] in " \t":
            self.index += 1

    def _identifier(self) -> tuple[str, int]:
        start = self.index
        match = _IDENTIFIER.match(self.source, start)
        if match is None:
            _fail(start)
        value = match.group()
        if len(value) > _MAX_IDENTIFIER_LENGTH:
            _fail(start + _MAX_IDENTIFIER_LENGTH)
        self.index = match.end()
        return value, start

    def _unsigned_decimal(self) -> tuple[int, str]:
        start = self.index
        while (
            self.index < len(self.source)
            and self.source[self.index] in "0123456789"
        ):
            self.index += 1
        if self.index == start:
            _fail(start)
        decimal = self.source[start : self.index]
        if len(decimal) > _MAX_DECIMAL_DIGITS:
            _fail(start + _MAX_DECIMAL_DIGITS)
        value = int(decimal)
        if value > _MAX_OPTION_VALUE:
            _fail(start)
        return value, decimal

    def _component(self) -> tuple[ComponentSpec, int]:
        name, name_position = self._identifier()
        self._skip_space()
        if self.index >= len(self.source) or self.source[self.index] != "(":
            return ComponentSpec(name, ()), name_position

        self.index += 1
        self._skip_space()
        if self.index >= len(self.source) or self.source[self.index] == ")":
            _fail(self.index)

        options: list[ComponentOption] = []
        option_keys: set[str] = set()
        while True:
            key, key_position = self._identifier()
            if key in option_keys:
                _fail(key_position)
            option_keys.add(key)

            self._skip_space()
            if self.index >= len(self.source) or self.source[self.index] != "=":
                _fail(self.index)
            self.index += 1
            self._skip_space()
            value, decimal = self._unsigned_decimal()
            options.append(ComponentOption(key, value, decimal))
            self.total_options += 1
            if len(options) > _MAX_OPTIONS_PER_COMPONENT:
                _fail(key_position)
            if self.total_options > _MAX_TOTAL_OPTIONS:
                _fail(key_position)

            self._skip_space()
            if self.index >= len(self.source):
                _fail(self.index)
            delimiter = self.source[self.index]
            self.index += 1
            if delimiter == ")":
                return ComponentSpec(name, tuple(options)), name_position
            if delimiter != ",":
                _fail(self.index - 1)
            self._skip_space()

    def parse(self) -> tuple[ComponentSpec, ...]:
        self._skip_space()
        if self.index == len(self.source):
            _fail(self.index)

        components: list[ComponentSpec] = []
        names: set[str] = set()
        while True:
            component, name_position = self._component()
            if component.name in names:
                _fail(name_position)
            names.add(component.name)
            components.append(component)
            if len(components) > _MAX_COMPONENTS:
                _fail(name_position)

            self._skip_space()
            if self.index == len(self.source):
                return tuple(components)
            if self.source[self.index] != ",":
                _fail(self.index)
            self.index += 1
            self._skip_space()
            if self.index == len(self.source):
                _fail(self.index)


def parse_component_options_expression(source: str) -> tuple[ComponentSpec, ...]:
    if type(source) is not str:
        raise TypeError("source must be text")
    if len(source) > _MAX_SOURCE_BYTES:
        _fail(_MAX_SOURCE_BYTES)
    try:
        source.encode("ascii")
    except UnicodeEncodeError as error:
        _fail(error.start)
    return _ComponentParser(source).parse()
```

## Example

```python
parsed = parse_component_options_expression(
    " cache ( ttl=00030, retries = 2 ) , indexer\t(mode=1) , plain "
)

invalid_sources = (
    "cache()",
    "cache(ttl=-1)",
    "cache(ttl=1,ttl=2)",
    "cache,cache",
    "cache(ttl=1),",
    "cache(ttl=(1))",
    "cache(name=\"one\")",
    "caché",
    "cache\n",
)
errors = []
for invalid in invalid_sources:
    try:
        parse_component_options_expression(invalid)
    except ComponentExpressionError as error:
        errors.append((error.position, str(error)))

assert (
    parsed,
    len(errors),
    all("cache" not in message for _, message in errors),
) == (
    (
        ComponentSpec(
            "cache",
            (
                ComponentOption("ttl", 30, "00030"),
                ComponentOption("retries", 2, "2"),
            ),
        ),
        ComponentSpec("indexer", (ComponentOption("mode", 1, "1"),)),
        ComponentSpec("plain", ()),
    ),
    9,
    True,
)
```

## Trade-offs and Limitations

Parsing is linear in the bounded source length and allocates one immutable
result object per accepted component and option. The parser preserves leading
zeros in `ComponentOption.decimal` while also exposing the checked integer in
`value`. Duplicate names are compared exactly because every identifier is
already in one canonical lowercase ASCII form.

The grammar intentionally has no signs, escapes, quoting, nesting, comments,
empty lists, or Unicode. Errors expose only a zero-based position and never
echo the supplied text or claim which semantic option was intended. The
parser does not decide whether a recognized component supports a particular
key; apply that policy after syntax succeeds.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Bounded Nested Bracket Tree](parse-a-bounded-nested-bracket-tree.md)
- [Parse Explicit Decimal and Binary Byte Sizes](parse-explicit-decimal-and-binary-byte-sizes.md)
- [Reject Unknown Options with Conservative Typo Suggestions](reject-unknown-options-with-conservative-typo-suggestions.md)
<!-- catalog:related:end -->
