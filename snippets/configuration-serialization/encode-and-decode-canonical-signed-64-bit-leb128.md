---
title: "Encode and Decode Canonical Signed 64-Bit LEB128"
snippet_type: algorithm
use_cases:
  - interoperability
  - serialization
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - ../networking-protocols/read-and-write-size-capped-varint-frames.md
  - encode-and-decode-signed-64-bit-integers-with-zigzag-mapping.md
  - encode-a-bounded-signed-integer-in-its-shortest-big-endian-twos-complement-byte-string.md
---

# Encode and Decode Canonical Signed 64-Bit LEB128

## Idea and Problem

Represent one signed 64-bit integer as canonical signed LEB128 and reject every alternate or incomplete byte spelling.

Each byte contributes seven payload bits in little-endian group order. The high
bit marks continuation, while bit 6 of the final byte supplies the sign
extension. Encoding stops only when the remaining value and the final sign bit
agree, which produces the shortest representation.

## When to Use

Use this codec at a binary boundary whose specification explicitly calls for
signed LEB128 over the signed 64-bit domain. Whole-input decoding is useful for
standalone fields, fixtures, and validated records because it rejects trailing
bytes instead of silently leaving them for another consumer.

Do not substitute this format for unsigned LEB128 with ZigZag mapping: the two
formats assign different bytes to signed values. A stream parser should first
isolate one bounded field or expose an explicit consumed-byte result rather
than passing an arbitrary remainder to this whole-input decoder.

## Implementation

```python
_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_SLEB128_BYTES = 10


def encode_sleb128_int64(value: int) -> bytes:
    """Encode one exact signed 64-bit integer as shortest-form SLEB128."""
    if type(value) is not int:
        raise TypeError("value must be an exact integer")
    if not _MIN_INT64 <= value <= _MAX_INT64:
        raise ValueError("value is outside the signed 64-bit range")

    encoded = bytearray()
    remaining = value
    while True:
        payload = remaining & 0x7F
        remaining >>= 7
        sign_bit_set = bool(payload & 0x40)
        finished = (remaining == 0 and not sign_bit_set) or (remaining == -1 and sign_bit_set)
        encoded.append(payload | (0 if finished else 0x80))
        if finished:
            return bytes(encoded)


def decode_sleb128_int64(encoded: bytes) -> int:
    """Decode one complete canonical signed 64-bit SLEB128 byte string."""
    if type(encoded) is not bytes:
        raise TypeError("encoded must be exact bytes")
    if not 1 <= len(encoded) <= _MAX_SLEB128_BYTES:
        raise ValueError("encoded length must be in 1..10 bytes")

    value = 0
    shift = 0
    for index, byte in enumerate(encoded):
        value |= (byte & 0x7F) << shift
        shift += 7

        if byte & 0x80:
            continue
        if index != len(encoded) - 1:
            raise ValueError("encoded contains bytes after its terminator")
        if byte & 0x40:
            value -= 1 << shift
        if not _MIN_INT64 <= value <= _MAX_INT64:
            raise ValueError("encoded value is outside the signed 64-bit range")
        if encode_sleb128_int64(value) != encoded:
            raise ValueError("encoded is not canonical signed LEB128")
        return value

    raise ValueError("encoded has no terminating byte")
```

## Example

```python
def reference_sleb128(value: int) -> bytes:
    """Construct shortest SLEB128 by choosing its mathematical bit width."""
    for byte_count in range(1, _MAX_SLEB128_BYTES + 1):
        signed_bits = 7 * byte_count
        if -(1 << (signed_bits - 1)) <= value < 1 << (signed_bits - 1):
            residue = value % (1 << signed_bits)
            groups = [(residue >> (7 * index)) & 0x7F for index in range(byte_count)]
            return bytes(
                group | (0x80 if index + 1 < byte_count else 0)
                for index, group in enumerate(groups)
            )
    raise ValueError("value is outside the reference domain")


def positional_sleb128(encoded: bytes) -> int:
    """Interpret terminated payload groups as one signed positional sum."""
    unsigned_sum = sum((byte & 0x7F) << (7 * index) for index, byte in enumerate(encoded))
    bit_width = 7 * len(encoded)
    if encoded[-1] & 0x40:
        unsigned_sum -= 1 << bit_width
    return unsigned_sum


def rejects_decode(candidate: object) -> bool:
    try:
        decode_sleb128_int64(candidate)  # type: ignore[arg-type]
    except (TypeError, ValueError):
        return True
    return False


for value in range(-20_000, 20_001):
    encoded = encode_sleb128_int64(value)
    assert encoded == reference_sleb128(value)
    assert positional_sleb128(encoded) == value
    assert decode_sleb128_int64(encoded) == value

transition_values = {_MIN_INT64, _MAX_INT64}
for byte_count in range(1, _MAX_SLEB128_BYTES + 1):
    boundary = 1 << (7 * byte_count - 1)
    transition_values.update(
        value
        for value in (-boundary - 1, -boundary, -boundary + 1, boundary - 1, boundary, boundary + 1)
        if _MIN_INT64 <= value <= _MAX_INT64
    )

for value in transition_values:
    encoded = encode_sleb128_int64(value)
    assert encoded == reference_sleb128(value)
    assert decode_sleb128_int64(encoded) == value

assert encode_sleb128_int64(0) == b"\x00"
assert encode_sleb128_int64(-1) == b"\x7f"
assert encode_sleb128_int64(64) == b"\xc0\x00"
assert encode_sleb128_int64(-65) == b"\xbf\x7f"
assert len(encode_sleb128_int64(_MIN_INT64)) == 10
assert len(encode_sleb128_int64(_MAX_INT64)) == 10

malformed = (
    b"",
    b"\x80",
    b"\x80" * 10,
    b"\x00\x00",
    b"\x80\x00",
    b"\xff\x7f",
    b"\x80" * 9 + b"\x01",
    b"\xff" * 9 + b"\x7e",
    b"\x80" * 10 + b"\x00",
)
assert all(rejects_decode(candidate) for candidate in malformed)
assert rejects_decode(bytearray(b"\x00"))
assert rejects_decode(memoryview(b"\x00"))

for invalid in (True, 1.0, _MIN_INT64 - 1, _MAX_INT64 + 1):
    try:
        encode_sleb128_int64(invalid)  # type: ignore[arg-type]
    except (TypeError, ValueError):
        pass
    else:
        raise AssertionError("invalid encoder input was accepted")

assert decode_sleb128_int64(encode_sleb128_int64(_MIN_INT64)) == _MIN_INT64
```

## Trade-offs and Limitations

Encoding and decoding take `O(k)` time, where `k` is at most ten bytes. The
encoder returns `O(k)` output, and the decoder uses `O(1)` auxiliary memory.
The decoder bounds input length before accumulating bits, requires termination
at the final byte, checks the signed 64-bit range, and re-encodes the result to
reject redundant sign-extension groups.

This is a whole-value codec, not a stream reader. It does not provide framing,
checksums, schema identification, byte resynchronization, or partial-input
state. Runtime and branch behavior depend on the encoded value, so the
implementation is not intended for constant-time cryptographic processing.

## Related Snippets

<!-- catalog:related:start -->
- [Read and Write Size-Capped Varint Frames](../networking-protocols/read-and-write-size-capped-varint-frames.md)
- [Encode and Decode Signed 64-Bit Integers with ZigZag Mapping](encode-and-decode-signed-64-bit-integers-with-zigzag-mapping.md)
- [Encode a Bounded Signed Integer in Its Shortest Big-Endian Two's-Complement Byte String](encode-a-bounded-signed-integer-in-its-shortest-big-endian-twos-complement-byte-string.md)
<!-- catalog:related:end -->
