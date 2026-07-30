---
title: "Compute a 16-Bit Internet Checksum Across Bounded Byte Segments"
snippet_type: algorithm
use_cases:
  - data-transformation
  - networking
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../testing-tooling/enumerate-every-contiguous-chunking-of-bounded-bytes-for-stream-tests.md
  - ../storage-databases/combine-zlib-crc-32-values-for-concatenated-byte-segments.md
  - ../algorithms-data-structures/read-a-bounded-range-from-non-overlapping-byte-segments.md
---

# Compute a 16-Bit Internet Checksum Across Bounded Byte Segments

## Idea and Problem

Compute the 16-bit one's-complement Internet checksum of bounded byte segments exactly as if they had first been concatenated.

Bytes form big-endian 16-bit words. A segment ending after the high byte of a
word therefore cannot be padded immediately: that byte must be paired with the
first byte of the next non-empty segment. Only the final unmatched byte is
paired with zero. End-around carry is folded back into the low 16 bits before
the sum is complemented.

## When to Use

Use this function when a closed protocol field or packet image is already
available as immutable byte segments and its Internet checksum must be computed
without allocating their concatenation. Preserve the protocol's exact octet
order, and zero an existing checksum field before computing its replacement.

For verification, place the returned two checksum octets in their aligned
protocol field and checksum the complete word stream. Do not append them
naively after an odd-length payload, because that changes which bytes form
16-bit words.

## Implementation

```python
from itertools import pairwise

_MAX_CHECKSUM_SEGMENTS = 1_024
_MAX_CHECKSUM_BYTES = 1 << 20


def internet_checksum(segments: tuple[bytes, ...]) -> int:
    """Return the RFC 1071 checksum of the logical segment concatenation."""
    if type(segments) is not tuple:
        raise TypeError("segments must be an exact tuple")
    if len(segments) > _MAX_CHECKSUM_SEGMENTS:
        raise ValueError("segment count exceeds the supported limit")

    aggregate_size = 0
    for position, segment in enumerate(segments):
        if type(segment) is not bytes:
            raise TypeError(f"segments[{position}] must be exact bytes")
        aggregate_size += len(segment)
        if aggregate_size > _MAX_CHECKSUM_BYTES:
            raise ValueError("aggregate content exceeds the supported byte limit")

    word_sum = 0
    pending_high: int | None = None
    for segment in segments:
        offset = 0
        if pending_high is not None and segment:
            word_sum += (pending_high << 8) | segment[0]
            pending_high = None
            offset = 1

        paired_end = len(segment) - ((len(segment) - offset) & 1)
        while offset < paired_end:
            word_sum += (segment[offset] << 8) | segment[offset + 1]
            offset += 2
        if offset < len(segment):
            pending_high = segment[offset]

    if pending_high is not None:
        word_sum += pending_high << 8

    while word_sum >> 16:
        word_sum = (word_sum & 0xFFFF) + (word_sum >> 16)
    return (~word_sum) & 0xFFFF
```

## Example

```python
def concatenating_oracle(data: bytes) -> int:
    padded = data if len(data) % 2 == 0 else data + b"\x00"
    word_sum = sum(
        (padded[offset] << 8) | padded[offset + 1]
        for offset in range(0, len(padded), 2)
    )
    while word_sum >> 16:
        word_sum = (word_sum & 0xFFFF) + (word_sum >> 16)
    return (~word_sum) & 0xFFFF


def every_chunking(data: bytes) -> tuple[tuple[bytes, ...], ...]:
    if not data:
        return ((), (b"",))
    chunkings: list[tuple[bytes, ...]] = []
    for cut_mask in range(1 << (len(data) - 1)):
        starts = [0]
        starts.extend(
            offset
            for offset in range(1, len(data))
            if cut_mask & (1 << (offset - 1))
        )
        starts.append(len(data))
        chunkings.append(
            tuple(data[left:right] for left, right in pairwise(starts))
        )
    return tuple(chunkings)


payloads = (
    *(
        bytes((length + 61 * offset) & 0xFF for offset in range(length))
        for length in range(11)
    ),
    b"\xff" * 9,
    b"\xff\xff\x00\x01\xff\xfe\x80",
)

checked_chunkings = 0
for payload in payloads:
    expected = concatenating_oracle(payload)
    for chunking in every_chunking(payload):
        assert internet_checksum(chunking) == expected
        checked_chunkings += 1

assert checked_chunkings > 1_000
assert internet_checksum(()) == 0xFFFF
assert internet_checksum((b"\x01", b"", b"\x02")) == 0xFEFD
assert internet_checksum((b"\xff\xff", b"\xff\xff")) == 0x0000

packet_with_zero_field = b"\x12\x34\x00\x00\xab"
checksum = internet_checksum((packet_with_zero_field,))
packet = packet_with_zero_field[:2] + checksum.to_bytes(2, "big") + b"\xab"
assert checksum == 0x42CB
assert internet_checksum((packet,)) == 0x0000

assert internet_checksum(tuple(b"" for _ in range(_MAX_CHECKSUM_SEGMENTS))) == 0xFFFF

try:
    internet_checksum((b"x" * (_MAX_CHECKSUM_BYTES + 1),))
except ValueError:
    pass
else:
    raise AssertionError("an oversized checksum input was accepted")

assert (
    checked_chunkings > 1_000
    and internet_checksum((b"\x12", b"\x34\xab"))
    == concatenating_oracle(b"\x12\x34\xab")
)
```

## Trade-offs and Limitations

For `S` segments containing `B` bytes, the function uses `O(S + B)` time and
`O(1)` auxiliary space. Python's integer accumulator may grow above 16 bits
until the final fold, but the 1 MiB input cap bounds that intermediate value.
Empty segments are harmless and do not consume a pending high byte.

This is only the closed byte checksum. The caller remains responsible for
constructing any protocol pseudo-header, locating and zeroing the checksum
field, and parsing packet boundaries. The function does not implement
incremental field replacement. A 16-bit one's-complement checksum detects some
accidental corruption but is not collision resistant and provides no
authentication or protection against deliberate modification.

## Related Snippets

<!-- catalog:related:start -->
- [Enumerate Every Contiguous Chunking of Bounded Bytes for Stream Tests](../testing-tooling/enumerate-every-contiguous-chunking-of-bounded-bytes-for-stream-tests.md)
- [Combine zlib CRC-32 Values for Concatenated Byte Segments](../storage-databases/combine-zlib-crc-32-values-for-concatenated-byte-segments.md)
- [Read a Bounded Range from Non-Overlapping Byte Segments](../algorithms-data-structures/read-a-bounded-range-from-non-overlapping-byte-segments.md)
<!-- catalog:related:end -->
