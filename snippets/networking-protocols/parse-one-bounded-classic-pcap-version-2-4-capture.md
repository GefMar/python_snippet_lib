---
title: "Parse One Bounded Classic PCAP Version 2.4 Capture"
snippet_type: recipe
use_cases:
  - interoperability
  - networking
  - parsing
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - parse-one-checksum-valid-bounded-ipv4-packet.md
  - parse-one-bounded-proxy-protocol-version-two-tcp-header-without-tlvs.md
  - ../testing-tooling/render-bounded-request-snapshots-into-a-length-framed-replay-payload.md
---

# Parse One Bounded Classic PCAP Version 2.4 Capture

## Idea and Problem

Parse one complete classic PCAP version 2.4 capture while selecting its byte order and timestamp resolution from the exact magic bytes stored on disk.

The 24-byte file header is followed by zero or more unpadded records. Each
record has a 16-byte header and exactly the declared number of captured packet
bytes. Four deployed magic-byte spellings distinguish little- and big-endian
fields with microsecond or nanosecond timestamp fractions; treating the magic
as an integer in the host's byte order can reverse that decision.

The legacy `Reserved1` and `Reserved2` fields are deliberately ignored. The
packed LinkType word is stricter: its reserved bits must be zero, and its
high-nibble FCS length is reported only when the `P` bit says that the value is
present. Packet payloads remain opaque bytes, and a legacy record whose
original length is smaller than its captured length is preserved as written.

## When to Use

Use this parser for bounded in-memory PCAP fixtures, offline metadata
inspection, import gates, or adapters that hand each returned packet to a
separate link-layer decoder. Its explicit limits are useful when the caller
expects one small, complete classic capture and wants malformed or truncated
input rejected before further processing.

Use libpcap, Wireshark, or another maintained capture library for live capture,
streaming input, PCAPNG, multiple interfaces, link-layer interpretation, or
large files. Captured packets can contain credentials and other sensitive
payloads; successful structural parsing neither anonymizes nor makes those
bytes safe to log or publish.

## Implementation

```python
from dataclasses import dataclass
from enum import StrEnum
from struct import Struct

_FILE_HEADER_BYTES = 24
_RECORD_HEADER_BYTES = 16
_MAX_CAPTURE_BYTES = 8 * 1024 * 1024
_MAX_SNAPSHOT_BYTES = 1024 * 1024
_MAX_RECORDS = 4096
_EXPECTED_VERSION = (2, 4)

_FCS_WORDS_SHIFT = 28
_RESERVED_R_MASK = 1 << 27
_FCS_PRESENT_MASK = 1 << 26
_RESERVED3_MASK = ((1 << 10) - 1) << 16
_LINK_TYPE_MASK = (1 << 16) - 1


class PcapByteOrder(StrEnum):
    LITTLE = "little"
    BIG = "big"


class PcapTimestampResolution(StrEnum):
    MICROSECONDS = "microseconds"
    NANOSECONDS = "nanoseconds"


_MAGIC_PROFILES: dict[
    bytes,
    tuple[str, PcapByteOrder, PcapTimestampResolution, int],
] = {
    b"\xd4\xc3\xb2\xa1": (
        "<",
        PcapByteOrder.LITTLE,
        PcapTimestampResolution.MICROSECONDS,
        1_000_000,
    ),
    b"\x4d\x3c\xb2\xa1": (
        "<",
        PcapByteOrder.LITTLE,
        PcapTimestampResolution.NANOSECONDS,
        1_000_000_000,
    ),
    b"\xa1\xb2\xc3\xd4": (
        ">",
        PcapByteOrder.BIG,
        PcapTimestampResolution.MICROSECONDS,
        1_000_000,
    ),
    b"\xa1\xb2\x3c\x4d": (
        ">",
        PcapByteOrder.BIG,
        PcapTimestampResolution.NANOSECONDS,
        1_000_000_000,
    ),
}


class ClassicPcapError(ValueError):
    """Raised when bytes are not one supported, bounded classic PCAP."""


@dataclass(frozen=True, slots=True)
class ClassicPcapRecord:
    timestamp_seconds: int
    timestamp_fraction: int
    original_length: int
    packet: bytes


@dataclass(frozen=True, slots=True)
class ClassicPcap:
    byte_order: PcapByteOrder
    timestamp_resolution: PcapTimestampResolution
    snapshot_length: int
    link_type: int
    frame_check_sequence_bytes: int | None
    records: tuple[ClassicPcapRecord, ...]


def parse_classic_pcap(capture: bytes) -> ClassicPcap:
    """Parse one complete, bounded classic PCAP version 2.4 capture."""
    if type(capture) is not bytes:
        raise TypeError("capture must be exactly bytes")
    if len(capture) < _FILE_HEADER_BYTES:
        raise ClassicPcapError("capture is shorter than the file header")
    if len(capture) > _MAX_CAPTURE_BYTES:
        raise ClassicPcapError("capture exceeds the 8 MiB limit")

    profile = _MAGIC_PROFILES.get(capture[:4])
    if profile is None:
        raise ClassicPcapError("unsupported classic PCAP magic bytes")
    struct_prefix, byte_order, timestamp_resolution, fraction_limit = profile

    file_header = Struct(f"{struct_prefix}HHIIII")
    (
        version_major,
        version_minor,
        _reserved1,
        _reserved2,
        snapshot_length,
        link_type_word,
    ) = file_header.unpack_from(capture, 4)

    if (version_major, version_minor) != _EXPECTED_VERSION:
        raise ClassicPcapError("only classic PCAP version 2.4 is supported")
    if not 1 <= snapshot_length <= _MAX_SNAPSHOT_BYTES:
        raise ClassicPcapError("snapshot length is outside the supported range")
    if link_type_word & _RESERVED_R_MASK:
        raise ClassicPcapError("the LinkType R bit must be zero")
    if link_type_word & _RESERVED3_MASK:
        raise ClassicPcapError("the LinkType Reserved3 bits must be zero")

    if link_type_word & _FCS_PRESENT_MASK:
        fcs_words = link_type_word >> _FCS_WORDS_SHIFT
        frame_check_sequence_bytes: int | None = fcs_words * 2
    else:
        frame_check_sequence_bytes = None

    record_header = Struct(f"{struct_prefix}IIII")
    records: list[ClassicPcapRecord] = []
    offset = _FILE_HEADER_BYTES

    while offset < len(capture):
        if len(records) >= _MAX_RECORDS:
            raise ClassicPcapError("capture exceeds the record-count limit")
        if len(capture) - offset < _RECORD_HEADER_BYTES:
            raise ClassicPcapError("capture ends inside a record header")

        (
            timestamp_seconds,
            timestamp_fraction,
            captured_length,
            original_length,
        ) = record_header.unpack_from(capture, offset)
        if timestamp_fraction >= fraction_limit:
            raise ClassicPcapError("record timestamp fraction is out of range")
        if captured_length > snapshot_length:
            raise ClassicPcapError("record captured length exceeds snapshot length")

        packet_start = offset + _RECORD_HEADER_BYTES
        packet_end = packet_start + captured_length
        if packet_end > len(capture):
            raise ClassicPcapError("capture ends inside a packet payload")

        records.append(
            ClassicPcapRecord(
                timestamp_seconds=timestamp_seconds,
                timestamp_fraction=timestamp_fraction,
                original_length=original_length,
                packet=capture[packet_start:packet_end],
            )
        )
        offset = packet_end

    return ClassicPcap(
        byte_order=byte_order,
        timestamp_resolution=timestamp_resolution,
        snapshot_length=snapshot_length,
        link_type=link_type_word & _LINK_TYPE_MASK,
        frame_check_sequence_bytes=frame_check_sequence_bytes,
        records=tuple(records),
    )
```

## Example

```python
variant_fixtures = (
    (
        bytes.fromhex(
            "d4c3b2a1 0200 0400 00000000 00000000 08000000 01000000 "
            "01000000 3f420f00 01000000 01000000 7f"
        ),
        PcapByteOrder.LITTLE,
        PcapTimestampResolution.MICROSECONDS,
        999_999,
    ),
    (
        bytes.fromhex(
            "4d3cb2a1 0200 0400 00000000 00000000 08000000 01000000 "
            "01000000 ffc99a3b 01000000 01000000 7f"
        ),
        PcapByteOrder.LITTLE,
        PcapTimestampResolution.NANOSECONDS,
        999_999_999,
    ),
    (
        bytes.fromhex(
            "a1b2c3d4 0002 0004 00000000 00000000 00000008 00000001 "
            "00000001 000f423f 00000001 00000001 7f"
        ),
        PcapByteOrder.BIG,
        PcapTimestampResolution.MICROSECONDS,
        999_999,
    ),
    (
        bytes.fromhex(
            "a1b23c4d 0002 0004 00000000 00000000 00000008 00000001 "
            "00000001 3b9ac9ff 00000001 00000001 7f"
        ),
        PcapByteOrder.BIG,
        PcapTimestampResolution.NANOSECONDS,
        999_999_999,
    ),
)

for raw_capture, expected_order, expected_resolution, expected_fraction in variant_fixtures:
    decoded = parse_classic_pcap(raw_capture)
    assert decoded.byte_order is expected_order
    assert decoded.timestamp_resolution is expected_resolution
    assert decoded.snapshot_length == 8
    assert decoded.link_type == 1
    assert decoded.frame_check_sequence_bytes is None
    assert decoded.records == (ClassicPcapRecord(1, expected_fraction, 1, b"\x7f"),)

# Two unpadded records; P is set and the high nibble declares two FCS words.
record_capture = bytes.fromhex(
    "a1b2c3d4 0002 0004 00000000 00000000 00000008 24000001 "
    "00000001 000f423f 00000003 00000005 616263 "
    "00000002 00000000 00000001 00000000 ff"
)
decoded_records = parse_classic_pcap(record_capture)
assert decoded_records.frame_check_sequence_bytes == 4
assert decoded_records.records == (
    ClassicPcapRecord(1, 999_999, 5, b"abc"),
    ClassicPcapRecord(2, 0, 0, b"\xff"),
)


def replaced_u32(
    capture: bytes,
    offset: int,
    value: int,
    byte_order: str,
) -> bytes:
    changed = bytearray(capture)
    changed[offset : offset + 4] = value.to_bytes(4, byte_order)
    return bytes(changed)


def rejected_as(
    candidate: bytes | bytearray,
    expected_error: type[Exception],
) -> bool:
    try:
        parse_classic_pcap(candidate)  # type: ignore[arg-type]
    except expected_error:
        return True
    return False


# Reserved1 and Reserved2 carry legacy values that a reader must ignore.
nonzero_ignored_fields = replaced_u32(record_capture, 8, 0x11223344, "big")
nonzero_ignored_fields = replaced_u32(
    nonzero_ignored_fields,
    12,
    0x55667788,
    "big",
)
assert parse_classic_pcap(nonzero_ignored_fields) == decoded_records

# A high-nibble value has no meaning unless P is set; P with zero words is zero.
fcs_not_present = replaced_u32(record_capture, 20, 0x20000001, "big")
zero_length_fcs = replaced_u32(record_capture, 20, 0x04000001, "big")
assert parse_classic_pcap(fcs_not_present).frame_check_sequence_bytes is None
assert parse_classic_pcap(zero_length_fcs).frame_check_sequence_bytes == 0

# Every prefix is rejected except exact file and record boundaries.
valid_prefix_lengths = {24, 43, len(record_capture)}
for prefix_length in range(len(record_capture) + 1):
    prefix = record_capture[:prefix_length]
    if prefix_length in valid_prefix_lengths:
        parse_classic_pcap(prefix)
    else:
        assert rejected_as(prefix, ClassicPcapError)

wrong_version = bytearray(variant_fixtures[2][0])
wrong_version[4:8] = bytes.fromhex("0003 0000")
too_many_records = variant_fixtures[2][0][:24] + bytes(16) * 4097

invalid_cases = (
    (bytearray(variant_fixtures[0][0]), TypeError),
    (bytes.fromhex("0a0d0d0a") + variant_fixtures[2][0][4:], ClassicPcapError),
    (bytes(wrong_version), ClassicPcapError),
    (replaced_u32(variant_fixtures[2][0], 16, 0, "big"), ClassicPcapError),
    (
        replaced_u32(
            variant_fixtures[2][0],
            16,
            _MAX_SNAPSHOT_BYTES + 1,
            "big",
        ),
        ClassicPcapError,
    ),
    (replaced_u32(record_capture, 20, 0x08000001, "big"), ClassicPcapError),
    (replaced_u32(record_capture, 20, 0x00010001, "big"), ClassicPcapError),
    (
        replaced_u32(variant_fixtures[2][0], 28, 1_000_000, "big"),
        ClassicPcapError,
    ),
    (
        replaced_u32(variant_fixtures[3][0], 28, 1_000_000_000, "big"),
        ClassicPcapError,
    ),
    (replaced_u32(record_capture, 32, 9, "big"), ClassicPcapError),
    (too_many_records, ClassicPcapError),
    (bytes(_MAX_CAPTURE_BYTES + 1), ClassicPcapError),
)
assert all(rejected_as(candidate, error) for candidate, error in invalid_cases)
```

## Trade-offs and Limitations

The parser performs `O(N)` work for an `N`-byte capture. It copies each packet
slice and stores every record, so returned data also uses `O(N)` memory within
the 8 MiB input, 1 MiB snapshot, and 4096-record limits. Header-only captures
with zero records and zero-length packet records are structurally valid.

Only classic PCAP version 2.4 is accepted. PCAPNG, streaming, concatenated
captures, link-type registry validation, packet decoding, timestamp conversion,
and FCS verification are outside the function. `frame_check_sequence_bytes`
describes capture metadata, not proof that an FCS is present or correct in any
particular packet.

The timestamp seconds and fractional fields are returned as untrusted integers.
`original_length < len(packet)` is intentionally preserved for compatibility
with legacy files rather than rejected or normalized. Conversely,
`captured_length > snapshot_length`, nonzero LinkType reserved bits, trailing
partial headers, and trailing partial payloads are rejected even if a more
permissive capture tool might attempt recovery.

## Related Snippets

<!-- catalog:related:start -->
- [Parse One Checksum-Valid Bounded IPv4 Packet](parse-one-checksum-valid-bounded-ipv4-packet.md)
- [Parse One Bounded PROXY Protocol Version Two TCP Header Without TLVs](parse-one-bounded-proxy-protocol-version-two-tcp-header-without-tlvs.md)
- [Render Bounded Request Snapshots into a Length-Framed Replay Payload](../testing-tooling/render-bounded-request-snapshots-into-a-length-framed-replay-payload.md)
<!-- catalog:related:end -->
