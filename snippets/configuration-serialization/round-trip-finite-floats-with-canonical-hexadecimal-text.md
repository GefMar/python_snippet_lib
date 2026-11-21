---
title: "Round Trip Finite Floats with Canonical Hexadecimal Text"
snippet_type: recipe
use_cases:
  - serialization
  - interoperability
tested_python:
  - "3.14"
dependencies: []
related:
  - render-a-finite-decimal-in-canonical-plain-notation.md
  - ../testing-tooling/generate-binary64-boundary-probes-with-math-nextafter.md
---

# Round Trip Finite Floats with Canonical Hexadecimal Text

## Idea and Problem

Represent a finite Python float as exact hexadecimal text for a record that must retain every stored bit of a numeric value.

Use the built-in encoder and decoder. After parsing, require the received token to match the encoder output, so equivalent spellings cannot acquire different text identities. This also keeps negative zero distinct from positive zero.

## When to Use

Use this when small text fixtures or diagnostic snapshots must preserve the binary float exactly and both ends agree on Python-style hexadecimal notation. In a decimal interchange format, use that format's numeric representation instead.

## Implementation

```python
from math import isfinite


def render_finite_float(value: float) -> str:
    if type(value) is not float:
        raise TypeError("expected a float")
    if not isfinite(value):
        raise ValueError("expected a finite float")
    return value.hex()


def parse_finite_float(token: str) -> float:
    if len(token) > 32:
        raise ValueError("hexadecimal float token is too long")
    try:
        value = float.fromhex(token)
    except OverflowError as error:
        raise ValueError("hexadecimal float is out of range") from error
    if render_finite_float(value) != token:
        raise ValueError("expected canonical hexadecimal float text")
    return value
```

## Example

```python
from math import ulp

values = (0.1, -0.0, 0.0, ulp(0.0))
tokens = tuple(map(render_finite_float, values))
rewritten = tuple(render_finite_float(parse_finite_float(token)) for token in tokens)
rejected = []
for text in ("0X1.0P0", "inf", "0x1p+100000"):
    try:
        parse_finite_float(text)
    except ValueError:
        rejected.append(text)

assert (rewritten, tokens[:3], rejected) == (
    tokens,
    ("0x1.999999999999ap-4", "-0x0.0p+0", "0x0.0p+0"),
    ["0X1.0P0", "inf", "0x1p+100000"],
)
```

## Trade-offs and Limitations

This preserves the stored binary value, not the decimal input that originally produced it. It deliberately excludes infinities and NaNs and uses a token bound sized for Python binary64. Hexadecimal floats are not JSON numbers. Use ordinary decimal text when human reading or an external format matters more than exact binary round trips.

## Related Snippets

<!-- catalog:related:start -->
- [Render a Finite Decimal in Canonical Plain Notation](render-a-finite-decimal-in-canonical-plain-notation.md)
- [Generate Binary64 Boundary Probes with math.nextafter](../testing-tooling/generate-binary64-boundary-probes-with-math-nextafter.md)
<!-- catalog:related:end -->
