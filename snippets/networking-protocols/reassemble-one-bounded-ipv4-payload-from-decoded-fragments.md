---
title: "Reassemble One Bounded IPv4 Payload from Decoded Fragments"
snippet_type: algorithm
use_cases:
  - data-transformation
  - interoperability
  - networking
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - reassemble-one-bounded-websocket-message-from-decoded-data-fragments.md
  - compute-a-16-bit-internet-checksum-across-bounded-byte-segments.md
  - ../testing-tooling/enumerate-every-contiguous-chunking-of-bounded-bytes-for-stream-tests.md
---

# Reassemble One Bounded IPv4 Payload from Decoded Fragments

## Idea and Problem

Reassemble one already grouped IPv4 payload while rejecting every ambiguous fragment layout.

The decoded fragment offset counts eight-byte units. A non-final fragment must
therefore contain a positive multiple of eight payload bytes, while the sole
final fragment fixes the reassembled payload length. Sorting fragments by
their decoded offsets makes caller order irrelevant, and requiring exact
adjacency rejects gaps, overlaps, and duplicates under one fail-closed policy.

The first fragment's decoded header length bounds the payload against IPv4's
65,535-byte total-length field. The result retains canonical half-open spans
and the complete header-plus-payload length as immutable evidence.

## When to Use

Use this algorithm after a trusted IPv4 parser has validated each header and
grouped fragments by source, destination, protocol, and identification. Pass
the header length from the offset-zero fragment and application payload bytes
only. This is useful for bounded protocol fixtures, offline inspection, and
adapters that deliberately reject every overlap instead of reproducing a
host-specific overlap policy.

Use an operating-system IP stack or a maintained packet-processing library for
live traffic. A live reassembler also needs arrival deadlines, resource
accounting across incomplete datagrams, identification-reuse handling, ECN and
option policy, checksum validation, and eviction of abandoned fragment sets.

## Implementation

```python
from dataclasses import dataclass

_MIN_IPV4_HEADER_LENGTH = 20
_MAX_IPV4_HEADER_LENGTH = 60
_MAX_IPV4_TOTAL_LENGTH = 65_535
_MAX_FRAGMENT_OFFSET_UNITS = (1 << 13) - 1
_MAX_FRAGMENT_COUNT = 128
_MAX_SUPPLIED_PAYLOAD_BYTES = _MAX_IPV4_TOTAL_LENGTH - _MIN_IPV4_HEADER_LENGTH


class IPv4FragmentSequenceError(ValueError):
    pass


@dataclass(frozen=True, slots=True)
class DecodedIPv4Fragment:
    offset_units: int
    more_fragments: bool
    payload: bytes


@dataclass(frozen=True, slots=True)
class IPv4PayloadSpan:
    start: int
    stop: int


@dataclass(frozen=True, slots=True)
class ReassembledIPv4Payload:
    payload: bytes
    fragment_spans: tuple[IPv4PayloadSpan, ...]
    first_header_length: int
    total_length: int


def reassemble_ipv4_payload(
    fragments: tuple[DecodedIPv4Fragment, ...],
    *,
    first_header_length: int,
) -> ReassembledIPv4Payload:
    """Return one strict, complete IPv4 payload and its canonical spans."""
    if type(fragments) is not tuple:
        raise TypeError("fragments must be an exact tuple")
    if not 1 <= len(fragments) <= _MAX_FRAGMENT_COUNT:
        raise ValueError("fragment count is outside the supported range")
    if type(first_header_length) is not int:
        raise TypeError("first_header_length must be an exact integer")
    if not _MIN_IPV4_HEADER_LENGTH <= first_header_length <= _MAX_IPV4_HEADER_LENGTH:
        raise ValueError("first_header_length is outside the IPv4 header range")
    if first_header_length % 4 != 0:
        raise ValueError("first_header_length must be a multiple of four bytes")

    maximum_payload_length = _MAX_IPV4_TOTAL_LENGTH - first_header_length
    validated: list[tuple[int, int, DecodedIPv4Fragment]] = []
    supplied_payload_bytes = 0
    offset_zero_count = 0
    final_count = 0

    for index, fragment in enumerate(fragments):
        if type(fragment) is not DecodedIPv4Fragment:
            raise TypeError(f"fragments[{index}] must be an exact decoded fragment")
        if type(fragment.offset_units) is not int:
            raise TypeError(f"fragments[{index}].offset_units must be an exact integer")
        if not 0 <= fragment.offset_units <= _MAX_FRAGMENT_OFFSET_UNITS:
            raise ValueError(f"fragments[{index}].offset_units is outside the 13-bit range")
        if type(fragment.more_fragments) is not bool:
            raise TypeError(f"fragments[{index}].more_fragments must be an exact boolean")
        if type(fragment.payload) is not bytes:
            raise TypeError(f"fragments[{index}].payload must be exact bytes")
        if len(fragment.payload) > _MAX_SUPPLIED_PAYLOAD_BYTES - supplied_payload_bytes:
            raise ValueError("aggregate supplied payload exceeds the supported limit")

        supplied_payload_bytes += len(fragment.payload)
        start = fragment.offset_units * 8
        stop = start + len(fragment.payload)
        if stop > maximum_payload_length:
            raise ValueError(f"fragments[{index}] extends beyond the IPv4 total-length limit")

        if fragment.more_fragments:
            if not fragment.payload or len(fragment.payload) % 8 != 0:
                raise IPv4FragmentSequenceError(
                    "every non-final fragment needs a positive eight-byte payload multiple"
                )
        elif not fragment.payload and not (len(fragments) == 1 and fragment.offset_units == 0):
            raise IPv4FragmentSequenceError(
                "an empty payload is allowed only for one complete offset-zero datagram"
            )

        offset_zero_count += fragment.offset_units == 0
        final_count += not fragment.more_fragments
        validated.append((start, stop, fragment))

    if offset_zero_count != 1:
        raise IPv4FragmentSequenceError("exactly one fragment must start at offset zero")
    if final_count != 1:
        raise IPv4FragmentSequenceError("exactly one fragment must be final")

    ordered = sorted(validated, key=lambda item: (item[0], item[1]))
    if ordered[-1][2].more_fragments:
        raise IPv4FragmentSequenceError("the final fragment must be last by offset")

    expected_start = 0
    payload_parts: list[bytes] = []
    spans: list[IPv4PayloadSpan] = []
    for start, stop, fragment in ordered:
        if start < expected_start:
            raise IPv4FragmentSequenceError("fragment spans overlap or are duplicated")
        if start > expected_start:
            raise IPv4FragmentSequenceError("fragment spans contain a gap")
        spans.append(IPv4PayloadSpan(start, stop))
        payload_parts.append(fragment.payload)
        expected_start = stop

    total_length = first_header_length + expected_start
    if total_length > _MAX_IPV4_TOTAL_LENGTH:
        raise AssertionError("validated IPv4 total length must fit its field")
    return ReassembledIPv4Payload(
        payload=b"".join(payload_parts),
        fragment_spans=tuple(spans),
        first_header_length=first_header_length,
        total_length=total_length,
    )
```

## Example

```python
def every_aligned_fragmentation(
    payload: bytes,
) -> tuple[tuple[DecodedIPv4Fragment, ...], ...]:
    cut_positions = tuple(range(8, len(payload), 8))
    fragmentations: list[tuple[DecodedIPv4Fragment, ...]] = []
    for cut_mask in range(1 << len(cut_positions)):
        stops = (
            *(position for index, position in enumerate(cut_positions) if cut_mask & (1 << index)),
            len(payload),
        )
        starts = (0, *stops[:-1])
        fragmentations.append(
            tuple(
                DecodedIPv4Fragment(
                    offset_units=start // 8,
                    more_fragments=stop != len(payload),
                    payload=payload[start:stop],
                )
                for start, stop in zip(starts, stops, strict=True)
            )
        )
    return tuple(fragmentations)


def ownership_array_oracle(
    fragments: tuple[DecodedIPv4Fragment, ...],
    *,
    first_header_length: int,
) -> bytes:
    final_stops = tuple(
        fragment.offset_units * 8 + len(fragment.payload)
        for fragment in fragments
        if not fragment.more_fragments
    )
    assert len(final_stops) == 1
    payload_length = final_stops[0]
    assert first_header_length + payload_length <= _MAX_IPV4_TOTAL_LENGTH

    owners = [-1] * payload_length
    payload = bytearray(payload_length)
    for fragment_index, fragment in enumerate(fragments):
        start = fragment.offset_units * 8
        for position, value in enumerate(fragment.payload, start=start):
            assert position < payload_length
            assert owners[position] == -1
            owners[position] = fragment_index
            payload[position] = value
    assert all(owner != -1 for owner in owners)
    return bytes(payload)


def every_fragment_ordering(
    fragments: tuple[DecodedIPv4Fragment, ...],
):
    from itertools import permutations

    return permutations(fragments)


sample_payload = b"abcdefghijklmnopqrstuvwxy"
checked_layouts = 0
for fragmentation in every_aligned_fragmentation(sample_payload):
    expected_spans = tuple(
        IPv4PayloadSpan(
            fragment.offset_units * 8,
            fragment.offset_units * 8 + len(fragment.payload),
        )
        for fragment in sorted(fragmentation, key=lambda item: item.offset_units)
    )
    for candidate in every_fragment_ordering(fragmentation):
        reassembled = reassemble_ipv4_payload(candidate, first_header_length=20)
        assert reassembled.payload == ownership_array_oracle(
            candidate,
            first_header_length=20,
        )
        assert reassembled.fragment_spans == expected_spans
        assert reassembled.total_length == 20 + len(sample_payload)
        checked_layouts += 1

empty = reassemble_ipv4_payload(
    (DecodedIPv4Fragment(0, False, b""),),
    first_header_length=20,
)
maximum_with_short_header = reassemble_ipv4_payload(
    (DecodedIPv4Fragment(0, False, b"x" * 65_515),),
    first_header_length=20,
)
maximum_with_long_header = reassemble_ipv4_payload(
    (DecodedIPv4Fragment(0, False, b"x" * 65_475),),
    first_header_length=60,
)
maximum_count = reassemble_ipv4_payload(
    tuple(
        DecodedIPv4Fragment(
            offset_units=index,
            more_fragments=index != _MAX_FRAGMENT_COUNT - 1,
            payload=b"x" * (8 if index != _MAX_FRAGMENT_COUNT - 1 else 1),
        )
        for index in range(_MAX_FRAGMENT_COUNT)
    ),
    first_header_length=20,
)


def rejected_as(
    fragments: object,
    first_header_length: object,
    expected_error: type[Exception],
) -> bool:
    try:
        reassemble_ipv4_payload(
            fragments,  # type: ignore[arg-type]
            first_header_length=first_header_length,  # type: ignore[arg-type]
        )
    except expected_error:
        return True
    return False


too_many = tuple(DecodedIPv4Fragment(0, False, b"") for _ in range(_MAX_FRAGMENT_COUNT + 1))
invalid_cases = (
    ([], 20, TypeError),
    ((), 20, ValueError),
    (too_many, 20, ValueError),
    ((DecodedIPv4Fragment(0, False, b"x"),), True, TypeError),
    ((DecodedIPv4Fragment(0, False, b"x"),), 19, ValueError),
    ((DecodedIPv4Fragment(0, False, b"x"),), 22, ValueError),
    ((DecodedIPv4Fragment(0, False, b"x"),), 61, ValueError),
    ((object(),), 20, TypeError),
    ((DecodedIPv4Fragment(True, False, b"x"),), 20, TypeError),
    ((DecodedIPv4Fragment(-1, False, b"x"),), 20, ValueError),
    ((DecodedIPv4Fragment(8_192, False, b"x"),), 20, ValueError),
    ((DecodedIPv4Fragment(0, 0, b"x"),), 20, TypeError),
    ((DecodedIPv4Fragment(0, False, bytearray(b"x")),), 20, TypeError),
    ((DecodedIPv4Fragment(0, True, b""),), 20, IPv4FragmentSequenceError),
    ((DecodedIPv4Fragment(0, True, b"short"),), 20, IPv4FragmentSequenceError),
    ((DecodedIPv4Fragment(1, False, b"x"),), 20, IPv4FragmentSequenceError),
    ((DecodedIPv4Fragment(0, True, b"12345678"),), 20, IPv4FragmentSequenceError),
    (
        (
            DecodedIPv4Fragment(0, False, b"12345678"),
            DecodedIPv4Fragment(1, False, b"x"),
        ),
        20,
        IPv4FragmentSequenceError,
    ),
    (
        (
            DecodedIPv4Fragment(0, False, b"12345678"),
            DecodedIPv4Fragment(1, True, b"12345678"),
        ),
        20,
        IPv4FragmentSequenceError,
    ),
    (
        (
            DecodedIPv4Fragment(0, True, b"12345678"),
            DecodedIPv4Fragment(2, False, b"x"),
        ),
        20,
        IPv4FragmentSequenceError,
    ),
    (
        (
            DecodedIPv4Fragment(0, True, b"12345678abcdefgh"),
            DecodedIPv4Fragment(1, False, b"x"),
        ),
        20,
        IPv4FragmentSequenceError,
    ),
    (
        (
            DecodedIPv4Fragment(0, True, b"12345678"),
            DecodedIPv4Fragment(1, True, b"abcdefgh"),
            DecodedIPv4Fragment(1, False, b"ABCDEFGH"),
        ),
        20,
        IPv4FragmentSequenceError,
    ),
    (
        (
            DecodedIPv4Fragment(0, True, b"12345678"),
            DecodedIPv4Fragment(1, False, b""),
        ),
        20,
        IPv4FragmentSequenceError,
    ),
    ((DecodedIPv4Fragment(0, False, b"x" * 65_516),), 20, ValueError),
    ((DecodedIPv4Fragment(0, False, b"x" * 65_476),), 60, ValueError),
    ((DecodedIPv4Fragment(8_191, False, b"x"),), 20, ValueError),
)
rejected_count = sum(
    rejected_as(fragments, header_length, expected_error)
    for fragments, header_length, expected_error in invalid_cases
)

assert (
    checked_layouts,
    empty,
    maximum_with_short_header.total_length,
    maximum_with_long_header.total_length,
    len(maximum_count.fragment_spans),
    len(maximum_count.payload),
    rejected_count,
) == (
    49,
    ReassembledIPv4Payload(b"", (IPv4PayloadSpan(0, 0),), 20, 20),
    _MAX_IPV4_TOTAL_LENGTH,
    _MAX_IPV4_TOTAL_LENGTH,
    _MAX_FRAGMENT_COUNT,
    1_017,
    len(invalid_cases),
)
```

## Trade-offs and Limitations

For `f` fragments and `p` reassembled payload bytes, validation and sorting
take `O(f log f)` time and joining takes `O(p)` time and space. The exact input
payloads, an `O(f)` sorted list, canonical spans, and the joined result coexist.
The 128-fragment and 65,515-byte supplied-payload caps bound that work before
reassembly. The header-dependent check may impose the smaller payload limit
required by a longer offset-zero header.

This closed profile rejects all overlaps and duplicates, including byte-for-byte
identical retransmissions. Every non-final fragment contains a positive
multiple of eight bytes. The only empty form is one final offset-zero fragment,
which represents a header-only datagram. Returned spans are sorted, adjacent,
and cover the payload from zero through its exact length; `total_length`
includes the first header.

The function does not parse headers, verify checksums, group fragments, combine
header fields, process options or ECN, retain incomplete sets, apply a timeout,
or choose an overlap winner. It cannot by itself protect a live receiver from
fragment floods or identification reuse.

## Related Snippets

<!-- catalog:related:start -->
- [Reassemble One Bounded WebSocket Message from Decoded Data Fragments](reassemble-one-bounded-websocket-message-from-decoded-data-fragments.md)
- [Compute a 16-Bit Internet Checksum Across Bounded Byte Segments](compute-a-16-bit-internet-checksum-across-bounded-byte-segments.md)
- [Enumerate Every Contiguous Chunking of Bounded Bytes for Stream Tests](../testing-tooling/enumerate-every-contiguous-chunking-of-bounded-bytes-for-stream-tests.md)
<!-- catalog:related:end -->
