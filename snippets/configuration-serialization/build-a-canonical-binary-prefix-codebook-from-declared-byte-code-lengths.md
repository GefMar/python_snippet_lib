---
title: "Build a Canonical Binary Prefix Codebook from Declared Byte Code Lengths"
snippet_type: algorithm
use_cases:
  - data-transformation
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - decode-canonical-unpadded-base64url-under-encoded-and-decoded-byte-limits.md
  - encode-and-decode-one-exact-size-big-endian-binary-record-with-struct.md
  - register-and-unregister-a-bounded-single-byte-charmap-codec.md
---

# Build a Canonical Binary Prefix Codebook from Declared Byte Code Lengths

## Idea and Problem

Assign deterministic binary prefix codes when each byte symbol's code length is already known.

A canonical codebook sorts declarations by length and then symbol. Starting at
zero, it increments the previous code and left-shifts when the next length is
greater. This recurrence makes the codebook independent of declaration order
and detects an oversubscribed length set before any payload is processed.

## When to Use

Use this construction when a trusted format or a separate model has already
provided code lengths and both sides need the same compact lookup table. The
result stores each code as an integer plus its explicit bit length, which keeps
leading zeroes unambiguous without choosing a byte-packing convention.

Use a Huffman construction when frequencies must first be converted into
optimal lengths. Use a format-specific codec when headers, bit order, padding,
end markers, malformed payloads, or complete encode/decode behavior are part of
the interoperability contract.

## Implementation

```python
from dataclasses import dataclass

_MAX_BYTE_SYMBOLS = 256
_MAX_CODE_BITS = 64


@dataclass(frozen=True, slots=True)
class DeclaredByteCodeLength:
    symbol: int
    bit_length: int


@dataclass(frozen=True, slots=True)
class CanonicalBinaryCode:
    symbol: int
    bit_length: int
    code: int


def build_canonical_binary_prefix_codebook(
    declarations: tuple[DeclaredByteCodeLength, ...],
) -> tuple[CanonicalBinaryCode, ...]:
    """Return canonical codes ordered by declared length and byte symbol."""
    if type(declarations) is not tuple:
        raise TypeError("declarations must be an exact tuple")
    if not 1 <= len(declarations) <= _MAX_BYTE_SYMBOLS:
        raise ValueError("declaration count is outside the supported range")

    symbols: set[int] = set()
    for index, declaration in enumerate(declarations):
        if type(declaration) is not DeclaredByteCodeLength:
            raise TypeError(
                f"declarations[{index}] must be an exact DeclaredByteCodeLength"
            )
        if type(declaration.symbol) is not int:
            raise TypeError(f"declarations[{index}].symbol must be an exact integer")
        if not 0 <= declaration.symbol <= 255:
            raise ValueError(f"declarations[{index}].symbol is outside the byte range")
        if declaration.symbol in symbols:
            raise ValueError(f"declarations[{index}].symbol is duplicated")
        symbols.add(declaration.symbol)

        if type(declaration.bit_length) is not int:
            raise TypeError(f"declarations[{index}].bit_length must be an exact integer")
        if not 1 <= declaration.bit_length <= _MAX_CODE_BITS:
            raise ValueError(
                f"declarations[{index}].bit_length is outside the supported range"
            )

    ordered = sorted(declarations, key=lambda item: (item.bit_length, item.symbol))
    result: list[CanonicalBinaryCode] = []
    code = 0
    previous_length = ordered[0].bit_length

    for index, declaration in enumerate(ordered):
        if index:
            code = (code + 1) << (declaration.bit_length - previous_length)
            previous_length = declaration.bit_length
        if code >= 1 << declaration.bit_length:
            raise ValueError("declared code lengths are oversubscribed")
        result.append(
            CanonicalBinaryCode(
                symbol=declaration.symbol,
                bit_length=declaration.bit_length,
                code=code,
            )
        )

    return tuple(result)
```

## Example

```python
codebook = build_canonical_binary_prefix_codebook(
    (
        DeclaredByteCodeLength(68, 3),
        DeclaredByteCodeLength(65, 1),
        DeclaredByteCodeLength(67, 3),
        DeclaredByteCodeLength(66, 3),
    )
)

try:
    build_canonical_binary_prefix_codebook(
        (
            DeclaredByteCodeLength(1, 1),
            DeclaredByteCodeLength(2, 1),
            DeclaredByteCodeLength(3, 1),
        )
    )
except ValueError:
    oversubscribed_rejected = True
else:
    oversubscribed_rejected = False

assert (codebook, oversubscribed_rejected) == (
    (
        CanonicalBinaryCode(65, 1, 0b0),
        CanonicalBinaryCode(66, 3, 0b100),
        CanonicalBinaryCode(67, 3, 0b101),
        CanonicalBinaryCode(68, 3, 0b110),
    ),
    True,
)
```

## Trade-offs and Limitations

For `S` declared symbols, validation and canonical sorting take `O(S log S)`
time. The sorted declarations and frozen result use `O(S)` memory. Codes are at
most 64 bits, and the explicit `bit_length` distinguishes a code such as `001`
from the integer value alone.

Oversubscribed lengths are rejected because they cannot form a prefix code.
Incomplete trees are accepted: unused bit patterns do not make the declared
codes ambiguous. The result defines mathematical bit words, not their packing
into bytes, and does not derive lengths, optimize compression, encode or decode
payloads, parse headers, add end markers, or validate a complete wire format.

## Related Snippets

<!-- catalog:related:start -->
- [Decode Canonical Unpadded Base64url Under Encoded and Decoded Byte Limits](decode-canonical-unpadded-base64url-under-encoded-and-decoded-byte-limits.md)
- [Encode and Decode One Exact-Size Big-Endian Binary Record with struct.Struct](encode-and-decode-one-exact-size-big-endian-binary-record-with-struct.md)
- [Register and Unregister a Bounded Single-Byte Charmap Codec](register-and-unregister-a-bounded-single-byte-charmap-codec.md)
<!-- catalog:related:end -->
