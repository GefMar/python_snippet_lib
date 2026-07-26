---
title: "Resolve an Absolute or Percentage Limit"
snippet_type: recipe
use_cases:
  - parsing
  - validation
  - configuration
tested_python:
  - "3.14"
dependencies: []
related:
  - parse-explicit-decimal-and-binary-byte-sizes.md
  - convert-decimal-values-to-exact-minor-units.md
  - parse-compact-durations-into-timedelta.md
---

# Resolve an Absolute or Percentage Limit

## Idea and Problem

Resolve one strict configuration value as either an absolute count or a percentage of a known non-negative total.

Keeping the two syntaxes explicit avoids guessing whether a bare decimal is a
fraction or a count. Exact decimal ratios also make the documented floor
rounding independent of binary floating-point behavior.

## When to Use

Use this parser for bounded configuration or command-line thresholds where
operators may choose a stable count such as `25` or a scale-dependent value
such as `2.5%`. The caller must already know the relevant total and decide
whether an absolute count greater than that total is meaningful. Keep separate
parsers when fractional ratios, signed adjustments, or domain-specific units
are required.

## Implementation

```python
import re
from decimal import Decimal


_LIMIT = re.compile(
    r"(?:(?P<count>[0-9]+)|(?P<percent>[0-9]+(?:\.[0-9]+)?)%)"
)


def resolve_limit(text: str, *, total: int) -> int:
    if not isinstance(text, str):
        raise TypeError("limit must be text")
    if isinstance(total, bool) or not isinstance(total, int):
        raise TypeError("total must be an integer")
    if total < 0:
        raise ValueError("total must be non-negative")

    match = _LIMIT.fullmatch(text.strip())
    if match is None:
        raise ValueError("limit must be an unsigned count or decimal percentage")

    count = match.group("count")
    if count is not None:
        return int(count)

    percent_text = match.group("percent")
    assert percent_text is not None
    percentage = Decimal(percent_text)
    if percentage > 100:
        raise ValueError("percentage must not exceed 100")
    numerator, denominator = percentage.as_integer_ratio()
    return (total * numerator) // (100 * denominator)
```

## Example

```python
resolved = (
    resolve_limit("12", total=10),
    resolve_limit("2.5%", total=101),
    resolve_limit(" 100% ", total=7),
    resolve_limit("0.1%", total=5),
)

invalid = ("", "-1", "+1", "1e2", ".5%", "10.%", "100.1%", "1 %")
rejected = []
for value in invalid:
    try:
        resolve_limit(value, total=100)
    except ValueError:
        rejected.append(value)

try:
    resolve_limit("5", total=True)
except TypeError:
    boolean_total_rejected = True
else:
    boolean_total_rejected = False

assert (resolved, tuple(rejected), boolean_total_rejected) == (
    (12, 2, 7, 0),
    invalid,
    True,
)
```

## Trade-offs and Limitations

Percentage results always round down to a whole count, so a small positive
percentage may resolve to zero. Absolute counts are deliberately not clamped
to `total`; add that domain rule at the call site if required. The syntax
rejects signs, exponent notation, a leading decimal point, internal whitespace,
ratios, and percentages above 100. Very long untrusted text should be length
limited before parsing to avoid unnecessary large-integer work.

## Related Snippets

<!-- catalog:related:start -->
- [Parse Explicit Decimal and Binary Byte Sizes](parse-explicit-decimal-and-binary-byte-sizes.md)
- [Convert Decimal Values to Exact Minor Units](convert-decimal-values-to-exact-minor-units.md)
- [Parse Compact Durations into timedelta](parse-compact-durations-into-timedelta.md)
<!-- catalog:related:end -->
