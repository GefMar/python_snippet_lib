---
title: "Scan Bounded Macro Declarations into a Canonical Event Index"
snippet_type: algorithm
use_cases:
  - parsing
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - parse-a-bounded-space-indented-test-outline-into-leaf-paths.md
  - ../data-processing/count-static-imports-across-bounded-python-notebook-cells.md
  - ../configuration-serialization/parse-a-bounded-flat-placeholder-template.md
---

# Scan Bounded Macro Declarations into a Canonical Event Index

## Idea and Problem

Scan a small tuple of named text units for two exact macro-shaped declarations and return one immutable, order-independent event index.

An event implicitly belongs to the most recent group in the same unit. Resetting
that state at each unit boundary prevents a missing group declaration from
borrowing context from an earlier unit. Hard aggregate and per-line limits keep
the scanner's work predictable before it builds the canonical result.

## When to Use

Use this algorithm when a controlled generator emits the exact full-line forms
`EVENT_GROUP("name")` and `EVENT("name")`, with conservative lowercase ASCII
names. The caller should already have decoded the bounded text and assigned a
stable, unique name to every unit. Nonmatching lines are deliberately ignored.

Use a real parser for any language-aware inventory. This scanner is suitable
only when the tiny grammar is itself the producer-consumer contract and an
event before the first group in any unit is unambiguously invalid.

## Implementation

```python
import re
from dataclasses import dataclass


_MAX_UNITS = 64
_MAX_UTF8_BYTES = 512 * 1024
_MAX_LINES = 8_192
_MAX_LINE_BYTES = 512
_MAX_EVENT_DECLARATIONS = 4_096
_UNIT_NAME = re.compile(r"[a-z][a-z0-9_.-]{0,63}", re.ASCII)
_GROUP_LINE = re.compile(
    r'EVENT_GROUP\("(?P<name>[a-z][a-z0-9_]{0,31})"\)',
    re.ASCII,
)
_EVENT_LINE = re.compile(
    r'EVENT\("(?P<name>[a-z][a-z0-9_]{0,31})"\)',
    re.ASCII,
)


class MacroDeclarationError(ValueError):
    pass


@dataclass(frozen=True, slots=True)
class TextUnit:
    name: str
    text: str


@dataclass(frozen=True, slots=True)
class IndexedEvent:
    group: str
    name: str


@dataclass(frozen=True, slots=True)
class MacroEventIndex:
    groups: tuple[str, ...]
    events: tuple[IndexedEvent, ...]


def _validate_units(
    units: tuple[TextUnit, ...],
) -> tuple[tuple[TextUnit, tuple[str, ...]], ...]:
    if type(units) is not tuple:
        raise TypeError("units must be an exact tuple")
    if not 1 <= len(units) <= _MAX_UNITS:
        raise ValueError("unit count is outside the supported range")

    seen_names: set[str] = set()
    total_bytes = 0
    total_lines = 0
    validated: list[tuple[TextUnit, tuple[str, ...]]] = []

    for unit in units:
        if type(unit) is not TextUnit:
            raise TypeError("units must contain exact TextUnit values")
        if type(unit.name) is not str or _UNIT_NAME.fullmatch(unit.name) is None:
            raise ValueError("unit names must be conservative ASCII names")
        if unit.name in seen_names:
            raise ValueError(f"duplicate unit name: {unit.name!r}")
        if type(unit.text) is not str:
            raise TypeError("unit text must be exact text")
        remaining_bytes = _MAX_UTF8_BYTES - total_bytes
        if len(unit.text) > remaining_bytes:
            raise ValueError("unit text exceeds the aggregate UTF-8 byte limit")
        if "\r" in unit.text:
            raise MacroDeclarationError(
                f"{unit.name}: only LF line separators are supported"
            )

        try:
            encoded = unit.text.encode("utf-8")
        except UnicodeEncodeError as error:
            raise MacroDeclarationError(
                f"{unit.name}: text is not encodable as UTF-8"
            ) from error
        encoded_size = len(encoded)
        if encoded_size > remaining_bytes:
            raise ValueError("unit text exceeds the aggregate UTF-8 byte limit")
        total_bytes += encoded_size

        remaining_lines = _MAX_LINES - total_lines
        lines = tuple(unit.text.split("\n", remaining_lines))
        if len(lines) > remaining_lines:
            raise ValueError("unit text exceeds the aggregate line limit")
        total_lines += len(lines)
        for line_number, line in enumerate(lines, start=1):
            if len(line.encode("utf-8")) > _MAX_LINE_BYTES:
                raise ValueError(
                    f"{unit.name}:{line_number}: line exceeds the byte limit"
                )

        seen_names.add(unit.name)
        validated.append((unit, lines))

    return tuple(validated)


def scan_macro_event_index(
    units: tuple[TextUnit, ...],
) -> MacroEventIndex:
    validated_units = _validate_units(units)
    groups: set[str] = set()
    event_keys: set[tuple[str, str]] = set()
    event_declarations = 0

    for unit, lines in validated_units:
        current_group: str | None = None
        for line_number, line in enumerate(lines, start=1):
            group_match = _GROUP_LINE.fullmatch(line)
            if group_match is not None:
                current_group = group_match.group("name")
                groups.add(current_group)
                continue

            event_match = _EVENT_LINE.fullmatch(line)
            if event_match is None:
                continue
            if current_group is None:
                raise MacroDeclarationError(
                    f"{unit.name}:{line_number}: event appears before a group"
                )
            event_declarations += 1
            if event_declarations > _MAX_EVENT_DECLARATIONS:
                raise ValueError("event declaration count exceeds the limit")
            event_keys.add((current_group, event_match.group("name")))

    return MacroEventIndex(
        groups=tuple(sorted(groups)),
        events=tuple(
            IndexedEvent(group, name)
            for group, name in sorted(event_keys)
        ),
    )
```

## Example

```python
units = (
    TextUnit(
        "zeta-unit",
        'EVENT_GROUP("storage")\nEVENT("closed")\n',
    ),
    TextUnit(
        "alpha-unit",
        'EVENT_GROUP("api")\nEVENT("ready")\n'
        'EVENT_GROUP("storage")\nEVENT("opened")\n',
    ),
)

index = scan_macro_event_index(units)

try:
    scan_macro_event_index((TextUnit("orphan", 'EVENT("started")'),))
except MacroDeclarationError:
    orphan_rejected = True
else:
    orphan_rejected = False

assert (index, orphan_rejected) == (
    MacroEventIndex(
        groups=("api", "storage"),
        events=(
            IndexedEvent("api", "ready"),
            IndexedEvent("storage", "closed"),
            IndexedEvent("storage", "opened"),
        ),
    ),
    True,
)
```

## Trade-offs and Limitations

The result deduplicates equal group names and equal `(group, event)` pairs,
then sorts both tuples, so unit order does not affect a valid index. Declaration
order remains semantically significant because each event belongs to the most
recent group in its unit. The event limit still counts every matching
declaration before deduplication. Source locations are used for errors but
intentionally omitted from the canonical inventory.

This is not a C++ parser or a general-language parser. Leading or trailing
whitespace, comments, multiline invocations, quoted-string escapes, aliases,
and preprocessor constructs are outside the grammar; a line that does not
match either exact form is ignored. The helper does not read files, decode
JSON, expand macros, or determine whether ignored source text is valid.

All text is already resident in memory. Validation and scanning take linear
time in at most 64 units, 512 KiB of UTF-8 text, 8,192 logical lines, and 4,096
event declarations; sorting depends on the number of distinct indexed names.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Bounded Space-Indented Test Outline into Leaf Paths](parse-a-bounded-space-indented-test-outline-into-leaf-paths.md)
- [Count Static Imports Across Bounded Python Notebook Cells](../data-processing/count-static-imports-across-bounded-python-notebook-cells.md)
- [Parse a Bounded Flat Placeholder Template](../configuration-serialization/parse-a-bounded-flat-placeholder-template.md)
<!-- catalog:related:end -->
