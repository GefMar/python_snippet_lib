---
title: "Render a Finite Decimal in Canonical Plain Notation"
snippet_type: recipe
use_cases:
  - interoperability
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - convert-decimal-values-to-exact-minor-units.md
  - ../data-processing/build-quantized-decimal-views-with-deltas-at-explicit-scales.md
  - parse-explicit-decimal-and-binary-byte-sizes.md
---

# Render a Finite Decimal in Canonical Plain Notation

## Idea and Problem

Render a bounded finite decimal value as one context-independent plain ASCII representation without exponent notation or insignificant fractional zeroes.

Equivalent `Decimal` values can retain different coefficients and exponents,
such as `1`, `1.0`, and `1.00`. Constructing the output directly from the exact
decimal tuple preserves the number without rounding while making those
equivalent encodings converge on the same reader-facing text.

## When to Use

Use this renderer when an interchange boundary needs stable plain decimal text
and has already chosen to discard representational scale. It is useful for
canonical comparison keys, deterministic reports, and narrow formats that
forbid exponent notation but still require exact numeric round trips.

Use a scale-preserving formatter when trailing zeroes communicate precision or
significance. Define a format-specific number grammar separately: plain output
from this helper is not automatically a valid number in every JSON, database,
financial, or protocol schema.

## Implementation

```python
from decimal import Decimal

_MAX_COEFFICIENT_DIGITS = 1_024
_MIN_EXPONENT = -1_024
_MAX_EXPONENT = 1_024


def render_decimal_plain(value: Decimal) -> str:
    """Return one canonical plain representation of an admitted Decimal."""
    if type(value) is not Decimal:
        raise TypeError("value must be an exact Decimal")
    if not value.is_finite():
        raise ValueError("value must be finite")

    sign, digits, exponent = value.as_tuple()
    if not 1 <= len(digits) <= _MAX_COEFFICIENT_DIGITS:
        raise ValueError("coefficient digit count is outside the supported range")
    if not _MIN_EXPONENT <= exponent <= _MAX_EXPONENT:
        raise ValueError("exponent is outside the supported range")
    if not any(digits):
        return "0"

    digit_text = "".join(str(digit) for digit in digits)
    decimal_point = len(digit_text) + exponent

    if exponent >= 0:
        unsigned = digit_text + "0" * exponent
    elif decimal_point > 0:
        unsigned = digit_text[:decimal_point] + "." + digit_text[decimal_point:]
    else:
        unsigned = "0." + "0" * -decimal_point + digit_text

    if "." in unsigned:
        unsigned = unsigned.rstrip("0").rstrip(".")
    return ("-" if sign else "") + unsigned
```

## Example

```python
def canonical_plain_pattern():
    import re

    return re.compile(r"(?:0|-?(?:[1-9][0-9]*(?:\.[0-9]*[1-9])?|0\.[0-9]*[1-9]))")


def low_precision_context():
    from decimal import localcontext

    return localcontext()


canonical_plain = canonical_plain_pattern()


def assert_canonical_round_trip(value: Decimal) -> str:
    rendered = render_decimal_plain(value)
    assert canonical_plain.fullmatch(rendered)
    assert Decimal(rendered) == value
    return rendered


with low_precision_context() as context:
    context.prec = 2

    for coefficient in range(100):
        coefficient_digits = tuple(int(digit) for digit in str(coefficient))
        for sign in (0, 1):
            for exponent in range(-3, 4):
                assert_canonical_round_trip(Decimal((sign, coefficient_digits, exponent)))

    equivalent_groups = (
        (Decimal("1"), Decimal("1.0"), Decimal("1.00")),
        (Decimal("1000"), Decimal("1E+3"), Decimal("1000.000")),
        (Decimal("-0.125"), Decimal("-0.1250"), Decimal("-125E-3")),
    )
    equivalent_outputs = tuple(
        tuple({assert_canonical_round_trip(value) for value in group})
        for group in equivalent_groups
    )

    maximum_value = Decimal((1, (9,) * 1_024, 1_024))
    maximum_output = assert_canonical_round_trip(maximum_value)
    boundary_outputs = tuple(
        assert_canonical_round_trip(value)
        for value in (
            Decimal((0, (1,), -1_024)),
            Decimal((0, (9,) * 1_024, 1_024)),
            Decimal((1, (0,), 1_024)),
        )
    )

    rejected = 0
    for invalid in (
        Decimal((0, (1,) * 1_025, 0)),
        Decimal((0, (1,), -1_025)),
        Decimal((0, (1,), 1_025)),
        Decimal("NaN"),
        Decimal("Infinity"),
    ):
        try:
            render_decimal_plain(invalid)
        except ValueError:
            rejected += 1

assert (
    equivalent_outputs,
    len(maximum_output),
    boundary_outputs[-1],
    rejected,
) == ((("1",), ("1000",), ("-0.125",)), 2_049, "0", 5)
```

## Trade-offs and Limitations

For `d` coefficient digits and exponent `e`, validation and rendering take
`O(d + |e|)` time and output memory. The admitted bounds limit the result to
2,049 ASCII characters, including a possible minus sign. Tuple inspection and
string construction do not consult the active decimal precision or rounding
mode.

Signed zero is intentionally normalized to `"0"`. Fractional trailing zeroes
and a decimal point left empty by their removal are discarded, so the output
does not preserve input scale or stated significance. Integer zeroes remain
part of the value, and no numeric rounding is performed.

The function rejects non-finite values, floats, coefficients longer than 1,024
digits, and exponents outside `-1024..1024`. It does not localize separators,
choose a schema scale, preserve signed zero, produce scientific notation, or
claim compatibility with every JSON-number or protocol grammar.

## Related Snippets

<!-- catalog:related:start -->
- [Convert Decimal Values to Exact Minor Units](convert-decimal-values-to-exact-minor-units.md)
- [Build Quantized Decimal Views with Deltas at Explicit Scales](../data-processing/build-quantized-decimal-views-with-deltas-at-explicit-scales.md)
- [Parse Explicit Decimal and Binary Byte Sizes](parse-explicit-decimal-and-binary-byte-sizes.md)
<!-- catalog:related:end -->
