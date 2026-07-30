---
title: "Serialize a Bounded INI Mapping Only When It Round-Trips Exactly"
snippet_type: recipe
use_cases:
  - configuration
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - parse-a-bounded-ini-document-under-a-closed-no-interpolation-schema.md
  - parse-bounded-toml-into-an-immutable-typed-tree.md
  - round-trip-a-bounded-nullable-string-table-with-csv-quote-notnull.md
---

# Serialize a Bounded INI Mapping Only When It Round-Trips Exactly

## Idea and Problem

Serialize one bounded, ordered INI model only if a fresh parser reconstructs every section, option, value, and ordering choice exactly.

`ConfigParser.write()` can now reject option names that would be read as a
section header or that contain a configured delimiter. A second parse also
catches other lossy names, such as an option that becomes a comment. Together
with immutable input records and explicit limits, this turns a permissive text
format into a narrow serialization boundary.

## When to Use

Use this recipe when a small in-memory mapping must be emitted as deterministic
INI text and both producer and consumer follow the same closed profile. The
profile preserves insertion order and case, uses only `=`, disables defaults
and interpolation, and permits printable single-line values without boundary
whitespace.

Choose a round-trip editing library when comments and original layout must be
retained. Choose a schema-aware format when values need native types, nested
structures, or an independently specified canonical representation.

## Implementation

```python
import configparser
import io
from dataclasses import dataclass

_MAX_SECTIONS = 16
_MAX_OPTIONS_PER_SECTION = 32
_MAX_TOTAL_OPTIONS = 128
_MAX_NAME_UTF8_BYTES = 128
_MAX_VALUE_UTF8_BYTES = 1_024
_MAX_MODEL_UTF8_BYTES = 12_288
_MAX_OUTPUT_UTF8_BYTES = 16_384
_DEFAULT_SECTION = "DEFAULT"


class IniSerializationError(ValueError):
    """Raised when a value is outside the closed serialization profile."""


@dataclass(frozen=True, slots=True)
class IniOption:
    name: str
    value: str


@dataclass(frozen=True, slots=True)
class IniSection:
    name: str
    options: tuple[IniOption, ...]


def _fresh_ini_parser() -> configparser.ConfigParser:
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
    return parser


def _validated_text(
    value: object,
    *,
    field: str,
    max_bytes: int,
    allow_empty: bool = False,
) -> tuple[str, int]:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if not value and not allow_empty:
        raise IniSerializationError(f"{field} must not be empty")
    if "\r" in value or "\n" in value:
        raise IniSerializationError(f"{field} must be a single line")
    if value != value.strip():
        raise IniSerializationError(f"{field} has lossy boundary whitespace")
    if any(not character.isprintable() for character in value):
        raise IniSerializationError(f"{field} contains a control character")
    try:
        encoded = value.encode("utf-8")
    except UnicodeEncodeError:
        raise IniSerializationError(f"{field} is not valid UTF-8 text") from None
    if len(encoded) > max_bytes:
        raise IniSerializationError(f"{field} exceeds its UTF-8 byte limit")
    return value, len(encoded)


def _validated_ini_model(value: object) -> tuple[IniSection, ...]:
    if type(value) is not tuple:
        raise TypeError("sections must be an exact tuple")
    if not 1 <= len(value) <= _MAX_SECTIONS:
        raise IniSerializationError("section count is outside the supported range")

    checked_sections: list[IniSection] = []
    section_names: set[str] = set()
    total_options = 0
    total_bytes = 0

    for raw_section in value:
        if type(raw_section) is not IniSection:
            raise TypeError("sections must contain exact IniSection values")
        section_name, byte_count = _validated_text(
            raw_section.name,
            field="section name",
            max_bytes=_MAX_NAME_UTF8_BYTES,
        )
        total_bytes += byte_count
        if total_bytes > _MAX_MODEL_UTF8_BYTES:
            raise IniSerializationError("the model exceeds its UTF-8 byte limit")
        if section_name == _DEFAULT_SECTION:
            raise IniSerializationError("the default section is not supported")
        if section_name in section_names:
            raise IniSerializationError("section names must be unique")
        section_names.add(section_name)

        if type(raw_section.options) is not tuple:
            raise TypeError("section options must be exact tuples")
        if len(raw_section.options) > _MAX_OPTIONS_PER_SECTION:
            raise IniSerializationError("a section has too many options")

        checked_options: list[IniOption] = []
        option_names: set[str] = set()
        for raw_option in raw_section.options:
            if type(raw_option) is not IniOption:
                raise TypeError("options must contain exact IniOption values")
            option_name, name_bytes = _validated_text(
                raw_option.name,
                field="option name",
                max_bytes=_MAX_NAME_UTF8_BYTES,
            )
            option_value, value_bytes = _validated_text(
                raw_option.value,
                field="option value",
                max_bytes=_MAX_VALUE_UTF8_BYTES,
                allow_empty=True,
            )
            total_bytes += name_bytes + value_bytes
            total_options += 1
            if total_options > _MAX_TOTAL_OPTIONS:
                raise IniSerializationError("the model has too many options")
            if total_bytes > _MAX_MODEL_UTF8_BYTES:
                raise IniSerializationError("the model exceeds its UTF-8 byte limit")
            if option_name in option_names:
                raise IniSerializationError("option names within a section must be unique")
            option_names.add(option_name)
            checked_options.append(IniOption(option_name, option_value))

        checked_sections.append(IniSection(section_name, tuple(checked_options)))

    return tuple(checked_sections)


def _model_from_parser(
    parser: configparser.ConfigParser,
) -> tuple[IniSection, ...]:
    return tuple(
        IniSection(
            section_name,
            tuple(
                IniOption(option_name, option_value)
                for option_name, option_value in parser.items(
                    section_name,
                    raw=True,
                )
            ),
        )
        for section_name in parser.sections()
    )


def serialize_bounded_ini_mapping(sections: tuple[IniSection, ...]) -> str:
    checked_sections = _validated_ini_model(sections)
    parser = _fresh_ini_parser()
    for section in checked_sections:
        parser.add_section(section.name)
        for option in section.options:
            parser.set(section.name, option.name, option.value)

    stream = io.StringIO()
    try:
        parser.write(stream, space_around_delimiters=False)
    except configparser.InvalidWriteError as error:
        raise IniSerializationError("an option name is unsafe to serialize") from error

    output = stream.getvalue()
    if len(output.encode("utf-8")) > _MAX_OUTPUT_UTF8_BYTES:
        raise IniSerializationError("serialized INI exceeds its UTF-8 byte limit")

    reparsed = _fresh_ini_parser()
    try:
        reparsed.read_string(output, source="<serialized-ini>")
    except configparser.Error as error:
        raise IniSerializationError("serialized INI cannot be parsed back") from error
    if reparsed.defaults() or _model_from_parser(reparsed) != checked_sections:
        raise IniSerializationError("serialized INI does not round-trip exactly")
    return output


```

## Example

```python
model = (
    IniSection(
        "service",
        (
            IniOption("Host", "api.example"),
            IniOption("Retries", "3"),
        ),
    ),
    IniSection("feature-flags", (IniOption("Enabled", "yes"),)),
)
expected = "[service]\nHost=api.example\nRetries=3\n\n[feature-flags]\nEnabled=yes\n\n"

invalid_models = (
    (IniSection(" service", (IniOption("name", "value"),)),),
    (IniSection("service", (IniOption("name", " padded "),)),),
    (IniSection("service", (IniOption("[shadow]", "value"),)),),
    (IniSection("service", (IniOption("name=alias", "value"),)),),
    (IniSection("service", (IniOption("#shadow", "value"),)),),
    (
        IniSection("service", (IniOption("one", "1"),)),
        IniSection("service", (IniOption("two", "2"),)),
    ),
    (
        IniSection(
            "service",
            (IniOption("name", "one"), IniOption("name", "two")),
        ),
    ),
    (
        IniSection(
            "service",
            (IniOption("payload", "x" * (_MAX_VALUE_UTF8_BYTES + 1)),),
        ),
    ),
    (
        IniSection(
            "filled",
            tuple(IniOption(f"k{index:02d}", "x" * 1_020) for index in range(12)),
        ),
        IniSection("overflow", ()),
    ),
)
rejected = 0
for invalid_model in invalid_models:
    try:
        serialize_bounded_ini_mapping(invalid_model)
    except IniSerializationError:
        rejected += 1

assert (serialize_bounded_ini_mapping(model), rejected) == (expected, 9)
```

## Trade-offs and Limitations

This profile holds the complete model, serialized text, and reparsed model in
memory under fixed count and UTF-8 byte limits. It rejects duplicate names,
the default section, multiline or control-containing fields, and boundary
whitespace that `ConfigParser` would trim. Section and option order and exact
case are semantic parts of the model.

The writer rejects keys containing `=` or beginning like a section header. The
exact round-trip comparison catches other values that the INI grammar would
reinterpret or discard. The helper deliberately returns no partial output
after either kind of failure.

This is not a general INI formatter. It does not preserve comments, accept
defaults, interpolate values, merge documents, normalize equivalent input, or
provide typed values. Returning text also does not publish a file atomically or
durably. Both ends must agree on this narrow profile and on the same Python
behavior.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Bounded INI Document Under a Closed No-Interpolation Schema](parse-a-bounded-ini-document-under-a-closed-no-interpolation-schema.md)
- [Parse Bounded TOML into an Immutable Typed Tree](parse-bounded-toml-into-an-immutable-typed-tree.md)
- [Round-Trip a Bounded Nullable String Table with CSV Quote-Not-Null](round-trip-a-bounded-nullable-string-table-with-csv-quote-notnull.md)
<!-- catalog:related:end -->
