---
title: "Encode and Decode Bounded Bytes with a Canonical PackBits Profile"
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
  - encode-and-decode-a-bounded-byte-block-with-a-canonical-greedy-lz77-profile.md
  - derive-deterministic-huffman-code-lengths-from-bounded-byte-frequencies.md
  - ../data-processing/aggregate-consecutive-values-into-weighted-runs.md
---

# Encode and Decode Bounded Bytes with a Canonical PackBits Profile

## Idea and Problem

Represent a bounded byte string as literal and repeated-byte packets while choosing one deterministic minimum-length encoding.

A literal packet stores 1 through 128 bytes after a header in `0x00..0x7f`.
A repeat packet stores one byte to emit 2 through 128 times after a header in
`0x81..0xff`; `0x80` is deliberately invalid. Suffix dynamic programming
considers every legal next packet. Among encodings with the same minimum byte
length, it prefers a repeat packet over a literal packet and then the longer
current packet before applying the same rules to the remaining suffix.

The strict decoder parses only this closed grammar, bounds expansion, and then
re-encodes the result. That final comparison rejects valid-looking but
noncanonical alternatives such as two literal packets where one is sufficient.

## When to Use

Use this profile for deterministic fixtures, compact blocks with long runs, or
an instructional example of optimal packetization under a small byte grammar.
It is useful when inputs fit in memory and one exact canonical representation
is more important than streaming or sophisticated compression ratios.

Use an established compression or container format for exchanged data. This
profile intentionally defines no framing, version field, integrity check, or
interoperability contract. Choose a streaming codec when the entire decoded
block cannot be retained for canonical verification.

## Implementation

```python
_MAX_PACKBITS_DECODED_BYTES = 65_536
_MAX_PACKBITS_ENCODED_BYTES = 66_048
_MAX_PACKBITS_PACKET_BYTES = 128
_PACKBITS_REPEAT = 0
_PACKBITS_LITERAL = 1


def encode_canonical_packbits(data: bytes) -> bytes:
    """Encode exact bytes with the minimum-length canonical packetization."""
    if type(data) is not bytes:
        raise TypeError("data must be exact bytes")
    if len(data) > _MAX_PACKBITS_DECODED_BYTES:
        raise ValueError("data exceeds 65,536 bytes")

    size = len(data)
    if size == 0:
        return b""

    run_lengths = bytearray(size)
    for position in range(size - 1, -1, -1):
        if position + 1 < size and data[position] == data[position + 1]:
            run_lengths[position] = min(
                _MAX_PACKBITS_PACKET_BYTES,
                run_lengths[position + 1] + 1,
            )
        else:
            run_lengths[position] = 1

    costs = [0] * (size + 1)
    kinds = bytearray(size)
    lengths = bytearray(size)

    for position in range(size - 1, -1, -1):
        maximum_literal = min(_MAX_PACKBITS_PACKET_BYTES, size - position)
        best_cost = 2 + costs[position + 1]
        best_kind = _PACKBITS_LITERAL
        best_length = 1

        for packet_length in range(2, maximum_literal + 1):
            candidate_cost = packet_length + 1 + costs[position + packet_length]
            if candidate_cost < best_cost or (
                candidate_cost == best_cost and packet_length > best_length
            ):
                best_cost = candidate_cost
                best_length = packet_length

        for packet_length in range(2, run_lengths[position] + 1):
            candidate_cost = 2 + costs[position + packet_length]
            if candidate_cost < best_cost or (
                candidate_cost == best_cost
                and (best_kind == _PACKBITS_LITERAL or packet_length > best_length)
            ):
                best_cost = candidate_cost
                best_kind = _PACKBITS_REPEAT
                best_length = packet_length

        costs[position] = best_cost
        kinds[position] = best_kind
        lengths[position] = best_length

    encoded = bytearray()
    position = 0
    while position < size:
        packet_length = lengths[position]
        if kinds[position] == _PACKBITS_REPEAT:
            encoded.extend((257 - packet_length, data[position]))
        else:
            encoded.append(packet_length - 1)
            encoded.extend(data[position : position + packet_length])
        position += packet_length

    return bytes(encoded)


def decode_canonical_packbits(encoded: bytes) -> bytes:
    """Decode one canonical packet stream or reject it without partial output."""
    if type(encoded) is not bytes:
        raise TypeError("encoded must be exact bytes")
    if len(encoded) > _MAX_PACKBITS_ENCODED_BYTES:
        raise ValueError("encoded data exceeds 66,048 bytes")

    decoded = bytearray()
    position = 0
    while position < len(encoded):
        header = encoded[position]
        position += 1

        if header <= 0x7F:
            packet_length = header + 1
            packet_end = position + packet_length
            if packet_end > len(encoded):
                raise ValueError("truncated literal packet")
            if len(decoded) + packet_length > _MAX_PACKBITS_DECODED_BYTES:
                raise ValueError("decoded data exceeds 65,536 bytes")
            decoded.extend(encoded[position:packet_end])
            position = packet_end
        elif header == 0x80:
            raise ValueError("0x80 is not a packet in this profile")
        else:
            packet_length = 257 - header
            if position == len(encoded):
                raise ValueError("truncated repeat packet")
            if len(decoded) + packet_length > _MAX_PACKBITS_DECODED_BYTES:
                raise ValueError("decoded data exceeds 65,536 bytes")
            decoded.extend(bytes((encoded[position],)) * packet_length)
            position += 1

    result = bytes(decoded)
    if encode_canonical_packbits(result) != encoded:
        raise ValueError("encoded data is not canonical")
    return result
```

## Example

```python
from itertools import product


def all_packetizations(
    data: bytes,
    position: int = 0,
) -> list[tuple[bytes, tuple[tuple[int, int], ...]]]:
    if position == len(data):
        return [(b"", ())]

    results: list[tuple[bytes, tuple[tuple[int, int], ...]]] = []
    maximum = min(_MAX_PACKBITS_PACKET_BYTES, len(data) - position)
    for packet_length in range(1, maximum + 1):
        packet = bytes((packet_length - 1,)) + data[position : position + packet_length]
        for suffix, decisions in all_packetizations(data, position + packet_length):
            results.append(
                (
                    packet + suffix,
                    ((_PACKBITS_LITERAL, packet_length), *decisions),
                )
            )

    run_length = 1
    while run_length < maximum and data[position + run_length] == data[position]:
        run_length += 1
    for packet_length in range(2, run_length + 1):
        packet = bytes((257 - packet_length, data[position]))
        for suffix, decisions in all_packetizations(data, position + packet_length):
            results.append(
                (
                    packet + suffix,
                    ((_PACKBITS_REPEAT, packet_length), *decisions),
                )
            )
    return results


def oracle_key(
    candidate: tuple[bytes, tuple[tuple[int, int], ...]],
) -> tuple[int, tuple[tuple[int, int], ...]]:
    encoded, decisions = candidate
    tie_key = tuple((kind, -packet_length) for kind, packet_length in decisions)
    return len(encoded), tie_key


def rejects(encoded: object) -> bool:
    try:
        decode_canonical_packbits(encoded)  # type: ignore[arg-type]
    except (TypeError, ValueError):
        return True
    return False


examined_packetizations = 0
for size in range(9):
    for symbols in product((0, 1), repeat=size):
        data = bytes(symbols)
        candidates = all_packetizations(data)
        canonical, _ = min(candidates, key=oracle_key)
        actual = encode_canonical_packbits(data)
        assert actual == canonical
        assert decode_canonical_packbits(actual) == data
        for alternative, _ in candidates:
            assert (alternative == canonical) != rejects(alternative)
        examined_packetizations += len(candidates)

literal_128 = bytes(range(128))
literal_129 = bytes(range(129))
run_128 = b"x" * 128
run_129 = b"x" * 129
maximum_data = bytes(range(256)) * 256
maximum_encoded = encode_canonical_packbits(maximum_data)

assert encode_canonical_packbits(b"") == b""
assert encode_canonical_packbits(b"x") == b"\x00x"
assert encode_canonical_packbits(b"xx") == b"\xffx"
assert encode_canonical_packbits(literal_128) == b"\x7f" + literal_128
assert encode_canonical_packbits(literal_129) == (b"\x7f" + literal_128 + b"\x00\x80")
assert encode_canonical_packbits(run_128) == b"\x81x"
assert encode_canonical_packbits(run_129) == b"\x81x\x00x"
assert len(maximum_encoded) == _MAX_PACKBITS_ENCODED_BYTES
assert decode_canonical_packbits(maximum_encoded) == maximum_data

assert rejects(bytearray())
assert rejects(b"\x80")
assert rejects(b"\x00")
assert rejects(b"\x81")
assert rejects(b"\x01AA")
assert rejects(b"\x00A\x00B")
assert rejects(bytes((0x81, 0)) * 513)
assert rejects(b"\x00" * (_MAX_PACKBITS_ENCODED_BYTES + 1))

try:
    encode_canonical_packbits(b"x" * (_MAX_PACKBITS_DECODED_BYTES + 1))
except ValueError:
    pass
else:
    raise AssertionError("accepted oversized decoded input")

assert examined_packetizations == 82_621
```

## Trade-offs and Limitations

For `N` decoded bytes, the encoder examines at most 128 literal and 127 repeat
choices at each suffix, taking `O(128*N)` time and `O(N)` dynamic-programming
state. The returned bytes occupy at most `N + ceil(N/128)` bytes, which is
66,048 bytes at the decoded-size limit. Parsing is linear in encoded length;
strict canonical verification adds one encoding pass, for
`O(E + 128*D)` total decoder time and `O(D)` retained state.

Minimum encoded length is guaranteed only inside this packet grammar. The tie
rules choose one representation among equal-size packetizations; they do not
improve compression. Short or irregular data can grow, and the quadratic-looking
constant factor makes the encoder unsuitable for large blocks or latency-sensitive
paths.

There is no stream state, container framing, checksum, authentication,
encryption, entropy coding, dictionary compression, random access, or recovery
from corruption. The `0x80` byte is always rejected rather than treated as a
no-op, and the profile makes no compatibility claim with another PackBits
implementation or file format.

## Related Snippets

<!-- catalog:related:start -->
- [Encode and Decode a Bounded Byte Block with a Canonical Greedy LZ77 Profile](encode-and-decode-a-bounded-byte-block-with-a-canonical-greedy-lz77-profile.md)
- [Derive Deterministic Huffman Code Lengths from Bounded Byte Frequencies](derive-deterministic-huffman-code-lengths-from-bounded-byte-frequencies.md)
- [Aggregate Consecutive Values into Weighted Runs](../data-processing/aggregate-consecutive-values-into-weighted-runs.md)
<!-- catalog:related:end -->
