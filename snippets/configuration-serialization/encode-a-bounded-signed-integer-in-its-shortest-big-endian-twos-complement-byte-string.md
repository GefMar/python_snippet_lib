---
title: "Encode a Bounded Signed Integer in Its Shortest Big-Endian Two's-Complement Byte String"
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
  - encode-and-decode-signed-64-bit-integers-with-zigzag-mapping.md
  - ../networking-protocols/read-and-write-size-capped-varint-frames.md
---

# Encode a Bounded Signed Integer in Its Shortest Big-Endian Two's-Complement Byte String

## Idea and Problem

Encode an exact signed integer with the fewest non-empty big-endian two's-complement bytes that preserve its value and sign.

Positive values whose highest bit would look negative need a leading zero byte,
while some negative values need a leading `ff` byte to keep the next bit
negative. Counting the significant bits of the value, or of its complement for
a negative value, determines the required width directly without probing every
possible byte length.

## When to Use

Use this encoder when a bounded serialization profile explicitly requires the
canonical shortest signed big-endian representation used by
`int.from_bytes(..., signed=True)`. The caller-provided byte cap makes accepted
integer size and allocation cost visible at the boundary.

Use a fixed-width field when the surrounding schema specifies one. Use that
protocol's own varint or ZigZag rules when it defines a different variable-
length representation; these encodings are not interchangeable.

## Implementation

```python
_MAX_SIGNED_BYTES = 512


def encode_shortest_signed_integer(value: int, *, max_bytes: int) -> bytes:
    """Return the shortest signed big-endian bytes for one bounded integer."""
    if type(value) is not int:
        raise TypeError("value must be an exact non-boolean integer")
    if type(max_bytes) is not int:
        raise TypeError("max_bytes must be an exact non-boolean integer")
    if not 1 <= max_bytes <= _MAX_SIGNED_BYTES:
        raise ValueError("max_bytes is outside the supported range")

    if value >= 0:
        signed_bits = value.bit_length() + 1
    else:
        signed_bits = (~value).bit_length() + 1
    required_bytes = max(1, (signed_bits + 7) // 8)

    if required_bytes > max_bytes:
        raise ValueError("value does not fit within max_bytes signed bytes")
    return value.to_bytes(required_bytes, byteorder="big", signed=True)
```

## Example

```python
def direct_shortest_signed_bytes(value: int, max_bytes: int) -> bytes:
    for byte_count in range(1, max_bytes + 1):
        try:
            return value.to_bytes(byte_count, byteorder="big", signed=True)
        except OverflowError:
            continue
    raise ValueError("value does not fit")


def exercise_small_signed_boundaries() -> None:
    for byte_count in range(1, 5):
        lower = -(1 << (8 * byte_count - 1))
        upper = (1 << (8 * byte_count - 1)) - 1
        for value in (lower, lower + 1, -1, 0, 1, upper - 1, upper):
            encoded = encode_shortest_signed_integer(
                value,
                max_bytes=byte_count,
            )
            assert encoded == direct_shortest_signed_bytes(value, byte_count)
            assert int.from_bytes(encoded, byteorder="big", signed=True) == value
            if len(encoded) > 1:
                assert int.from_bytes(encoded[1:], byteorder="big", signed=True) != value
        for overflow in (lower - 1, upper + 1):
            try:
                encode_shortest_signed_integer(overflow, max_bytes=byte_count)
            except ValueError:
                pass
            else:
                raise AssertionError("out-of-range signed value was accepted")


exercise_small_signed_boundaries()

wide_value = -(1 << 4095)
wide_encoded = encode_shortest_signed_integer(wide_value, max_bytes=512)

try:
    encode_shortest_signed_integer(128, max_bytes=1)
except ValueError:
    one_byte_overflow_rejected = True
else:
    one_byte_overflow_rejected = False

assert (
    encode_shortest_signed_integer(0, max_bytes=1),
    encode_shortest_signed_integer(-1, max_bytes=1),
    encode_shortest_signed_integer(127, max_bytes=1),
    encode_shortest_signed_integer(128, max_bytes=2),
    encode_shortest_signed_integer(-128, max_bytes=1),
    encode_shortest_signed_integer(-129, max_bytes=2),
    len(wide_encoded),
    wide_encoded[0],
    int.from_bytes(wide_encoded, byteorder="big", signed=True),
    one_byte_overflow_rejected,
) == (
    b"\x00",
    b"\xff",
    b"\x7f",
    b"\x00\x80",
    b"\x80",
    b"\xff\x7f",
    512,
    0x80,
    wide_value,
    True,
)
```

## Trade-offs and Limitations

Bit-length calculation and encoding take `O(b)` time and the result uses
`O(b)` memory for `b` output bytes. The function computes the required width
before allocating the result, and the 512-byte cap bounds every successful
encoding. Python integers remain arbitrary precision outside this deliberately
bounded serialization boundary.

The output is canonical only for this shortest signed big-endian contract. It
does not preserve numeric sort order across variable-length byte strings and
does not include a length prefix, type tag, schema, checksum, framing, or byte-
order marker. Unsigned encodings, little-endian fields, fixed-width records,
ZigZag mappings, and protocol-specific varints require separate rules.

## Related Snippets

<!-- catalog:related:start -->
- [Encode and Decode One Exact-Size Big-Endian Binary Record with struct.Struct](encode-and-decode-one-exact-size-big-endian-binary-record-with-struct.md)
- [Encode and Decode Signed 64-Bit Integers with ZigZag Mapping](encode-and-decode-signed-64-bit-integers-with-zigzag-mapping.md)
- [Read and Write Size-Capped Varint Frames](../networking-protocols/read-and-write-size-capped-varint-frames.md)
<!-- catalog:related:end -->
