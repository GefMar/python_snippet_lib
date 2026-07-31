---
title: "Encode and Decode Bounded Bytes with a Fixed Extended Hamming(8,4) SECDED Profile"
snippet_type: algorithm
use_cases:
  - data-transformation
  - retry-recovery
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - recover-one-missing-equal-length-byte-shard-with-xor-parity.md
  - ../configuration-serialization/append-a-fixed-width-crc-check-to-a-human-readable-identifier.md
  - ../configuration-serialization/pack-and-unpack-a-bounded-boolean-tuple-with-explicit-bit-length.md
---

# Encode and Decode Bounded Bytes with a Fixed Extended Hamming(8,4) SECDED Profile

## Idea and Problem

Protect every four-bit nibble with one byte-sized codeword that can correct one unknown flipped bit or detect exactly two flipped bits.

The fixed extended Hamming `(8, 4)` layout places Hamming parity in bit
positions 1, 2, and 4, data in positions 3, 5, 6, and 7, and overall even
parity in position 8. Each source byte becomes two codeword bytes, with the
high nibble first. This makes the bit ordering and byte framing complete parts
of the profile rather than implicit conventions.

The decoder records each corrected codeword and bit position. A detected
double-bit error fails the whole call, so no partially decoded payload escapes.

## When to Use

Use this small codec to demonstrate or verify single-error correction and
double-error detection over independently protected nibbles. It can also serve
as a deterministic fixture codec when doubling the payload size is acceptable
and the channel model is explicitly limited to at most two flipped bits per
codeword.

Use a standardized storage or transport format when interoperability matters.
Use a stronger error-correcting code for burst errors, erasures, multiple
corruptions, or better redundancy. Use authentication, not error correction,
when changes may be malicious.

## Implementation

```python
from dataclasses import dataclass
from itertools import combinations

_MAX_SECDED_SOURCE_BYTES = 65_536
_HAMMING_DATA_POSITIONS = (3, 5, 6, 7)
_HAMMING_PARITY_POSITIONS = (1, 2, 4)


@dataclass(frozen=True, slots=True)
class SecdedCorrection:
    codeword_index: int
    bit_position: int


@dataclass(frozen=True, slots=True)
class DecodedSecdedBytes:
    data: bytes
    corrections: tuple[SecdedCorrection, ...]


class DetectedDoubleBitError(ValueError):
    def __init__(self, codeword_index: int) -> None:
        self.codeword_index = codeword_index
        super().__init__(f"codeword {codeword_index} has a detected double-bit error")


def _hamming84_syndrome_and_overall(codeword: int) -> tuple[int, int]:
    syndrome = 0
    for parity_position in _HAMMING_PARITY_POSITIONS:
        group_parity = 0
        for position in range(1, 8):
            if position & parity_position:
                group_parity ^= codeword >> (position - 1) & 1
        if group_parity:
            syndrome |= parity_position
    return syndrome, codeword.bit_count() & 1


def _encode_hamming84_nibble(nibble: int) -> int:
    codeword = 0
    for bit_index, position in enumerate(_HAMMING_DATA_POSITIONS):
        codeword |= (nibble >> bit_index & 1) << (position - 1)
    for parity_position in _HAMMING_PARITY_POSITIONS:
        group_parity = 0
        for position in range(1, 8):
            if position & parity_position:
                group_parity ^= codeword >> (position - 1) & 1
        codeword |= group_parity << (parity_position - 1)
    codeword |= (codeword.bit_count() & 1) << 7
    return codeword


def _decode_hamming84_codeword(
    codeword: int,
    codeword_index: int,
) -> tuple[int, SecdedCorrection | None]:
    syndrome, overall_odd = _hamming84_syndrome_and_overall(codeword)
    correction: SecdedCorrection | None = None
    if syndrome == 0 and overall_odd == 0:
        corrected = codeword
    elif overall_odd == 1:
        bit_position = syndrome if syndrome else 8
        corrected = codeword ^ (1 << (bit_position - 1))
        correction = SecdedCorrection(codeword_index, bit_position)
    else:
        raise DetectedDoubleBitError(codeword_index)

    if _hamming84_syndrome_and_overall(corrected) != (0, 0):
        raise AssertionError("SECDED correction did not restore valid parity")
    nibble = sum(
        (corrected >> (position - 1) & 1) << bit_index
        for bit_index, position in enumerate(_HAMMING_DATA_POSITIONS)
    )
    return nibble, correction


def encode_hamming84_secded(data: bytes) -> bytes:
    """Encode each source byte as high- then low-nibble SECDED codewords."""
    if type(data) is not bytes:
        raise TypeError("data must be exact bytes")
    if len(data) > _MAX_SECDED_SOURCE_BYTES:
        raise ValueError("data exceeds the 65,536-byte source limit")

    encoded = bytearray()
    for value in data:
        encoded.append(_encode_hamming84_nibble(value >> 4))
        encoded.append(_encode_hamming84_nibble(value & 0x0F))
    return bytes(encoded)


def decode_hamming84_secded(encoded: bytes) -> DecodedSecdedBytes:
    """Decode one complete profile value or fail on a detected double error."""
    if type(encoded) is not bytes:
        raise TypeError("encoded must be exact bytes")
    if len(encoded) > 2 * _MAX_SECDED_SOURCE_BYTES:
        raise ValueError("encoded data exceeds the 131,072-byte limit")
    if len(encoded) % 2:
        raise ValueError("encoded data must contain complete codeword pairs")

    nibbles: list[int] = []
    corrections: list[SecdedCorrection] = []
    for codeword_index, codeword in enumerate(encoded):
        nibble, correction = _decode_hamming84_codeword(codeword, codeword_index)
        nibbles.append(nibble)
        if correction is not None:
            corrections.append(correction)

    data = bytes(
        (nibbles[index] << 4) | nibbles[index + 1]
        for index in range(0, len(nibbles), 2)
    )
    return DecodedSecdedBytes(data, tuple(corrections))
```

## Example

```python
checked = 0
for nibble in range(16):
    clean = _encode_hamming84_nibble(nibble)
    assert _decode_hamming84_codeword(clean, 0) == (nibble, None)
    checked += 1

    for bit_index in range(8):
        decoded, correction = _decode_hamming84_codeword(
            clean ^ (1 << bit_index),
            0,
        )
        assert decoded == nibble
        assert correction == SecdedCorrection(0, bit_index + 1)
        checked += 1

    for first_bit, second_bit in combinations(range(8), 2):
        corrupted = clean ^ (1 << first_bit) ^ (1 << second_bit)
        try:
            _decode_hamming84_codeword(corrupted, 0)
        except DetectedDoubleBitError as error:
            assert error.codeword_index == 0
        else:
            raise AssertionError("a double-bit corruption was not detected")
        checked += 1

all_bytes = bytes(range(256))
assert decode_hamming84_secded(encode_hamming84_secded(all_bytes)).data == all_bytes

payload = b"\xab\x10"
corrupted = bytearray(encode_hamming84_secded(payload))
corrupted[1] ^= 1 << 4
corrected = decode_hamming84_secded(bytes(corrupted))
assert corrected == DecodedSecdedBytes(payload, (SecdedCorrection(1, 5),))

maximum = bytes(range(256)) * 256
maximum_encoded = encode_hamming84_secded(maximum)
assert (
    checked == 592
    and encode_hamming84_secded(b"") == b""
    and decode_hamming84_secded(b"") == DecodedSecdedBytes(b"", ())
    and len(maximum_encoded) == 131_072
    and decode_hamming84_secded(maximum_encoded).data == maximum
)
```

## Trade-offs and Limitations

Encoding and decoding take `O(N)` time. The encoded value and decoder state use
`O(N)` memory, and the fixed byte-aligned profile always expands one source
byte into two bytes. Smaller bit-packed representations would need an
additional framing, padding, and logical-length contract.

The SECDED guarantee assumes each codeword differs from a valid codeword in at
most two bit positions. One flip is corrected and exactly two are detected;
three or more flips may imitate a correctable pattern and be miscorrected.
There is no burst-error interleaving, erasure recovery, checksum,
authentication, cryptographic integrity, streaming resynchronization, or
compatibility claim for another Hamming layout.

## Related Snippets

<!-- catalog:related:start -->
- [Recover One Missing Equal-Length Byte Shard with XOR Parity](recover-one-missing-equal-length-byte-shard-with-xor-parity.md)
- [Append a Fixed-Width CRC Check to a Human-Readable Identifier](../configuration-serialization/append-a-fixed-width-crc-check-to-a-human-readable-identifier.md)
- [Pack and Unpack a Bounded Boolean Tuple with Explicit Bit Length](../configuration-serialization/pack-and-unpack-a-bounded-boolean-tuple-with-explicit-bit-length.md)
<!-- catalog:related:end -->
