---
title: "Parse Explicit Decimal and Binary Byte Sizes"
snippet_type: recipe
use_cases:
  - configuration
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - convert-decimal-values-to-exact-minor-units.md
  - parse-compact-durations-into-timedelta.md
---

# Parse Explicit Decimal and Binary Byte Sizes

## Idea and Problem

Parse case-sensitive decimal and binary unit suffixes into an exact non-negative integer number of bytes.

Decimal units such as `kB` and binary units such as `KiB` use different
multipliers. Parsing through `Decimal` avoids binary floating-point surprises,
and an exact divisibility check rejects quantities that would require a
fraction of a byte.

## When to Use

Use this parser for configuration that deliberately exposes the supported
suffixes `B`, `kB`, `MB`, `GB`, `TB`, `KiB`, `MiB`, `GiB`, and `TiB`.
Surrounding whitespace is allowed, but signs, exponent notation, implicit
units, and case variants are not. Keep a separate maximum-size policy at the
boundary that consumes the result.

## Implementation

```python
import re
from decimal import Decimal


_BYTE_SIZE = re.compile(
    r"(?P<number>[0-9]+(?:\.[0-9]+)?)(?P<unit>KiB|MiB|GiB|TiB|B|kB|MB|GB|TB)"
)
_BYTE_MULTIPLIER = {
    "B": 1,
    "kB": 10**3,
    "MB": 10**6,
    "GB": 10**9,
    "TB": 10**12,
    "KiB": 2**10,
    "MiB": 2**20,
    "GiB": 2**30,
    "TiB": 2**40,
}


def parse_byte_size(text: str) -> int:
    if not isinstance(text, str):
        raise TypeError("byte size must be text")

    match = _BYTE_SIZE.fullmatch(text.strip())
    if match is None:
        raise ValueError("byte size must contain a supported explicit unit")

    quantity = Decimal(match.group("number"))
    numerator, denominator = quantity.as_integer_ratio()
    scaled_numerator = numerator * _BYTE_MULTIPLIER[match.group("unit")]
    byte_count, remainder = divmod(scaled_numerator, denominator)
    if remainder:
        raise ValueError("byte size does not resolve to a whole byte")
    return byte_count
```

## Example

```python
parsed = (
    parse_byte_size("0B"),
    parse_byte_size("1kB"),
    parse_byte_size("1KiB"),
    parse_byte_size("1.5KiB"),
    parse_byte_size(" 2MB "),
)

invalid_text = ("1", "1KB", "1mb", "-1B", "+1B", "1e3B", "1 B", "0.1KiB")
rejected = []
for text in invalid_text:
    try:
        parse_byte_size(text)
    except ValueError:
        rejected.append(text)

try:
    parse_byte_size(1024)
except TypeError:
    non_text_rejected = True
else:
    non_text_rejected = False

assert (parsed, tuple(rejected), non_text_rejected) == (
    (0, 1000, 1024, 1536, 2_000_000),
    invalid_text,
    True,
)
```

## Trade-offs and Limitations

The accepted spelling is intentionally narrow and excludes bits, signs,
scientific notation, implicit bytes, and alternative capitalization.
Untrusted text still needs a length limit because very long decimal input can
construct large integers. The parser enforces integrality but not a maximum
allocation or transport size. Formatting is separate because unit selection
and rounding are presentation policies.

## Related Snippets

<!-- catalog:related:start -->
- [Convert Decimal Values to Exact Minor Units](convert-decimal-values-to-exact-minor-units.md)
- [Parse Compact Durations into timedelta](parse-compact-durations-into-timedelta.md)
<!-- catalog:related:end -->
