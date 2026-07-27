---
title: "Render Fixed Date Placeholders from an Explicit Anchor"
snippet_type: recipe
use_cases:
  - automation
  - configuration
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - parse-a-bounded-flat-placeholder-template.md
  - parse-compact-durations-into-timedelta.md
  - ../python-language/load-text-templates-from-package-resources.md
---

# Render Fixed Date Placeholders from an Explicit Anchor

## Idea and Problem

Render five fixed ISO date placeholders deterministically from exact date values and an explicit anchor instead of consulting an ambient clock.

The accepted fields are `today`, `yesterday`, `tomorrow`, `start`, and `end`.
Every field is a bare name inside one pair of braces, while doubled braces emit
literal braces. `start` and `end` are optional inputs but become mandatory when
their fields appear; adjacent-day arithmetic is performed only when its field
is referenced.

## When to Use

Use this recipe for a small controlled filename, report label, or configuration
value whose producers need only these calendar substitutions. Pass the anchor
from the job boundary or test fixture, and pass range dates only when the
template refers to them. Exact `date` checks prevent a `datetime` from silently
discarding its time or time-zone information.

Use a real template engine for general presentation and a dedicated scheduling
or calendar system for locale rules, business days, time zones, or recurring
events. Validate any surrounding filename, path, or URL after rendering with a
format-specific validator.

## Implementation

```python
from datetime import date, timedelta


_MAX_TEMPLATE_BYTES = 16_384
_MAX_OUTPUT_BYTES = 32_768
_MAX_PLACEHOLDERS = 512
_FIELDS = frozenset({"today", "yesterday", "tomorrow", "start", "end"})
_ONE_DAY = timedelta(days=1)


class DateTemplateError(ValueError):
    def __init__(self, position: int) -> None:
        self.position = position
        super().__init__(f"invalid date template at position {position}")


def _fail(position: int) -> None:
    raise DateTemplateError(position)


def _optional_exact_date(value: date | None, *, name: str) -> date | None:
    if value is not None and type(value) is not date:
        raise TypeError(f"{name} must be an exact date or None")
    return value


def render_fixed_date_placeholders(
    template: str,
    *,
    anchor: date,
    start: date | None = None,
    end: date | None = None,
) -> str:
    if type(template) is not str:
        raise TypeError("template must be text")
    if type(anchor) is not date:
        raise TypeError("anchor must be an exact date")
    start = _optional_exact_date(start, name="start")
    end = _optional_exact_date(end, name="end")

    if len(template) > _MAX_TEMPLATE_BYTES:
        _fail(_MAX_TEMPLATE_BYTES)
    try:
        template_size = len(template.encode("utf-8"))
    except UnicodeEncodeError as error:
        _fail(error.start)
    if template_size > _MAX_TEMPLATE_BYTES:
        _fail(len(template))

    pieces: list[str] = []
    literal: list[str] = []
    output_bytes = 0
    placeholder_count = 0

    def append_piece(value: str, position: int) -> None:
        nonlocal output_bytes
        output_bytes += len(value.encode("utf-8"))
        if output_bytes > _MAX_OUTPUT_BYTES:
            _fail(position)
        pieces.append(value)

    def flush_literal(position: int) -> None:
        if literal:
            append_piece("".join(literal), position)
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
            field = template[index + 1 : close]
            if field not in _FIELDS:
                _fail(index)
            placeholder_count += 1
            if placeholder_count > _MAX_PLACEHOLDERS:
                _fail(index)

            if field == "today":
                value = anchor
            elif field == "yesterday":
                try:
                    value = anchor - _ONE_DAY
                except OverflowError:
                    _fail(index)
            elif field == "tomorrow":
                try:
                    value = anchor + _ONE_DAY
                except OverflowError:
                    _fail(index)
            elif field == "start":
                if start is None:
                    _fail(index)
                value = start
            else:
                if end is None:
                    _fail(index)
                value = end

            append_piece(value.isoformat(), index)
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
    return "".join(pieces)
```

## Example

```python
from datetime import datetime


rendered = render_fixed_date_placeholders(
    "Café report {{daily}}: {yesterday} | {today} | {tomorrow}; "
    "range={start}..{end}",
    anchor=date(2026, 7, 27),
    start=date(2026, 7, 1),
    end=date(2026, 7, 31),
)
anchor_only = render_fixed_date_placeholders(
    "snapshot-{today}",
    anchor=date(2026, 7, 27),
)

hostile_templates = (
    "{today.year}",
    "{today[0]}",
    "{today!r}",
    "{today:%Y-%m-%d}",
    "{today:}",
    "{unknown}",
    "{today",
    "single } brace",
)
rejected = 0
for hostile in hostile_templates:
    try:
        render_fixed_date_placeholders(
            hostile,
            anchor=date(2026, 7, 27),
        )
    except DateTemplateError:
        rejected += 1

try:
    render_fixed_date_placeholders(
        "{start}..{end}",
        anchor=date(2026, 7, 27),
        start=date(2026, 7, 1),
    )
except DateTemplateError:
    missing_range_rejected = True
else:
    missing_range_rejected = False

try:
    render_fixed_date_placeholders(
        "{yesterday}",
        anchor=date.min,
    )
except DateTemplateError:
    overflow_rejected = True
else:
    overflow_rejected = False

try:
    render_fixed_date_placeholders(
        "{today}",
        anchor=datetime(2026, 7, 27, 12, 0),
    )
except TypeError:
    datetime_rejected = True
else:
    datetime_rejected = False

assert (
    rendered,
    anchor_only,
    rejected,
    missing_range_rejected,
    overflow_rejected,
    datetime_rejected,
) == (
    "Café report {daily}: 2026-07-26 | 2026-07-27 | 2026-07-28; "
    "range=2026-07-01..2026-07-31",
    "snapshot-2026-07-27",
    8,
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

The parser and renderer run in linear time over a bounded template and create
one bounded output string. Placeholder results are always ten-character ISO
calendar dates; there is no locale-sensitive formatting, arbitrary `strftime`
directive, time of day, time zone, duration expression, or clock lookup.
Adjacent-day arithmetic at `date.min` or `date.max` fails only if the overflowing
field is actually referenced.

The helper does not verify that `start` precedes `end`, interpret either as an
interval boundary, normalize filenames, parse or validate URLs, fetch network
content, or perform I/O. Its five-name grammar is closed: adding another field
requires a deliberate code and contract change, and every unknown or structured
field is rejected rather than delegated to Python's general formatting rules.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Bounded Flat Placeholder Template](parse-a-bounded-flat-placeholder-template.md)
- [Parse Compact Durations into timedelta](parse-compact-durations-into-timedelta.md)
- [Load Text Templates from Package Resources](../python-language/load-text-templates-from-package-resources.md)
<!-- catalog:related:end -->
