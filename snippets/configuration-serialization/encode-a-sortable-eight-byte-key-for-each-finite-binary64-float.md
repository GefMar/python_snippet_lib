---
title: "Encode a Sortable Eight-Byte Key for Each Finite Binary64 Float"
snippet_type: recipe
use_cases:
  - interoperability
  - serialization
tested_python:
  - "3.14"
dependencies: []
related:
  - encode-a-bounded-signed-integer-in-its-shortest-big-endian-twos-complement-byte-string.md
  - ../data-processing/normalize-a-finite-binary64-value-into-a-canonical-half-open-period.md
---

# Encode a Sortable Eight-Byte Key for Each Finite Binary64 Float

## Idea and Problem

Encode finite binary64 floating-point values so bytewise lexicographic order matches numeric order, with negative zero before positive zero.

The exponent and fraction bits increase with magnitude. For a value with
the sign bit set, complement all bits to reverse magnitude order. For the
other values, set the sign bit so their keys follow the negative keys.
Writing the result in big-endian order makes the most significant byte
decide first. The two zero values therefore occupy adjacent distinct keys.

## When to Use

Use this for fixed-width sort keys on Python implementations with IEEE 754
binary64 `float` semantics. Compare keys lexicographically as unsigned bytes.
The function accepts finite floats, including subnormal values and both zero
signs. Non-finite values and other numeric types require their own policy.

## Implementation

```python
from math import isfinite
from struct import pack


_SIGN_BIT = 1 << 63
_ALL_BITS = (1 << 64) - 1


def sortable_float_key(value: float) -> bytes:
    """Encode a finite binary64 float as an eight-byte sortable key."""
    if type(value) is not float:
        raise TypeError("value must be a float")
    if not isfinite(value):
        raise ValueError("value must be finite")

    bits = int.from_bytes(pack(">d", value), "big")
    ordered = bits ^ _ALL_BITS if bits & _SIGN_BIT else bits ^ _SIGN_BIT
    return ordered.to_bytes(8, "big")
```

## Example

```python
values = (1.0, 0.0, -0.0, -1.0)
ordered = sorted(values, key=sortable_float_key)

assert tuple(sortable_float_key(value).hex() for value in ordered) == (
    "400fffffffffffff",  # -1.0
    "7fffffffffffffff",  # -0.0
    "8000000000000000",  # +0.0
    "bff0000000000000",  # +1.0
)
```

## Trade-offs and Limitations

Each conversion produces exactly eight bytes. The zero keys are distinct
even though Python treats the input zeros as equal. If the application uses
one key for both zeros, collapse zero to `0.0` before encoding.

This ordering applies to keys produced by the same transformation and to
bytewise comparison. The packed format `>d` defines the binary64 layout.
This recipe assumes the input already has that precision, so it does not
implement an ordering for decimal numbers or arbitrary-precision floats.

## Related Snippets

<!-- catalog:related:start -->
- [Encode a Bounded Signed Integer in Its Shortest Big-Endian Two's-Complement Byte String](encode-a-bounded-signed-integer-in-its-shortest-big-endian-twos-complement-byte-string.md)
- [Normalize a Finite Binary64 Value into a Canonical Half-Open Period](../data-processing/normalize-a-finite-binary64-value-into-a-canonical-half-open-period.md)
<!-- catalog:related:end -->
