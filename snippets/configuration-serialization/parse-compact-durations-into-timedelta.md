---
title: "Parse Compact Durations into timedelta"
snippet_type: recipe
use_cases:
  - configuration
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - parse-explicit-decimal-and-binary-byte-sizes.md
---

# Parse Compact Durations into timedelta

## Idea and Problem

Parse one unsigned ASCII integer and one explicit lowercase unit into a timedelta under a deliberately small grammar.

Compact duration strings are convenient in configuration, but permissive
parsers create ambiguity around signs, decimals, compound values, and months.
This parser accepts only seconds, minutes, hours, days, or weeks and converts
overflow into the same clear input error as malformed syntax.

## When to Use

Use this recipe when a configuration contract intentionally supports forms
such as `30s`, `15m`, or `2h` and does not need compound or fractional
durations. Surrounding whitespace is accepted; internal whitespace and
uppercase units are not. Use a standardized duration format or a dedicated
library when interoperability requires a broader grammar.

## Implementation

```python
import re
from datetime import timedelta


_COMPACT_DURATION = re.compile(r"(?P<amount>[0-9]+)(?P<unit>[smhdw])")
_TIMEDELTA_ARGUMENT = {
    "s": "seconds",
    "m": "minutes",
    "h": "hours",
    "d": "days",
    "w": "weeks",
}


def parse_compact_duration(text: str) -> timedelta:
    if not isinstance(text, str):
        raise TypeError("duration must be text")

    match = _COMPACT_DURATION.fullmatch(text.strip())
    if match is None:
        raise ValueError("duration must be an unsigned integer followed by s, m, h, d, or w")

    try:
        amount = int(match.group("amount"))
        argument = _TIMEDELTA_ARGUMENT[match.group("unit")]
        return timedelta(**{argument: amount})
    except (OverflowError, ValueError) as error:
        raise ValueError("duration is outside the supported range") from error
```

## Example

```python
from datetime import timedelta


parsed = (
    parse_compact_duration("0s"),
    parse_compact_duration("90s"),
    parse_compact_duration("2h"),
    parse_compact_duration(" 3d "),
)

invalid_text = ("", "5", "-1s", "+1s", "1.5h", "1 h", "1H", "1m30s")
rejected = []
for text in invalid_text:
    try:
        parse_compact_duration(text)
    except ValueError:
        rejected.append(text)

try:
    parse_compact_duration(30)
except TypeError:
    non_text_rejected = True
else:
    non_text_rejected = False

try:
    parse_compact_duration("100000000000000000000w")
except ValueError:
    overflow_rejected = True
else:
    overflow_rejected = False

assert (parsed, tuple(rejected), non_text_rejected, overflow_rejected) == (
    (timedelta(0), timedelta(seconds=90), timedelta(hours=2), timedelta(days=3)),
    invalid_text,
    True,
    True,
)
```

## Trade-offs and Limitations

The grammar excludes negative, fractional, compound, and uppercase forms.
Weeks and days are fixed elapsed durations; calendar months and daylight-saving
transitions are different concepts. Very long untrusted text should be bounded
before parsing. The function returns a value only and does not preserve the
original unit for display or round-trip formatting.

## Related Snippets

<!-- catalog:related:start -->
- [Parse Explicit Decimal and Binary Byte Sizes](parse-explicit-decimal-and-binary-byte-sizes.md)
<!-- catalog:related:end -->
