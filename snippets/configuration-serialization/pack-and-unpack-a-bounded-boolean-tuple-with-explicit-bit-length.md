---
title: "Pack and Unpack a Bounded Boolean Tuple with Explicit Bit Length"
snippet_type: algorithm
use_cases:
  - interoperability
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - build-a-canonical-binary-prefix-codebook-from-declared-byte-code-lengths.md
  - encode-and-decode-one-exact-size-big-endian-binary-record-with-struct.md
  - encode-a-bounded-signed-integer-in-its-shortest-big-endian-twos-complement-byte-string.md
---

# Pack and Unpack a Bounded Boolean Tuple with Explicit Bit Length

## Idea and Problem

Store a bounded Boolean tuple as canonical MSB-first bytes while carrying its exact logical bit length.

Eight consecutive values share one byte. A value at logical index `i` occupies
bit `7 - (i % 8)` in byte `i // 8`, so tuple order is visible without relying
on machine endianness. The explicit length distinguishes meaningful trailing
false values from zero padding, and rejecting nonzero unused bits gives every
Boolean tuple one byte representation.

## When to Use

Use this codec when an in-memory Boolean vector needs a compact deterministic
representation and both sides can carry the logical length beside the bytes.
It fits bounded fixtures, capability flags, or small binary records whose bit
order is controlled by this exact contract.

Use a format-specific codec when an external protocol already defines bit
numbering, length framing, or integrity checks. Use a specialized bit-array
type when values need frequent mutation, slicing, vectorized operations, or
compression rather than one immutable encode/decode boundary.

## Implementation

```python
_MAX_BOOLEAN_BITS = 1_000_000


def pack_boolean_tuple(values: tuple[bool, ...]) -> tuple[int, bytes]:
    """Return the logical length and canonical MSB-first packed bytes."""
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    if len(values) > _MAX_BOOLEAN_BITS:
        raise ValueError("value count exceeds the supported bit limit")
    for index, value in enumerate(values):
        if type(value) is not bool:
            raise TypeError(f"values[{index}] must be an exact boolean")

    bit_length = len(values)
    packed = bytearray((bit_length + 7) // 8)
    for index, value in enumerate(values):
        if value:
            packed[index // 8] |= 1 << (7 - index % 8)
    return bit_length, bytes(packed)


def unpack_boolean_tuple(bit_length: int, data: bytes) -> tuple[bool, ...]:
    """Return the exact Boolean tuple encoded by one canonical payload."""
    if type(bit_length) is not int:
        raise TypeError("bit_length must be an exact non-boolean integer")
    if type(data) is not bytes:
        raise TypeError("data must be exact bytes")
    if not 0 <= bit_length <= _MAX_BOOLEAN_BITS:
        raise ValueError("bit_length is outside the supported range")

    expected_bytes = (bit_length + 7) // 8
    if len(data) != expected_bytes:
        raise ValueError("data length does not match bit_length")

    used_bits = bit_length % 8
    if used_bits:
        unused_mask = (1 << (8 - used_bits)) - 1
        if data[-1] & unused_mask:
            raise ValueError("unused low bits in data must be zero")

    return tuple(bool(data[index // 8] & (1 << (7 - index % 8))) for index in range(bit_length))
```

## Example

```python
def reference_pack(values: tuple[bool, ...]) -> tuple[int, bytes]:
    logical_bits = "".join("1" if value else "0" for value in values)
    padded_bits = logical_bits + "0" * (-len(logical_bits) % 8)
    if not padded_bits:
        return 0, b""
    encoded = int(padded_bits, 2).to_bytes(len(padded_bits) // 8, "big")
    return len(values), encoded


def reference_unpack(bit_length: int, data: bytes) -> tuple[bool, ...]:
    padded_bits = "".join(f"{byte:08b}" for byte in data)
    return tuple(bit == "1" for bit in padded_bits[:bit_length])


def exercise_short_boolean_tuples() -> int:
    from itertools import product

    checked = 0
    for bit_length in range(13):
        for values in product((False, True), repeat=bit_length):
            packed = pack_boolean_tuple(values)
            assert packed == reference_pack(values)
            assert unpack_boolean_tuple(*packed) == values
            assert reference_unpack(*packed) == values
            checked += 1
    return checked


checked = exercise_short_boolean_tuples()
sample = (True, False, True, False, False, False, True, True, True)

try:
    unpack_boolean_tuple(9, b"\xa3\x81")
except ValueError:
    nonzero_padding_rejected = True
else:
    nonzero_padding_rejected = False

try:
    unpack_boolean_tuple(9, b"\xa3")
except ValueError:
    wrong_length_rejected = True
else:
    wrong_length_rejected = False

try:
    pack_boolean_tuple((True, 1))
except TypeError:
    wrong_type_rejected = True
else:
    wrong_type_rejected = False

assert (
    checked,
    pack_boolean_tuple(sample),
    unpack_boolean_tuple(9, b"\xa3\x80"),
    pack_boolean_tuple(()),
    unpack_boolean_tuple(0, b""),
    nonzero_padding_rejected,
    wrong_length_rejected,
    wrong_type_rejected,
) == (
    8_191,
    (9, b"\xa3\x80"),
    sample,
    (0, b""),
    (),
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

Packing takes `O(n)` time and uses a `ceil(n / 8)` bytearray before creating
the immutable bytes result, so both packed buffers briefly coexist. Unpacking
takes `O(n)` time and returns an `O(n)` tuple of Boolean references. The
one-million-bit limit bounds the packed payload at 125,000 bytes and bounds the
larger unpacked result explicitly.

The length is required even when every trailing logical value is false. For a
partial final byte, only its high bits carry values; all unused low bits must
be zero. This canonicality check rejects alternative payloads that would
otherwise decode to the same logical tuple.

The representation is not self-framing and does not include a schema,
compression, tri-state/null values, a checksum, authentication, or implicit
length recovery. It has one fixed MSB-first order and is not interchangeable
with an external format that assigns bits differently.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Canonical Binary Prefix Codebook from Declared Byte Code Lengths](build-a-canonical-binary-prefix-codebook-from-declared-byte-code-lengths.md)
- [Encode and Decode One Exact-Size Big-Endian Binary Record with struct.Struct](encode-and-decode-one-exact-size-big-endian-binary-record-with-struct.md)
- [Encode a Bounded Signed Integer in Its Shortest Big-Endian Two's-Complement Byte String](encode-a-bounded-signed-integer-in-its-shortest-big-endian-twos-complement-byte-string.md)
<!-- catalog:related:end -->
