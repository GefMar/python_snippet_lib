---
title: "Encode and Decode Signed 64-Bit Integers with ZigZag Mapping"
snippet_type: algorithm
use_cases:
  - interoperability
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - encode-and-decode-one-exact-size-big-endian-binary-record-with-struct.md
  - ../networking-protocols/read-and-write-size-capped-varint-frames.md
  - build-a-canonical-binary-prefix-codebook-from-declared-byte-code-lengths.md
---

# Encode and Decode Signed 64-Bit Integers with ZigZag Mapping

## Idea and Problem

Map every signed 64-bit integer bijectively to one unsigned 64-bit integer while placing small absolute values near zero.

ZigZag assigns even codes to non-negative values and odd codes to negative
values: `0, -1, 1, -2, 2` becomes `0, 1, 2, 3, 4`. The parity of an
unsigned code therefore carries the sign, and shifting the remaining bits
recovers the magnitude without a lookup table.

## When to Use

Use this mapping at a serialization boundary that accepts only unsigned
integers but must represent signed 64-bit values reversibly. It is especially
useful before a separate variable-length integer encoder because small values
of either sign then have small unsigned codes.

Keep the mapping separate from its byte representation. A protocol must still
specify its own integer framing, byte order or varint rules, and the decoder
must validate the unsigned domain before applying the inverse operation.

## Implementation

```python
_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_UINT64 = (1 << 64) - 1


def encode_zigzag_int64(value: int) -> int:
    """Map one exact signed 64-bit integer to an unsigned 64-bit code."""
    if type(value) is not int:
        raise TypeError("value must be an exact integer")
    if not _MIN_INT64 <= value <= _MAX_INT64:
        raise ValueError("value is outside the signed 64-bit range")
    return (value << 1) ^ (value >> 63)


def decode_zigzag_int64(code: int) -> int:
    """Invert one exact unsigned 64-bit ZigZag code."""
    if type(code) is not int:
        raise TypeError("code must be an exact integer")
    if not 0 <= code <= _MAX_UINT64:
        raise ValueError("code is outside the unsigned 64-bit range")
    return (code >> 1) ^ -(code & 1)
```

## Example

```python
known_pairs = (
    (0, 0),
    (-1, 1),
    (1, 2),
    (-2, 3),
    (_MIN_INT64, _MAX_UINT64),
    (_MAX_INT64, _MAX_UINT64 - 1),
)

assert tuple((value, encode_zigzag_int64(value)) for value, _ in known_pairs) == known_pairs
assert tuple(decode_zigzag_int64(code) for _, code in known_pairs) == tuple(
    value for value, _ in known_pairs
)
```

## Trade-offs and Limitations

Both operations use `O(1)` time and memory because every admitted value has a
fixed 64-bit domain. Python performs the shifts with exact integers, while the
explicit bounds make the result identical to the mathematical signed-to-
unsigned mapping without relying on machine overflow or masking.

ZigZag is a bijection, not compression by itself, and it does not preserve
ordinary signed numeric order. It deliberately rejects arbitrary-precision
inputs and Booleans. The functions do not encode bytes, choose endianness,
write varints, add framing, define a schema, or validate a complete wire
format.

## Related Snippets

<!-- catalog:related:start -->
- [Encode and Decode One Exact-Size Big-Endian Binary Record with struct.Struct](encode-and-decode-one-exact-size-big-endian-binary-record-with-struct.md)
- [Read and Write Size-Capped Varint Frames](../networking-protocols/read-and-write-size-capped-varint-frames.md)
- [Build a Canonical Binary Prefix Codebook from Declared Byte Code Lengths](build-a-canonical-binary-prefix-codebook-from-declared-byte-code-lengths.md)
<!-- catalog:related:end -->
