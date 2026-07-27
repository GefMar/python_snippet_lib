---
title: "Parse a Bounded ASCII Media Type Value"
snippet_type: recipe
use_cases:
  - interoperability
  - parsing
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - parse-a-bounded-host-and-port-with-bracketed-ipv6.md
  - ../data-processing/collect-expected-parse-failures-without-stopping-a-batch.md
  - ../data-processing/split-quoted-and-bracketed-log-fields.md
---

# Parse a Bounded ASCII Media Type Value

## Idea and Problem

Parse one strict media type and its parameters into an immutable value instead of passing an ambiguous raw header string through an application.

The parser accepts a deliberately small ASCII grammar: one token-based
`type/subtype`, followed by unique token-named parameters whose values are
tokens or quoted strings. It normalizes names, preserves parameter-value case,
sorts parameters by normalized name, and emits one deterministic representation.

## When to Use

Use this value object at a boundary that intentionally requires a strict,
bounded media type rather than tolerant MIME recovery. It fits configuration,
protocol fixtures, and APIs where malformed or combined values must be rejected
explicitly. Use a standards-complete library when RFC 8187 extended parameters,
obsolete syntax, comments, negotiation lists, or application-specific media
semantics are required.

## Implementation

```python
import re
from dataclasses import dataclass


_MAX_MEDIA_TYPE_LENGTH = 1024
_MAX_PARAMETERS = 16
_TOKEN = re.compile(r"[!#$%&'*+\-.^_`|~0-9A-Za-z]+", re.ASCII)


class MediaTypeParseError(ValueError):
    pass


def _normalize_token(value: str, *, name: str) -> str:
    if not isinstance(value, str):
        raise TypeError(f"{name} must be text")
    if not 1 <= len(value) <= _MAX_MEDIA_TYPE_LENGTH:
        raise ValueError(f"{name} length is outside the supported range")
    if _TOKEN.fullmatch(value) is None:
        raise ValueError(f"{name} must be an ASCII token")
    return value.lower()


def _validate_parameter_value(value: str) -> str:
    if not isinstance(value, str):
        raise TypeError("parameter values must be text")
    if len(value) > _MAX_MEDIA_TYPE_LENGTH:
        raise ValueError("parameter value exceeds the supported length")
    if any(not 0x20 <= ord(character) <= 0x7E for character in value):
        raise ValueError("parameter values must contain printable ASCII")
    return value


def _serialize_parameter_value(value: str) -> str:
    if value and _TOKEN.fullmatch(value) is not None:
        return value
    escaped = value.replace("\\", "\\\\").replace('"', '\\"')
    return f'"{escaped}"'


@dataclass(frozen=True, slots=True)
class MediaType:
    type: str
    subtype: str
    parameters: tuple[tuple[str, str], ...] = ()

    def __post_init__(self) -> None:
        object.__setattr__(
            self,
            "type",
            _normalize_token(self.type, name="type"),
        )
        object.__setattr__(
            self,
            "subtype",
            _normalize_token(self.subtype, name="subtype"),
        )
        if not isinstance(self.parameters, tuple):
            raise TypeError("parameters must be a tuple")
        if len(self.parameters) > _MAX_PARAMETERS:
            raise ValueError("parameters exceed the supported count")

        normalized: list[tuple[str, str]] = []
        names: set[str] = set()
        for parameter in self.parameters:
            if not isinstance(parameter, tuple) or len(parameter) != 2:
                raise TypeError("each parameter must be a name-value tuple")
            raw_name, raw_value = parameter
            parameter_name = _normalize_token(
                raw_name,
                name="parameter name",
            )
            if parameter_name in names:
                raise ValueError("parameter names must be unique")
            names.add(parameter_name)
            normalized.append(
                (parameter_name, _validate_parameter_value(raw_value))
            )
        normalized.sort(key=lambda item: item[0])
        object.__setattr__(self, "parameters", tuple(normalized))
        if len(self.serialize()) > _MAX_MEDIA_TYPE_LENGTH:
            raise ValueError("serialized media type exceeds the supported length")

    def parameter(self, name: str) -> str | None:
        normalized_name = _normalize_token(name, name="parameter name")
        for parameter_name, value in self.parameters:
            if parameter_name == normalized_name:
                return value
        return None

    def serialize(self) -> str:
        suffix = "".join(
            f"; {name}={_serialize_parameter_value(value)}"
            for name, value in self.parameters
        )
        return f"{self.type}/{self.subtype}{suffix}"


def _read_token(text: str, position: int) -> tuple[str, int]:
    match = _TOKEN.match(text, position)
    if match is None:
        raise MediaTypeParseError("expected an ASCII token")
    return match.group(0), match.end()


def _skip_optional_whitespace(text: str, position: int) -> int:
    while position < len(text) and text[position] in " \t":
        position += 1
    return position


def _read_quoted_value(text: str, position: int) -> tuple[str, int]:
    position += 1
    output: list[str] = []
    while position < len(text):
        character = text[position]
        if character == '"':
            return "".join(output), position + 1
        if character == "\\":
            position += 1
            if position >= len(text) or text[position] not in {'"', "\\"}:
                raise MediaTypeParseError("unsupported quoted-string escape")
            character = text[position]
        output.append(character)
        position += 1
    raise MediaTypeParseError("unterminated quoted-string")


def parse_media_type(text: str) -> MediaType:
    if not isinstance(text, str):
        raise TypeError("media type must be text")
    if not 1 <= len(text) <= _MAX_MEDIA_TYPE_LENGTH:
        raise ValueError("media type length is outside the supported range")
    if any(
        ord(character) >= 0x80
        or ord(character) == 0x7F
        or (ord(character) < 0x20 and character != "\t")
        for character in text
    ):
        raise ValueError("media type must contain supported ASCII characters")

    type_name, position = _read_token(text, 0)
    if position >= len(text) or text[position] != "/":
        raise MediaTypeParseError("expected a type/subtype separator")
    subtype, position = _read_token(text, position + 1)

    parameters: list[tuple[str, str]] = []
    names: set[str] = set()
    while True:
        position = _skip_optional_whitespace(text, position)
        if position == len(text):
            return MediaType(type_name, subtype, tuple(parameters))
        if text[position] != ";":
            raise MediaTypeParseError("expected a parameter separator")
        position = _skip_optional_whitespace(text, position + 1)
        if len(parameters) >= _MAX_PARAMETERS:
            raise MediaTypeParseError("parameters exceed the supported count")

        raw_name, position = _read_token(text, position)
        parameter_name = raw_name.lower()
        if parameter_name in names:
            raise MediaTypeParseError("parameter names must be unique")
        if position >= len(text) or text[position] != "=":
            raise MediaTypeParseError("expected a parameter value")
        position += 1

        if position < len(text) and text[position] == '"':
            value, position = _read_quoted_value(text, position)
        else:
            value, position = _read_token(text, position)
        names.add(parameter_name)
        parameters.append((parameter_name, value))
```

## Example

```python
media_type = parse_media_type(
    'Application/JSON; Charset="UTF-8"; note="a; \\"quoted\\""'
)
serialized = media_type.serialize()
round_trip = parse_media_type(serialized)
reordered_parameters = parse_media_type(
    'application/json; note="a; \\"quoted\\""; charset=UTF-8'
)

invalid_values = (
    "text/plain;",
    "text/plain; charset =utf-8",
    "text/plain; charset=utf-8; CHARSET=ascii",
    "text/plain, application/json",
    'text/plain; note="unterminated',
    'text/plain; note="bad\\escape"',
    "text/plain\r\nX-Test: value",
)
rejected = []
for value in invalid_values:
    try:
        parse_media_type(value)
    except ValueError:
        rejected.append(value)

assert (
    media_type.type,
    media_type.subtype,
    media_type.parameter("CHARSET"),
    media_type.parameter("note"),
    serialized,
    round_trip == media_type,
    reordered_parameters == media_type,
    tuple(rejected),
) == (
    "application",
    "json",
    "UTF-8",
    'a; "quoted"',
    'application/json; charset=UTF-8; note="a; \\"quoted\\""',
    True,
    True,
    invalid_values,
)
```

## Trade-offs and Limitations

This parser implements a strict ASCII subset, not every MIME or HTTP recovery
rule. It excludes non-ASCII field content, obsolete text, comments,
comma-separated negotiation values, and escape forms other than quoted `"` and
`\`. Token-shaped extended-parameter forms remain opaque; the parser does not
apply RFC 8187 decoding semantics. It validates syntax but not media-specific
rules such as required parameters or parameter-value case semantics. The
1,024-character and 16-parameter ceilings are application safety bounds.
Canonical serialization can change the original spelling, quoting, and
parameter order, so it must not be used to reconstruct bytes for an existing
message signature.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Bounded Host and Port with Bracketed IPv6](parse-a-bounded-host-and-port-with-bracketed-ipv6.md)
- [Collect Expected Parse Failures Without Stopping a Batch](../data-processing/collect-expected-parse-failures-without-stopping-a-batch.md)
- [Split Quoted and Bracketed Log Fields](../data-processing/split-quoted-and-bracketed-log-fields.md)
<!-- catalog:related:end -->
