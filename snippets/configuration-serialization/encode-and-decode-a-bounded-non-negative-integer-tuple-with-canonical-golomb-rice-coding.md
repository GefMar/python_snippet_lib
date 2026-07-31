---
title: "Encode and Decode a Bounded Non-Negative Integer Tuple with Canonical Golomb–Rice Coding"
snippet_type: algorithm
use_cases:
  - data-transformation
  - serialization
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - pack-and-restore-a-bounded-strictly-increasing-integer-sequence-with-elias-fano.md
  - encode-and-decode-signed-64-bit-integers-with-zigzag-mapping.md
  - ../networking-protocols/read-and-write-size-capped-varint-frames.md
---

# Encode and Decode a Bounded Non-Negative Integer Tuple with Canonical Golomb–Rice Coding

## Idea and Problem

Pack bounded non-negative integers with a fixed Rice parameter while defining one strict, reversible bit representation.

For parameter `k`, each value is split into quotient `value >> k` and its
`k` low remainder bits. The quotient is written as that many one bits followed
by a zero terminator; the remainder follows in exactly `k` bits. Small
quotients therefore use short codes without a per-value byte boundary.

This closed profile packs the resulting bit stream most-significant bit first,
records its meaningful length, and requires unused low bits in the last byte to
be zero. A strict decoder validates the complete shape and re-encodes the
result so alternate spellings cannot be accepted silently.

## When to Use

Use Rice coding when non-negative integers are expected to have small
quotients under one parameter that the caller can choose in advance. It is
useful for deterministic fixtures, compact residual sequences, and
instructional bit-level formats whose complete payload fits in memory.

The parameter is part of the contract, not estimated by this snippet. Use a
varint when independent byte-aligned values and streaming are more important,
or Elias–Fano when the input is strictly increasing and monotone-set queries or
that layout are the real objective. A general compressor is a better choice
when no plausible Rice parameter keeps quotients small.

## Implementation

```python
from dataclasses import dataclass

_MAX_RICE_COUNT = 65_536
_MAX_RICE_VALUE = (1 << 31) - 1
_MAX_RICE_PARAMETER = 31
_MAX_RICE_BITS = 1 << 20


@dataclass(frozen=True, slots=True)
class RiceBlock:
    count: int
    parameter: int
    bit_length: int
    payload: bytes


def _set_msb_bit(payload: bytearray, position: int) -> None:
    payload[position // 8] |= 1 << (7 - position % 8)


def _read_msb_bit(payload: bytes, position: int) -> int:
    return (payload[position // 8] >> (7 - position % 8)) & 1


def encode_rice(values: tuple[int, ...], parameter: int) -> RiceBlock:
    """Encode one exact integer tuple under a fixed canonical Rice profile."""
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    if len(values) > _MAX_RICE_COUNT:
        raise ValueError("values contains more than 65,536 items")
    if type(parameter) is not int:
        raise TypeError("parameter must be an exact integer")
    if not 0 <= parameter <= _MAX_RICE_PARAMETER:
        raise ValueError("parameter is outside 0..31")

    bit_length = 0
    for index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"values[{index}] must be an exact integer")
        if not 0 <= value <= _MAX_RICE_VALUE:
            raise ValueError(f"values[{index}] is outside 0..2^31-1")
        bit_length += (value >> parameter) + 1 + parameter
        if bit_length > _MAX_RICE_BITS:
            raise ValueError("encoded form exceeds 1,048,576 meaningful bits")

    payload = bytearray((bit_length + 7) // 8)
    position = 0
    remainder_mask = (1 << parameter) - 1
    for value in values:
        quotient = value >> parameter
        for _ in range(quotient):
            _set_msb_bit(payload, position)
            position += 1
        position += 1  # The zero unary terminator is already clear.

        remainder = value & remainder_mask
        for shift in range(parameter - 1, -1, -1):
            if remainder & (1 << shift):
                _set_msb_bit(payload, position)
            position += 1

    return RiceBlock(len(values), parameter, bit_length, bytes(payload))


def decode_rice(block: RiceBlock) -> tuple[int, ...]:
    """Validate and decode one canonical RiceBlock."""
    if type(block) is not RiceBlock:
        raise TypeError("block must be an exact RiceBlock")
    if type(block.count) is not int:
        raise TypeError("block.count must be an exact integer")
    if type(block.parameter) is not int:
        raise TypeError("block.parameter must be an exact integer")
    if type(block.bit_length) is not int:
        raise TypeError("block.bit_length must be an exact integer")
    if type(block.payload) is not bytes:
        raise TypeError("block.payload must be exact bytes")
    if not 0 <= block.count <= _MAX_RICE_COUNT:
        raise ValueError("block.count is outside 0..65,536")
    if not 0 <= block.parameter <= _MAX_RICE_PARAMETER:
        raise ValueError("block.parameter is outside 0..31")
    if not 0 <= block.bit_length <= _MAX_RICE_BITS:
        raise ValueError("block.bit_length is outside 0..1,048,576")
    if len(block.payload) != (block.bit_length + 7) // 8:
        raise ValueError("block.payload has the wrong byte length")
    used_bits = block.bit_length % 8
    if used_bits and block.payload[-1] & ((1 << (8 - used_bits)) - 1):
        raise ValueError("block.payload has non-zero padding bits")

    values: list[int] = []
    position = 0
    for _ in range(block.count):
        quotient = 0
        while position < block.bit_length and _read_msb_bit(block.payload, position):
            quotient += 1
            position += 1
        if position == block.bit_length:
            raise ValueError("block ends inside a unary quotient")
        position += 1

        if block.bit_length - position < block.parameter:
            raise ValueError("block ends inside a fixed-width remainder")
        remainder = 0
        for _ in range(block.parameter):
            remainder = (remainder << 1) | _read_msb_bit(block.payload, position)
            position += 1

        value = (quotient << block.parameter) | remainder
        if value > _MAX_RICE_VALUE:
            raise ValueError("block decodes a value above 2^31-1")
        values.append(value)

    if position != block.bit_length:
        raise ValueError("block contains trailing meaningful code bits")
    result = tuple(values)
    if encode_rice(result, block.parameter) != block:
        raise ValueError("block is not in canonical Rice form")
    return result
```

## Example

```python
from itertools import product


def reference_rice(values: tuple[int, ...], parameter: int) -> RiceBlock:
    pieces: list[str] = []
    for value in values:
        quotient = value >> parameter
        remainder = value & ((1 << parameter) - 1)
        pieces.append("1" * quotient + "0")
        if parameter:
            pieces.append(f"{remainder:0{parameter}b}")
    bits = "".join(pieces)
    padded = bits + "0" * (-len(bits) % 8)
    payload = bytes(int(padded[offset : offset + 8], 2) for offset in range(0, len(padded), 8))
    return RiceBlock(len(values), parameter, len(bits), payload)


checked = 0
for parameter in range(4):
    for size in range(5):
        for values in product(range(8), repeat=size):
            expected = reference_rice(values, parameter)
            assert encode_rice(values, parameter) == expected
            assert decode_rice(expected) == values
            checked += 1

known_one = encode_rice((0, 1, 2, 3), 1)
known_zero = encode_rice((0, 1, 2), 0)
maximum_count = (0,) * _MAX_RICE_COUNT
maximum_bit_block = encode_rice((_MAX_RICE_BITS - 1,), 0)
maximum_value = encode_rice((_MAX_RICE_VALUE,), 31)


def rejects_encode(values: object, parameter: object) -> bool:
    try:
        encode_rice(values, parameter)  # type: ignore[arg-type]
    except (TypeError, ValueError):
        return True
    return False


def rejects_decode(block: object) -> bool:
    try:
        decode_rice(block)  # type: ignore[arg-type]
    except (TypeError, ValueError):
        return True
    return False


malformed = (
    object(),
    RiceBlock(True, 0, 0, b""),
    RiceBlock(0, 0, 0, bytearray()),
    RiceBlock(1, 0, 1, b"\x80"),
    RiceBlock(1, 1, 1, b"\x00"),
    RiceBlock(4, 1, known_one.bit_length, known_one.payload[:-1]),
    RiceBlock(4, 1, known_one.bit_length, known_one.payload[:-1] + b"A"),
    RiceBlock(3, 1, known_one.bit_length, known_one.payload),
    RiceBlock(1, 31, 33, b"\x80\x00\x00\x00\x00"),
)
rejected_records = sum(rejects_decode(block) for block in malformed)

assert (
    checked,
    known_one,
    known_zero,
    decode_rice(encode_rice(maximum_count, 0)) == maximum_count,
    decode_rice(maximum_bit_block),
    decode_rice(maximum_value),
    rejects_encode((_MAX_RICE_BITS,), 0),
    rejects_encode((True,), 0),
    rejects_encode((0,), True),
    rejected_records,
) == (
    18_724,
    RiceBlock(4, 1, 10, b"\x19@"),
    RiceBlock(3, 0, 6, b"X"),
    True,
    (_MAX_RICE_BITS - 1,),
    (_MAX_RICE_VALUE,),
    True,
    True,
    True,
    len(malformed),
)
```

## Trade-offs and Limitations

For `N` values and `B` meaningful encoded bits, encoding and decoding perform
`O(N + B)` bit work. The payload occupies exactly `ceil(B / 8)` bytes, while
decoding additionally creates `O(N)` Python integers. Per-bit Python loops and
the record object add overhead that a native packed implementation would avoid.

Unary quotients make badly chosen parameters expensive; the one-megabit cap
rejects that expansion before allocation. The count and meaningful bit length
are required metadata and are not included in the payload-size comparison.
Nothing here chooses a good parameter or guarantees that the result is smaller
than a varint or the original representation.

This is one closed Rice-only profile with divisor `2**k`. It does not support
signed mapping, arbitrary Golomb divisors, streaming, framing, checksums,
adaptive models, corruption recovery, random access, or interoperability with
another format that happens to use Rice codes.

## Related Snippets

<!-- catalog:related:start -->
- [Pack and Restore a Bounded Strictly Increasing Integer Sequence with Elias–Fano](pack-and-restore-a-bounded-strictly-increasing-integer-sequence-with-elias-fano.md)
- [Encode and Decode Signed 64-Bit Integers with ZigZag Mapping](encode-and-decode-signed-64-bit-integers-with-zigzag-mapping.md)
- [Read and Write Size-Capped Varint Frames](../networking-protocols/read-and-write-size-capped-varint-frames.md)
<!-- catalog:related:end -->
