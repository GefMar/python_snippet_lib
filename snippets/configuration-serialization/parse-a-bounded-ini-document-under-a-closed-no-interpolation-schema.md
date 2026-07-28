---
title: "Parse a Bounded INI Document Under a Closed No-Interpolation Schema"
snippet_type: recipe
use_cases:
  - configuration
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - reject-unknown-options-with-conservative-typo-suggestions.md
  - tokenize-a-bounded-posix-style-argument-string-without-expansion.md
  - parse-bounded-utf-8-json-without-duplicate-object-names-or-non-finite-numbers.md
---

# Parse a Bounded INI Document Under a Closed No-Interpolation Schema

## Idea and Problem

Parse one bounded INI document only after closing its syntax and its complete section-and-option vocabulary.

The profile accepts exact section headers and `name = value` lines, preserves
case, disables interpolation and defaults, rejects continuation lines, and
returns immutable values in schema order rather than document order.

## When to Use

Use this recipe when a small configuration boundary intentionally accepts one
narrow INI dialect and every required section and option is known in advance.
Values remain strings for a separate domain-specific validation step, so the
parser does not silently choose Boolean, numeric, path, or duration semantics.

Use a maintained schema or configuration library when optional fields,
defaults, includes, aliases, rich source locations, round-trip formatting, or
compatibility with a broader INI dialect is required.

## Implementation

```python
import configparser
import re
from dataclasses import dataclass

_MAX_DOCUMENT_BYTES = 16_384
_MAX_LINES = 256
_MAX_LINE_BYTES = 1_024
_MAX_SECTIONS = 16
_MAX_OPTIONS_PER_SECTION = 32
_MAX_TOTAL_OPTIONS = 128
_NAME_TEXT = r"[A-Za-z][A-Za-z0-9_-]{0,31}"
_NAME = re.compile(_NAME_TEXT, re.ASCII)
_SECTION_LINE = re.compile(rf"\[(?P<name>{_NAME_TEXT})\]", re.ASCII)
_OPTION_LINE = re.compile(
    rf"(?P<name>{_NAME_TEXT}) = (?P<value>.+)",
    re.ASCII,
)
_DEFAULT_SECTION = "DEFAULT"


class ClosedIniError(ValueError):
    """Raised when the schema or document violates the closed INI profile."""


@dataclass(frozen=True, slots=True)
class IniSectionSchema:
    name: str
    options: tuple[str, ...]


@dataclass(frozen=True, slots=True)
class ParsedIniSection:
    name: str
    options: tuple[tuple[str, str], ...]


def _validated_name(value: object, *, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if _NAME.fullmatch(value) is None:
        raise ClosedIniError(f"{field} is outside the supported name grammar")
    return value


def _validated_schema(
    value: object,
) -> tuple[IniSectionSchema, ...]:
    if type(value) is not tuple:
        raise TypeError("schema must be an exact tuple")
    if not 1 <= len(value) <= _MAX_SECTIONS:
        raise ClosedIniError("schema section count is outside the supported range")

    sections: list[IniSectionSchema] = []
    section_names: set[str] = set()
    total_options = 0

    for raw_section in value:
        if type(raw_section) is not IniSectionSchema:
            raise TypeError("schema must contain exact IniSectionSchema values")
        name = _validated_name(raw_section.name, field="section name")
        if name == _DEFAULT_SECTION:
            raise ClosedIniError("default section names are not supported")
        if name in section_names:
            raise ClosedIniError("schema section names must be unique")
        section_names.add(name)

        if type(raw_section.options) is not tuple:
            raise TypeError("schema options must be exact tuples")
        if len(raw_section.options) > _MAX_OPTIONS_PER_SECTION:
            raise ClosedIniError("a schema section has too many options")

        options: list[str] = []
        option_names: set[str] = set()
        for raw_option in raw_section.options:
            option = _validated_name(raw_option, field="option name")
            if option in option_names:
                raise ClosedIniError("option names within a section must be unique")
            option_names.add(option)
            options.append(option)
            total_options += 1
            if total_options > _MAX_TOTAL_OPTIONS:
                raise ClosedIniError("schema has too many options")
        sections.append(IniSectionSchema(name, tuple(options)))

    return tuple(sections)


def _validate_document_syntax(document: object) -> str:
    if type(document) is not str:
        raise TypeError("document must be an exact string")
    try:
        encoded = document.encode("utf-8")
    except UnicodeEncodeError:
        raise ClosedIniError("document must be valid UTF-8 text") from None
    if len(encoded) > _MAX_DOCUMENT_BYTES:
        raise ClosedIniError("document exceeds the supported byte size")
    if any(character != "\n" and not character.isprintable() for character in document):
        raise ClosedIniError("document contains unsupported control characters")

    lines = document.split("\n")
    if len(lines) > _MAX_LINES:
        raise ClosedIniError("document has too many lines")

    current_section: str | None = None
    for line in lines:
        if len(line.encode("utf-8")) > _MAX_LINE_BYTES:
            raise ClosedIniError("a document line exceeds the supported byte size")
        if not line or line.startswith(("#", ";")):
            continue

        section_match = _SECTION_LINE.fullmatch(line)
        if section_match is not None:
            current_section = section_match.group("name")
            if current_section == _DEFAULT_SECTION:
                raise ClosedIniError("default sections are not supported")
            continue

        option_match = _OPTION_LINE.fullmatch(line)
        if option_match is None or current_section is None:
            raise ClosedIniError("document is outside the closed INI syntax")
        option_value = option_match.group("value")
        if option_value != option_value.strip(" "):
            raise ClosedIniError("option values must not have boundary whitespace")

    return document


def parse_closed_ini_document(
    document: str,
    schema: tuple[IniSectionSchema, ...],
) -> tuple[ParsedIniSection, ...]:
    checked_schema = _validated_schema(schema)
    checked_document = _validate_document_syntax(document)

    parser = configparser.ConfigParser(
        allow_no_value=False,
        delimiters=("=",),
        comment_prefixes=("#", ";"),
        inline_comment_prefixes=None,
        strict=True,
        empty_lines_in_values=False,
        default_section=_DEFAULT_SECTION,
        interpolation=None,
        allow_unnamed_section=False,
    )
    parser.optionxform = str
    try:
        parser.read_string(checked_document, source="<bounded-ini>")
    except configparser.Error:
        raise ClosedIniError("document is not valid under the closed INI profile") from None

    if parser.defaults():
        raise ClosedIniError("default options are not supported")
    expected_sections = tuple(section.name for section in checked_schema)
    actual_sections = tuple(parser.sections())
    if len(actual_sections) != len(expected_sections) or set(actual_sections) != set(
        expected_sections
    ):
        raise ClosedIniError("document sections do not exactly match the schema")

    parsed_sections: list[ParsedIniSection] = []
    for section in checked_schema:
        actual_options = tuple(parser.options(section.name))
        if len(actual_options) != len(section.options) or set(actual_options) != set(
            section.options
        ):
            raise ClosedIniError(
                f"options in section {section.name!r} do not exactly match the schema"
            )
        parsed_sections.append(
            ParsedIniSection(
                name=section.name,
                options=tuple(
                    (
                        option,
                        parser.get(section.name, option, raw=True),
                    )
                    for option in section.options
                ),
            )
        )
    return tuple(parsed_sections)
```

## Example

```python
schema = (
    IniSectionSchema("service", ("host", "literal")),
    IniSectionSchema("features", ("enabled",)),
)
document = """# closed configuration
[features]
enabled = yes

[service]
literal = %(host)s # retained
host = api.example
"""
parsed = parse_closed_ini_document(document, schema)

invalid_documents = (
    "[DEFAULT]\nhost = inherited\n",
    "[service]\nhost = api.example\n continued\n",
    "[service]\nhost = one\nhost = two\n",
)
rejected = 0
for invalid_document in invalid_documents:
    try:
        parse_closed_ini_document(invalid_document, schema)
    except ClosedIniError:
        rejected += 1

assert (parsed, rejected) == (
    (
        ParsedIniSection(
            "service",
            (("host", "api.example"), ("literal", "%(host)s # retained")),
        ),
        ParsedIniSection("features", (("enabled", "yes"),)),
    ),
    3,
)
```

## Trade-offs and Limitations

This profile is intentionally narrower than general INI. Section headers have
no surrounding whitespace, option lines use exactly one space around `=`, and
names are case-sensitive ASCII identifiers. Values must be non-empty printable
single-line text without boundary spaces. `#` and `;` start comments only in
column zero and remain literal inside values. Blank lines are allowed, but
indentation, continuation lines, `:` delimiters, unnamed sections, default
inheritance, interpolation, valueless options, and duplicate names are not.

Parsing and validation use memory proportional to the bounded document. The
returned tuples and strings are immutable, but values remain untyped text and
receive no normalization. Schema order controls the result; document order is
not preserved. This helper does not merge files, apply defaults, expand
variables, retain comments, report line-oriented diagnostics, or write INI
text back out.

## Related Snippets

<!-- catalog:related:start -->
- [Reject Unknown Options with Conservative Typo Suggestions](reject-unknown-options-with-conservative-typo-suggestions.md)
- [Tokenize a Bounded POSIX-Style Argument String Without Expansion](tokenize-a-bounded-posix-style-argument-string-without-expansion.md)
- [Parse Bounded UTF-8 JSON Without Duplicate Object Names or Non-Finite Numbers](parse-bounded-utf-8-json-without-duplicate-object-names-or-non-finite-numbers.md)
<!-- catalog:related:end -->
