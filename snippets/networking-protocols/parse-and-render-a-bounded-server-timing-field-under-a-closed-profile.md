---
title: "Parse and Render a Bounded Server-Timing Field Under a Closed Profile"
snippet_type: recipe
use_cases:
  - interoperability
  - networking
  - parsing
  - serialization
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - parse-a-bounded-response-cache-control-field-under-a-closed-profile.md
  - parse-a-bounded-ascii-media-type-value.md
  - update-one-bounded-tracestate-member-while-preserving-peer-order.md
---

# Parse and Render a Bounded Server-Timing Field Under a Closed Profile

## Idea and Problem

Represent one bounded Server-Timing field as ordered immutable metrics and require one canonical spelling in both directions.

This closed sender profile preserves metric-name case, input order, and
duplicate names. Durations use decimal milliseconds instead of binary floating
point, and descriptions always use one quoted escaping form. Parsing succeeds
only when rendering the parsed value reproduces the exact input field.

## When to Use

Use this recipe when a controlled producer and consumer agree on this exact
canonical subset and need reproducible fixtures, signatures, comparisons, or
logs. Optional `dur` and `desc` parameters remain typed instead of being
collapsed into a mapping.

Use a maintained HTTP and Server Timing implementation at a browser-facing or
extension-tolerant boundary. The web specification permits behavior outside
this deliberately closed grammar, including processing that should not fail
because an unknown parameter appears.

## Implementation

```python
import re
from dataclasses import dataclass
from decimal import Decimal

_MAX_FIELD_LENGTH = 2048
_MAX_METRICS = 16
_MAX_NAME_LENGTH = 64
_MAX_DESCRIPTION_LENGTH = 128
_MAX_COEFFICIENT_DIGITS = 32
_MAX_ABSOLUTE_EXPONENT = 32
_MAX_DURATION_MS = Decimal(3_600_000)
_TOKEN_CHARACTER = frozenset(
    "!#$%&'*+-.^_`|~0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"
)
_DURATION_TEXT = re.compile(r"[0-9]+(?:\.[0-9]{1,3})?", re.ASCII).fullmatch


class ServerTimingProfileError(ValueError):
    """The value is outside this canonical Server-Timing profile."""


@dataclass(frozen=True, slots=True)
class ServerTimingMetric:
    name: str
    duration_ms: Decimal | None = None
    description: str | None = None


def _canonical_duration(duration: Decimal) -> str:
    if type(duration) is not Decimal:
        raise TypeError("duration_ms must be an exact Decimal or None")
    if not duration.is_finite() or duration.is_signed():
        raise ServerTimingProfileError("duration must be finite and nonnegative")

    sign, raw_digits, raw_exponent = duration.as_tuple()
    if sign != 0 or type(raw_exponent) is not int:
        raise ServerTimingProfileError("duration has an unsupported representation")
    if len(raw_digits) > _MAX_COEFFICIENT_DIGITS:
        raise ServerTimingProfileError("duration coefficient is too large")
    if not -_MAX_ABSOLUTE_EXPONENT <= raw_exponent <= _MAX_ABSOLUTE_EXPONENT:
        raise ServerTimingProfileError("duration exponent is too large")
    if duration > _MAX_DURATION_MS:
        raise ServerTimingProfileError("duration exceeds 3,600,000 milliseconds")
    if duration.is_zero():
        return "0"

    digits = list(raw_digits)
    exponent = raw_exponent
    while digits[-1] == 0:
        digits.pop()
        exponent += 1
    if exponent < -3:
        raise ServerTimingProfileError("duration has more than three decimal places")

    coefficient = "".join(str(digit) for digit in digits)
    decimal_position = len(coefficient) + exponent
    if decimal_position <= 0:
        return "0." + "0" * -decimal_position + coefficient
    if decimal_position < len(coefficient):
        return coefficient[:decimal_position] + "." + coefficient[decimal_position:]
    return coefficient + "0" * (decimal_position - len(coefficient))


def _validate_metric(metric: ServerTimingMetric) -> None:
    if type(metric) is not ServerTimingMetric:
        raise TypeError("every metric must be an exact ServerTimingMetric")
    if type(metric.name) is not str:
        raise TypeError("metric.name must be an exact string")
    if not 1 <= len(metric.name) <= _MAX_NAME_LENGTH:
        raise ServerTimingProfileError("metric name length is outside the profile")
    if any(character not in _TOKEN_CHARACTER for character in metric.name):
        raise ServerTimingProfileError("metric name is not an ASCII token")
    if metric.duration_ms is not None:
        _canonical_duration(metric.duration_ms)
    if metric.description is not None:
        if type(metric.description) is not str:
            raise TypeError("metric.description must be an exact string or None")
        if len(metric.description) > _MAX_DESCRIPTION_LENGTH:
            raise ServerTimingProfileError("metric description is too long")
        if any(not 0x20 <= ord(character) <= 0x7E for character in metric.description):
            raise ServerTimingProfileError(
                "metric description must contain printable ASCII only",
            )


def render_server_timing(
    metrics: tuple[ServerTimingMetric, ...],
) -> str:
    if type(metrics) is not tuple:
        raise TypeError("metrics must be an exact tuple")
    if not 1 <= len(metrics) <= _MAX_METRICS:
        raise ServerTimingProfileError("metric count is outside the profile")

    rendered_metrics: list[str] = []
    for metric in metrics:
        _validate_metric(metric)
        rendered = metric.name
        if metric.duration_ms is not None:
            rendered += ";dur=" + _canonical_duration(metric.duration_ms)
        if metric.description is not None:
            escaped = metric.description.replace("\\", "\\\\").replace('"', '\\"')
            rendered += f';desc="{escaped}"'
        rendered_metrics.append(rendered)

    field_value = ", ".join(rendered_metrics)
    if len(field_value) > _MAX_FIELD_LENGTH:
        raise ServerTimingProfileError("rendered field is too long")
    return field_value


def _parse_description(field_value: str, position: int) -> tuple[str, int]:
    if position >= len(field_value) or field_value[position] != '"':
        raise ServerTimingProfileError("description must start with a quote")
    position += 1
    characters: list[str] = []
    while position < len(field_value):
        character = field_value[position]
        if character == '"':
            return "".join(characters), position + 1
        if character == "\\":
            position += 1
            if position >= len(field_value) or field_value[position] not in ('"', "\\"):
                raise ServerTimingProfileError("description escape is outside the profile")
            character = field_value[position]
        if not 0x20 <= ord(character) <= 0x7E:
            raise ServerTimingProfileError("description must contain printable ASCII only")
        characters.append(character)
        if len(characters) > _MAX_DESCRIPTION_LENGTH:
            raise ServerTimingProfileError("metric description is too long")
        position += 1
    raise ServerTimingProfileError("description is missing its closing quote")


def parse_server_timing(
    field_value: str,
) -> tuple[ServerTimingMetric, ...]:
    if type(field_value) is not str:
        raise TypeError("field_value must be an exact string")
    if not 1 <= len(field_value) <= _MAX_FIELD_LENGTH:
        raise ServerTimingProfileError("field length is outside the profile")
    if any(ord(character) > 0x7F for character in field_value):
        raise ServerTimingProfileError("field must contain ASCII only")

    metrics: list[ServerTimingMetric] = []
    position = 0
    while True:
        name_start = position
        while position < len(field_value) and field_value[position] in _TOKEN_CHARACTER:
            position += 1
        name = field_value[name_start:position]
        if not 1 <= len(name) <= _MAX_NAME_LENGTH:
            raise ServerTimingProfileError("metric name is missing or too long")

        duration: Decimal | None = None
        description: str | None = None
        if field_value.startswith(";dur=", position):
            position += len(";dur=")
            duration_start = position
            while position < len(field_value) and (
                field_value[position].isdigit() or field_value[position] == "."
            ):
                position += 1
            duration_text = field_value[duration_start:position]
            if _DURATION_TEXT(duration_text) is None:
                raise ServerTimingProfileError("duration spelling is outside the profile")
            duration = Decimal(duration_text)

        if field_value.startswith(";desc=", position):
            position += len(";desc=")
            description, position = _parse_description(field_value, position)

        metrics.append(ServerTimingMetric(name, duration, description))
        if len(metrics) > _MAX_METRICS:
            raise ServerTimingProfileError("metric count is outside the profile")
        if position == len(field_value):
            break
        if not field_value.startswith(", ", position):
            raise ServerTimingProfileError("metric separator is not canonical")
        position += 2
        if position == len(field_value):
            raise ServerTimingProfileError("field ends with an empty metric")

    result = tuple(metrics)
    if render_server_timing(result) != field_value:
        raise ServerTimingProfileError("field is not in canonical form")
    return result
```

## Example

```python


metrics = (
    ServerTimingMetric("db", Decimal("12.5"), 'primary, "warm" \\ lane'),
    ServerTimingMetric("DB", description=""),
    ServerTimingMetric("db", Decimal(0)),
)
field_value = render_server_timing(metrics)
round_trip = parse_server_timing(field_value)

invalid_values = (
    "db;dur=1.0",
    'db;desc="read";dur=1',
    'db;desc="bad\\n-escape"',
    "db,disk;dur=2",
    "db;other=1",
)
rejected = []
for invalid_value in invalid_values:
    try:
        parse_server_timing(invalid_value)
    except ServerTimingProfileError:
        rejected.append(invalid_value)

assert (
    field_value,
    round_trip,
    tuple(rejected),
) == (
    'db;dur=12.5;desc="primary, \\"warm\\" \\\\ lane", DB;desc="", db;dur=0',
    metrics,
    invalid_values,
)
```

## Trade-offs and Limitations

Parsing and rendering are linear in the bounded field length. Before building
a fixed-point duration string, the renderer caps both the `Decimal`
coefficient and exponent; this avoids expanding a compact adversarial decimal
into an enormous string. Signed decimals, infinities, NaNs, values above one
hour, and values with more than three effective fractional places are
rejected.

This is a canonical sender profile, not a tolerant W3C Server Timing parser or
a browser behavior model. It rejects optional whitespace, alternate decimal
spellings, unquoted descriptions, reordered or unknown parameters, and every
escape except `\\` and `\"`. It deliberately preserves repeated metric names
rather than treating them as mapping keys.

Server-Timing values may expose application topology, operation names, timing
details, and correlations. This helper neither decides which metrics may
cross a privacy boundary nor implements response-header access policy, HTTP
field combination, user-agent exposure, measurement, or clock handling.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Bounded Response Cache-Control Field Under a Closed Profile](parse-a-bounded-response-cache-control-field-under-a-closed-profile.md)
- [Parse a Bounded ASCII Media Type Value](parse-a-bounded-ascii-media-type-value.md)
- [Update One Bounded tracestate Member While Preserving Peer Order](update-one-bounded-tracestate-member-while-preserving-peer-order.md)
<!-- catalog:related:end -->
