---
title: "Convert Decimal Values to Exact Minor Units"
snippet_type: recipe
use_cases:
  - interoperability
  - validation
tested_python:
  - "3.14"
dependencies: []
related: []
---

# Convert Decimal Values to Exact Minor Units

## Idea and Problem

Convert a finite decimal value to an integer at an explicitly supplied scale without rounding or depending on ambient precision.

Many storage and interchange formats represent a decimal quantity as an
integer plus a known number of decimal places. Multiplication followed by
rounding can silently accept a value that is not exactly representable at that
scale. Using the value's exact integer ratio makes divisibility explicit.

## When to Use

Use this conversion at a boundary that requires exact scaled integers and
already has an authoritative scale, such as a schema field with two or three
decimal places. Keep scale lookup and domain rules outside the helper. Accept
a `Decimal` created from exact text or integer input rather than a binary
floating-point value.

## Implementation

```python
from decimal import Decimal


def to_exact_scaled_integer(
    value: Decimal,
    *,
    decimal_places: int,
) -> int:
    if not isinstance(value, Decimal):
        raise TypeError("value must be a Decimal")
    if isinstance(decimal_places, bool) or not isinstance(decimal_places, int):
        raise TypeError("decimal_places must be an integer")
    if decimal_places < 0:
        raise ValueError("decimal_places must not be negative")

    try:
        numerator, denominator = value.as_integer_ratio()
    except (OverflowError, ValueError) as error:
        raise ValueError("value must be finite") from error

    scaled_numerator = numerator * 10**decimal_places
    scaled_value, remainder = divmod(scaled_numerator, denominator)
    if remainder:
        raise ValueError("value has a fractional scaled unit")
    return scaled_value
```

## Example

```python
from decimal import Decimal


converted = (
    to_exact_scaled_integer(Decimal("12.34"), decimal_places=2),
    to_exact_scaled_integer(Decimal("-0.125"), decimal_places=3),
    to_exact_scaled_integer(Decimal("7"), decimal_places=0),
    to_exact_scaled_integer(Decimal("0"), decimal_places=2),
)

rejected_values = []
for invalid in (Decimal("1.005"), Decimal("NaN"), Decimal("Infinity")):
    try:
        to_exact_scaled_integer(invalid, decimal_places=2)
    except ValueError:
        rejected_values.append(invalid)

try:
    to_exact_scaled_integer(Decimal("1"), decimal_places=True)
except TypeError:
    bool_scale_rejected = True
else:
    bool_scale_rejected = False

try:
    to_exact_scaled_integer(0.1, decimal_places=1)
except TypeError:
    float_rejected = True
else:
    float_rejected = False

try:
    to_exact_scaled_integer(Decimal("1"), decimal_places=-1)
except ValueError:
    negative_scale_rejected = True
else:
    negative_scale_rejected = False

assert (
    converted,
    len(rejected_values),
    bool_scale_rejected,
    float_rejected,
    negative_scale_rejected,
) == ((1234, -125, 7, 0), 3, True, True, True)
```

## Trade-offs and Limitations

The function rejects rather than rounds fractional scaled units. A caller that
permits rounding must choose and document a rounding mode separately. Negative
values are accepted even when a particular domain forbids them. The helper
does not know currency or schema metadata. Untrusted values need schema limits
on their digit count and exponent because `as_integer_ratio` can construct very
large integers; an untrusted scale also needs an upper bound because
`10**decimal_places` has the same resource risk.

## Related Snippets

<!-- catalog:related:start -->
No related snippets.
<!-- catalog:related:end -->
