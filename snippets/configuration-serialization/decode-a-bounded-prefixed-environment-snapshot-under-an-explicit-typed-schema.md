---
title: "Decode a Bounded Prefixed Environment Snapshot under an Explicit Typed Schema"
snippet_type: recipe
use_cases:
  - configuration
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - normalize-bounded-named-options-with-explicit-default-semantics.md
  - resolve-bounded-configuration-through-dependent-adapters.md
  - parse-a-bounded-ini-document-under-a-closed-no-interpolation-schema.md
---

# Decode a Bounded Prefixed Environment Snapshot under an Explicit Typed Schema

## Idea and Problem

Decode one caller-supplied environment snapshot through a small prefix-scoped schema without reading process-global state or exposing invalid values in errors.

The prefix selects owned keys while unrelated entries remain untouched. Every
owned suffix must be declared, so a typo cannot disappear silently. Required
and optional absence are explicit, empty text remains data, and integer,
Boolean, and enum values use closed case-sensitive grammars.

## When to Use

Use this recipe after a bootstrap layer has frozen environment strings into an
immutable bounded tuple. It fits small flat application settings whose prefix,
field order, types, requiredness, and enum choices are known before decoding.

Keep secret acquisition, redaction, default layering, interpolation, nested
configuration, reloads, and direct `os.environ` access outside this boundary.
The returned values can still be sensitive; do not log either the input or the
decoded result merely because parsing errors omit raw values.

## Implementation

```python
import re
from dataclasses import dataclass
from enum import Enum, StrEnum

_MIN_INT64 = -(2**63)
_MAX_INT64 = 2**63 - 1
_MAX_FIELDS = 64
_MAX_ENTRIES = 256
_MAX_VALUE_BYTES = 4_096
_MAX_AGGREGATE_VALUE_BYTES = 262_144
_MAX_ENUM_CHOICES = 32
_MAX_ENUM_CHOICE_BYTES = 64
_PREFIX = re.compile(r"[A-Z][A-Z0-9_]{0,31}_", re.ASCII).fullmatch
_LOCAL_NAME = re.compile(r"[A-Z][A-Z0-9_]{0,31}", re.ASCII).fullmatch
_FULL_NAME = re.compile(r"[A-Z][A-Z0-9_]{0,95}", re.ASCII).fullmatch
_INTEGER = re.compile(r"(?:0|-?[1-9][0-9]{0,18})", re.ASCII).fullmatch


class EnvironmentKind(StrEnum):
    TEXT = "text"
    INTEGER = "integer"
    BOOLEAN = "boolean"
    ENUM = "enum"


class _AbsentValue(Enum):
    ABSENT = object()


ABSENT = _AbsentValue.ABSENT


@dataclass(frozen=True, slots=True)
class EnvironmentField:
    name: str
    kind: EnvironmentKind
    required: bool
    choices: tuple[str, ...] = ()


@dataclass(frozen=True, slots=True)
class EnvironmentEntry:
    name: str
    value: str


@dataclass(frozen=True, slots=True)
class DecodedEnvironmentValue:
    name: str
    value: object


def _utf8_size(value: str, *, field: str, maximum: int) -> int:
    try:
        size = len(value.encode("utf-8"))
    except UnicodeEncodeError:
        raise ValueError(f"{field} must be UTF-8 encodable") from None
    if size > maximum:
        raise ValueError(f"{field} exceeds its UTF-8 byte limit")
    return size


def _validated_schema(
    prefix: object,
    fields: object,
) -> tuple[str, tuple[EnvironmentField, ...]]:
    if type(prefix) is not str:
        raise TypeError("prefix must be an exact string")
    if _PREFIX(prefix) is None:
        raise ValueError("prefix is outside the closed uppercase grammar")
    if type(fields) is not tuple:
        raise TypeError("fields must be an exact tuple")
    if not 1 <= len(fields) <= _MAX_FIELDS:
        raise ValueError("field count is outside 1..64")

    seen_names: set[str] = set()
    for index, specification in enumerate(fields):
        field = f"fields[{index}]"
        if type(specification) is not EnvironmentField:
            raise TypeError(f"{field} must be an exact EnvironmentField")
        if type(specification.name) is not str:
            raise TypeError(f"{field}.name must be an exact string")
        if _LOCAL_NAME(specification.name) is None:
            raise ValueError(f"{field}.name is outside the closed grammar")
        if specification.name in seen_names:
            raise ValueError("field names must be unique")
        seen_names.add(specification.name)
        if type(specification.kind) is not EnvironmentKind:
            raise TypeError(f"{field}.kind must be an exact EnvironmentKind")
        if type(specification.required) is not bool:
            raise TypeError(f"{field}.required must be an exact boolean")
        if type(specification.choices) is not tuple:
            raise TypeError(f"{field}.choices must be an exact tuple")

        if specification.kind is EnvironmentKind.ENUM:
            if not 1 <= len(specification.choices) <= _MAX_ENUM_CHOICES:
                raise ValueError(f"{field}.choices count is outside 1..32")
            for choice_index, choice in enumerate(specification.choices):
                choice_field = f"{field}.choices[{choice_index}]"
                if type(choice) is not str:
                    raise TypeError(f"{choice_field} must be an exact string")
                if not choice:
                    raise ValueError(f"{choice_field} must not be empty")
                _utf8_size(
                    choice,
                    field=choice_field,
                    maximum=_MAX_ENUM_CHOICE_BYTES,
                )
        elif specification.choices:
            raise ValueError("only enum fields can declare choices")
    return prefix, fields


def _decode_value(
    raw: str,
    specification: EnvironmentField,
    *,
    full_name: str,
) -> object:
    if specification.kind is EnvironmentKind.TEXT:
        return raw
    if specification.kind is EnvironmentKind.INTEGER:
        if _INTEGER(raw) is None:
            raise ValueError(f"{full_name} is not a canonical signed integer")
        value = int(raw)
        if not _MIN_INT64 <= value <= _MAX_INT64:
            raise ValueError(f"{full_name} is outside the signed 64-bit range")
        return value
    if specification.kind is EnvironmentKind.BOOLEAN:
        if raw == "true":
            return True
        if raw == "false":
            return False
        raise ValueError(f"{full_name} is not the exact Boolean true or false")
    if raw not in specification.choices:
        raise ValueError(f"{full_name} is not one of the declared enum choices")
    return raw


def decode_environment_snapshot(
    prefix: str,
    fields: tuple[EnvironmentField, ...],
    entries: tuple[EnvironmentEntry, ...],
) -> tuple[DecodedEnvironmentValue, ...]:
    checked_prefix, checked_fields = _validated_schema(prefix, fields)
    if type(entries) is not tuple:
        raise TypeError("entries must be an exact tuple")
    if len(entries) > _MAX_ENTRIES:
        raise ValueError("entry count exceeds the supported limit")

    seen_entries: set[str] = set()
    owned_values: dict[str, str] = {}
    declared_names = {field.name for field in checked_fields}
    aggregate_value_bytes = 0
    for index, entry in enumerate(entries):
        field = f"entries[{index}]"
        if type(entry) is not EnvironmentEntry:
            raise TypeError(f"{field} must be an exact EnvironmentEntry")
        if type(entry.name) is not str:
            raise TypeError(f"{field}.name must be an exact string")
        if _FULL_NAME(entry.name) is None:
            raise ValueError(f"{field}.name is outside the closed uppercase grammar")
        if entry.name in seen_entries:
            raise ValueError(f"{field}.name duplicates a snapshot key")
        seen_entries.add(entry.name)
        if type(entry.value) is not str:
            raise TypeError(f"{field}.value must be an exact string")
        aggregate_value_bytes += _utf8_size(
            entry.value,
            field=f"{field}.value",
            maximum=_MAX_VALUE_BYTES,
        )
        if aggregate_value_bytes > _MAX_AGGREGATE_VALUE_BYTES:
            raise ValueError("snapshot values exceed the aggregate UTF-8 byte limit")

        if entry.name.startswith(checked_prefix):
            local_name = entry.name[len(checked_prefix) :]
            if local_name not in declared_names:
                raise ValueError(f"{entry.name} is an unknown prefixed key")
            owned_values[local_name] = entry.value

    decoded: list[DecodedEnvironmentValue] = []
    for specification in checked_fields:
        full_name = f"{checked_prefix}{specification.name}"
        if specification.name not in owned_values:
            if specification.required:
                raise ValueError(f"required key {full_name} is absent")
            value = ABSENT
        else:
            value = _decode_value(
                owned_values[specification.name],
                specification,
                full_name=full_name,
            )
        decoded.append(DecodedEnvironmentValue(specification.name, value))
    return tuple(decoded)
```

## Example

```python
schema = (
    EnvironmentField("LABEL", EnvironmentKind.TEXT, True),
    EnvironmentField("WORKERS", EnvironmentKind.INTEGER, False),
    EnvironmentField("ENABLED", EnvironmentKind.BOOLEAN, True),
    EnvironmentField(
        "MODE",
        EnvironmentKind.ENUM,
        False,
        ("safe", "fast"),
    ),
)
snapshot = (
    EnvironmentEntry("OTHER_VALUE", "ignored"),
    EnvironmentEntry("APP_MODE", "safe"),
    EnvironmentEntry("APP_ENABLED", "false"),
    EnvironmentEntry("APP_LABEL", ""),
    EnvironmentEntry("APP_WORKERS", "12"),
)
expected = (
    DecodedEnvironmentValue("LABEL", ""),
    DecodedEnvironmentValue("WORKERS", 12),
    DecodedEnvironmentValue("ENABLED", False),
    DecodedEnvironmentValue("MODE", "safe"),
)


def every_snapshot_permutation_matches() -> bool:
    from itertools import permutations

    return all(
        decode_environment_snapshot("APP_", schema, declaration) == expected
        for declaration in permutations(snapshot)
    )


assert every_snapshot_permutation_matches()

optional = decode_environment_snapshot(
    "APP_",
    (EnvironmentField("LABEL", EnvironmentKind.TEXT, False),),
    (),
)
empty_text = decode_environment_snapshot(
    "APP_",
    (EnvironmentField("LABEL", EnvironmentKind.TEXT, False),),
    (EnvironmentEntry("APP_LABEL", ""),),
)

valid_integers = ("0", "1", "-1", str(_MIN_INT64), str(_MAX_INT64))
for raw in valid_integers:
    decoded = decode_environment_snapshot(
        "APP_",
        (EnvironmentField("VALUE", EnvironmentKind.INTEGER, True),),
        (EnvironmentEntry("APP_VALUE", raw),),
    )
    assert decoded[0].value == int(raw)

invalid_entries = (
    EnvironmentEntry("APP_VALUE", ""),
    EnvironmentEntry("APP_VALUE", "+1"),
    EnvironmentEntry("APP_VALUE", "01"),
    EnvironmentEntry("APP_VALUE", "DO_NOT_ECHO"),
)
for entry in invalid_entries:
    try:
        decode_environment_snapshot(
            "APP_",
            (EnvironmentField("VALUE", EnvironmentKind.INTEGER, True),),
            (entry,),
        )
    except ValueError as error:
        if entry.value:
            assert entry.value not in str(error)
    else:
        raise AssertionError("invalid integer was accepted")

try:
    decode_environment_snapshot(
        "APP_",
        schema,
        (*snapshot, EnvironmentEntry("APP_TYPO", "hidden")),
    )
except ValueError as error:
    unknown_rejected_without_value = "hidden" not in str(error)
else:
    unknown_rejected_without_value = False

maximum_text = "x" * _MAX_VALUE_BYTES
maximum_decoded = decode_environment_snapshot(
    "APP_",
    (EnvironmentField("TEXT", EnvironmentKind.TEXT, True),),
    (EnvironmentEntry("APP_TEXT", maximum_text),),
)
duplicate_enum_choice = decode_environment_snapshot(
    "APP_",
    (
        EnvironmentField(
            "MODE",
            EnvironmentKind.ENUM,
            True,
            ("safe", "safe"),
        ),
    ),
    (EnvironmentEntry("APP_MODE", "safe"),),
)

assert (
    optional == (DecodedEnvironmentValue("LABEL", ABSENT),)
    and empty_text == (DecodedEnvironmentValue("LABEL", ""),)
    and unknown_rejected_without_value
    and maximum_decoded == (DecodedEnvironmentValue("TEXT", maximum_text),)
    and duplicate_enum_choice == (DecodedEnvironmentValue("MODE", "safe"),)
)
```

## Trade-offs and Limitations

For `E` snapshot entries and `F` schema fields, validation and decoding take
`O(E + F + B)` time for `B` inspected UTF-8 bytes and `O(E + F)` memory. Input
order does not affect output order. Unrelated keys are validated and counted
toward limits but are not returned.

The decoder has no defaults, aliases, nested objects, list syntax, numeric
units, case folding, or interpolation. The explicit absence sentinel prevents
optional absence from becoming an empty string or `None`. Error text identifies
the responsible key and rule without echoing the raw value, but callers remain
responsible for protecting every input and decoded value elsewhere.

## Related Snippets

<!-- catalog:related:start -->
- [Normalize Bounded Named Options with Explicit Default Semantics](normalize-bounded-named-options-with-explicit-default-semantics.md)
- [Resolve Bounded Configuration Through Dependent Adapters](resolve-bounded-configuration-through-dependent-adapters.md)
- [Parse a Bounded INI Document Under a Closed No-Interpolation Schema](parse-a-bounded-ini-document-under-a-closed-no-interpolation-schema.md)
<!-- catalog:related:end -->
