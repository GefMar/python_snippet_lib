---
title: "Normalize Bounded Named Options with Explicit Default Semantics"
snippet_type: recipe
use_cases:
  - configuration
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../python-language/apply-partial-dataclass-updates-with-an-omitted-value-sentinel.md
  - ../python-language/build-a-read-only-mapping-with-canonical-text-keys.md
  - reject-unknown-options-with-conservative-typo-suggestions.md
---

# Normalize Bounded Named Options with Explicit Default Semantics

## Idea and Problem

Normalize a closed mapping of named scalar options only after validating a complete ordered schema with explicit missing, default, and nullable semantics.

A private sentinel separates “no default” from a default of `None`. Every rule
is either required with no default or optional with an explicit default, and
nullability independently controls whether a present value may be `None`.
Exact runtime type checks keep booleans from satisfying integer rules.

## When to Use

Use this recipe at a small in-memory configuration boundary where callers
already provide Python scalar values and schema order must determine the result
order. The ordinary pure function fits one closed option set without wrapping a
callable, changing a signature, or introducing decorator lifecycle concerns.

Use a schema library when inputs need parsing, aliases, nested structures,
constraints beyond exact scalar types, or accumulated diagnostics. Parse text
and authorize sensitive options before this normalization boundary.

## Implementation

```python
import re
from dataclasses import dataclass


Scalar = bool | int | float | str
_MISSING = object()
_SCALAR_TYPES = (bool, int, float, str)
_MAX_RULES = 32
_MAX_RAW_OPTIONS = 32
_OPTION_NAME = re.compile(r"[a-z][a-z0-9_]{0,31}", re.ASCII).fullmatch


@dataclass(frozen=True, slots=True)
class OptionRule:
    name: str
    value_type: type
    required: bool
    nullable: bool = False
    default: object = _MISSING


@dataclass(frozen=True, slots=True)
class NormalizedOption:
    name: str
    value: Scalar | None


def _validated_rules(
    value: object,
) -> tuple[tuple[OptionRule, ...], frozenset[str]]:
    if type(value) is not tuple:
        raise TypeError("rules must be an exact tuple")
    if not 1 <= len(value) <= _MAX_RULES:
        raise ValueError("rule count is outside the supported range")

    validated = []
    names: set[str] = set()
    for rule in value:
        if type(rule) is not OptionRule:
            raise TypeError("rules must contain exact OptionRule values")
        if type(rule.name) is not str:
            raise TypeError("option names must be exact strings")
        if _OPTION_NAME(rule.name) is None:
            raise ValueError("option names must be conservative ASCII identifiers")
        if rule.name in names:
            raise ValueError("option names must be unique")
        names.add(rule.name)

        if not any(rule.value_type is candidate for candidate in _SCALAR_TYPES):
            raise TypeError("value_type must be bool, int, float, or str")
        if type(rule.required) is not bool or type(rule.nullable) is not bool:
            raise TypeError("required and nullable must be exact booleans")

        has_default = rule.default is not _MISSING
        if rule.required and has_default:
            raise ValueError("a required option cannot have a default")
        if not rule.required and not has_default:
            raise ValueError("an optional option must have an explicit default")
        if has_default:
            if rule.default is None:
                if not rule.nullable:
                    raise ValueError("a None default requires nullable=True")
            elif type(rule.default) is not rule.value_type:
                raise TypeError("a default must have the rule's exact scalar type")

        validated.append(rule)
    return tuple(validated), frozenset(names)


def normalize_named_options(
    rules: tuple[OptionRule, ...],
    raw: dict[str, object],
) -> tuple[NormalizedOption, ...]:
    validated_rules, known_names = _validated_rules(rules)

    if type(raw) is not dict:
        raise TypeError("raw options must be an exact dictionary")
    if len(raw) > _MAX_RAW_OPTIONS:
        raise ValueError("raw option count exceeds the supported limit")
    for name in raw:
        if type(name) is not str:
            raise TypeError("raw option names must be exact strings")
        if _OPTION_NAME(name) is None:
            raise ValueError("raw option names must be conservative ASCII identifiers")
    if any(name not in known_names for name in raw):
        raise ValueError("raw options contain an unknown name")
    if any(rule.required and rule.name not in raw for rule in validated_rules):
        raise ValueError("a required option is missing")

    normalized = []
    for rule in validated_rules:
        if rule.name in raw:
            value = raw[rule.name]
        else:
            value = rule.default
            if value is _MISSING:
                raise AssertionError("a validated rule has no default")
        if value is None:
            if not rule.nullable:
                raise ValueError(f"option {rule.name!r} does not allow None")
        elif type(value) is not rule.value_type:
            raise TypeError(f"option {rule.name!r} has the wrong exact type")
        normalized.append(NormalizedOption(rule.name, value))
    return tuple(normalized)
```

## Example

```python
rules = (
    OptionRule("host", str, required=True),
    OptionRule("retries", int, required=False, default=3),
    OptionRule(
        "label",
        str,
        required=False,
        nullable=True,
        default="automatic",
    ),
    OptionRule("enabled", bool, required=False, default=True),
)
raw = {"label": None, "host": "worker"}
before = raw.copy()

normalized = normalize_named_options(rules, raw)
defaulted = normalize_named_options(rules, {"host": "worker"})

try:
    normalize_named_options(rules, {"host": "worker", "retries": True})
except TypeError:
    bool_as_int_rejected = True
else:
    bool_as_int_rejected = False

try:
    normalize_named_options(rules, {"enabled": False})
except ValueError:
    missing_required_rejected = True
else:
    missing_required_rejected = False

assert (
    normalized,
    defaulted[2],
    raw == before,
    bool_as_int_rejected,
    missing_required_rejected,
) == (
    (
        NormalizedOption("host", "worker"),
        NormalizedOption("retries", 3),
        NormalizedOption("label", None),
        NormalizedOption("enabled", True),
    ),
    NormalizedOption("label", "automatic"),
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

Schema and input validation are linear in at most 32 names, and the result is
a tuple of frozen records in schema order. The function validates the complete
schema before inspecting the raw dictionary, then rejects every unknown or
missing required name before building a result. Accepted values and defaults
are exact immutable built-in scalars or allowed `None`, so returning their
references needs neither copying nor ownership transfer.

The closed type set has no subclasses, enums, bytes, collections, or custom
validators. Float finiteness, numeric ranges, and text length remain separate
domain rules. The function performs no coercion, aliases, typo correction,
recursive merge, deep copy, mutation, I/O, or environment lookup. Adding an
option changes a complete closed schema and should be versioned accordingly.

## Related Snippets

<!-- catalog:related:start -->
- [Apply Partial Dataclass Updates with an Omitted-Value Sentinel](../python-language/apply-partial-dataclass-updates-with-an-omitted-value-sentinel.md)
- [Build a Read-Only Mapping with Canonical Text Keys](../python-language/build-a-read-only-mapping-with-canonical-text-keys.md)
- [Reject Unknown Options with Conservative Typo Suggestions](reject-unknown-options-with-conservative-typo-suggestions.md)
<!-- catalog:related:end -->
