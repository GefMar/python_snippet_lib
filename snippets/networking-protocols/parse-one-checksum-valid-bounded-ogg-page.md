---
title: "Parse One Checksum-Valid Bounded Ogg Page"
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
  - parse-one-bounded-classic-pcap-version-2-4-capture.md
  - ../configuration-serialization/inspect-bounded-png-critical-chunk-structure-without-decompressing-pixels.md
  - ../storage-databases/combine-zlib-crc-32-values-for-concatenated-byte-segments.md
---

# Parse One Checksum-Valid Bounded Ogg Page

## Idea and Problem

Parse exactly one complete Ogg version 0 page, derive its body boundary from the lacing table, and verify its page checksum before returning immutable raw fields.

[RFC 3533](https://www.rfc-editor.org/rfc/rfc3533) defines a 27-byte fixed
header followed by at most 255 one-byte lacing values and at most 255 bytes of
body for each value. Those wire limits make a page inherently bounded to
`27 + 255 + 255 * 255 = 65,307` bytes. All multibyte fields are little-endian;
the granule position is signed because the two's-complement value `-1` means
that no packet finishes on the page.

The RFC names the checksum polynomial but not its complete parameter set. The
[Xiph framing documentation](https://xiph.org/ogg/doc/framing.html) specifies
the interoperable profile used here: direct, unreflected CRC-32 with initial
value and final XOR both zero, using polynomial `0x04C11DB7`. This is not the
reflected CRC convention implemented by `binascii.crc32` or `zlib.crc32`.

## When to Use

Use this parser for bounded in-memory fixtures, offline page inspection, import
gates, or adapters that have already isolated one complete Ogg page. It
rejects corrupt framing, truncation, trailing bytes, unsupported stream
structure versions, and checksum mismatches before a codec sees the body.

Use libogg or another maintained Ogg implementation for stream capture,
resynchronization, multiplexing, seeking, packet reconstruction, or codec
decoding. A valid page checksum detects accidental corruption; it does not
authenticate the page or make its opaque body safe to decode, log, or publish.

## Implementation

```python
from dataclasses import dataclass
from struct import Struct

_FIXED_HEADER_BYTES = 27
_MAX_PAGE_SEGMENTS = 255
_MAX_PAGE_BODY_BYTES = _MAX_PAGE_SEGMENTS * 255
_MAX_PAGE_BYTES = _FIXED_HEADER_BYTES + _MAX_PAGE_SEGMENTS + _MAX_PAGE_BODY_BYTES
_OGG_CRC_POLYNOMIAL = 0x04C11DB7
_UINT32_MASK = 0xFFFF_FFFF
_PAGE_HEADER = Struct("<4sBBqIIIB")


class OggPageError(ValueError):
    """Raised when bytes are not one checksum-valid bounded Ogg page."""


@dataclass(frozen=True, slots=True)
class OggPage:
    header_type: int
    granule_position: int
    bitstream_serial_number: int
    page_sequence_number: int
    checksum: int
    lacing_values: tuple[int, ...]
    body: bytes


def _build_ogg_crc_table() -> tuple[int, ...]:
    table: list[int] = []
    for leading_byte in range(256):
        remainder = leading_byte << 24
        for _ in range(8):
            if remainder & 0x8000_0000:
                remainder = ((remainder << 1) ^ _OGG_CRC_POLYNOMIAL) & _UINT32_MASK
            else:
                remainder = (remainder << 1) & _UINT32_MASK
        table.append(remainder)
    return tuple(table)


_OGG_CRC_TABLE = _build_ogg_crc_table()


def _ogg_page_checksum(encoded: bytes) -> int:
    checksum = 0
    for offset, byte in enumerate(encoded):
        if 22 <= offset < 26:
            byte = 0
        table_index = ((checksum >> 24) ^ byte) & 0xFF
        checksum = ((checksum << 8) & _UINT32_MASK) ^ _OGG_CRC_TABLE[table_index]
    return checksum


def parse_ogg_page(encoded: bytes) -> OggPage:
    """Parse exactly one complete checksum-valid Ogg version 0 page."""
    if type(encoded) is not bytes:
        raise TypeError("encoded must be exact immutable bytes")
    if not _FIXED_HEADER_BYTES <= len(encoded) <= _MAX_PAGE_BYTES:
        raise OggPageError("page length is outside 27..65,307 bytes")

    (
        capture_pattern,
        version,
        header_type,
        granule_position,
        bitstream_serial_number,
        page_sequence_number,
        checksum,
        page_segments,
    ) = _PAGE_HEADER.unpack_from(encoded)

    if capture_pattern != b"OggS":
        raise OggPageError("capture pattern must be OggS")
    if version != 0:
        raise OggPageError("only Ogg stream structure version 0 is supported")

    segment_table_end = _FIXED_HEADER_BYTES + page_segments
    if segment_table_end > len(encoded):
        raise OggPageError("page ends inside its segment table")

    # RFC 3533 assigns meanings to bits 0..2 but does not require the other
    # header-type bits to be zero, so the exact raw byte is preserved.
    lacing_values = tuple(encoded[_FIXED_HEADER_BYTES:segment_table_end])
    expected_page_bytes = segment_table_end + sum(lacing_values)
    if expected_page_bytes != len(encoded):
        raise OggPageError("lacing values do not frame exactly one page body")
    if _ogg_page_checksum(encoded) != checksum:
        raise OggPageError("page checksum does not match its header and body")

    return OggPage(
        header_type=header_type,
        granule_position=granule_position,
        bitstream_serial_number=bitstream_serial_number,
        page_sequence_number=page_sequence_number,
        checksum=checksum,
        lacing_values=lacing_values,
        body=encoded[segment_table_end:],
    )
```

## Example

```python
def reference_ogg_crc32(data: bytes) -> int:
    """Independent bitwise oracle for the direct, unreflected Ogg CRC."""
    checksum = 0
    for byte in data:
        checksum ^= byte << 24
        for _ in range(8):
            if checksum & 0x8000_0000:
                checksum = ((checksum << 1) ^ 0x04C11DB7) & 0xFFFF_FFFF
            else:
                checksum = (checksum << 1) & 0xFFFF_FFFF
    return checksum


assert reference_ogg_crc32(b"123456789") == 0x89A1897F


def build_ogg_page(
    *,
    header_type: int = 0,
    granule_position: int = 0,
    bitstream_serial_number: int = 1,
    page_sequence_number: int = 0,
    lacing_values: tuple[int, ...] = (),
    body: bytes = b"",
) -> bytes:
    """Build neutral fixtures without calling the production CRC helper."""
    if len(lacing_values) > 255:
        raise ValueError("fixture has too many lacing values")
    if any(not 0 <= value <= 255 for value in lacing_values):
        raise ValueError("fixture lacing values must be bytes")
    if len(body) != sum(lacing_values):
        raise ValueError("fixture body length does not match its lacing values")

    page_with_zero_checksum = (
        b"OggS"
        + bytes((0, header_type))
        + granule_position.to_bytes(8, "little", signed=True)
        + bitstream_serial_number.to_bytes(4, "little")
        + page_sequence_number.to_bytes(4, "little")
        + bytes(4)
        + bytes((len(lacing_values),))
        + bytes(lacing_values)
        + body
    )
    checksum = reference_ogg_crc32(page_with_zero_checksum)
    return (
        page_with_zero_checksum[:22] + checksum.to_bytes(4, "little") + page_with_zero_checksum[26:]
    )


empty_page = bytes.fromhex("4f676753 00 00 0000000000000000 01000000 00000000 14f30576 00")
nil_packet_page = bytes.fromhex("4f676753 00 02 0000000000000000 01000000 00000000 29d64021 01 00")
data_page = bytes.fromhex("4f676753 00 00 ffffffffffffffff 78563412 07000000 a817fbcb 01 03 616263")

assert empty_page == build_ogg_page()
assert nil_packet_page == build_ogg_page(header_type=0x02, lacing_values=(0,))
assert data_page == build_ogg_page(
    granule_position=-1,
    bitstream_serial_number=0x12345678,
    page_sequence_number=7,
    lacing_values=(3,),
    body=b"abc",
)

assert parse_ogg_page(empty_page).lacing_values == ()
assert parse_ogg_page(nil_packet_page).lacing_values == (0,)
decoded = parse_ogg_page(data_page)
assert decoded.granule_position == -1
assert decoded.bitstream_serial_number == 0x12345678
assert decoded.page_sequence_number == 7
assert decoded.checksum == 0xCBFB17A8
assert decoded.lacing_values == (3,)
assert decoded.body == b"abc"

unknown_flags_page = build_ogg_page(
    header_type=0xF8,
    lacing_values=(1,),
    body=b"x",
)
assert parse_ogg_page(unknown_flags_page).header_type == 0xF8

continued_page = build_ogg_page(lacing_values=(255,), body=bytes(255))
assert parse_ogg_page(continued_page).lacing_values == (255,)

maximum_page = build_ogg_page(
    lacing_values=(255,) * 255,
    body=b"x" * (255 * 255),
)
maximum_decoded = parse_ogg_page(maximum_page)
assert len(maximum_page) == 65_307
assert len(maximum_decoded.lacing_values) == 255
assert len(maximum_decoded.body) == 65_025


def rejected_as(candidate: object, expected_error: type[Exception]) -> bool:
    try:
        parse_ogg_page(candidate)  # type: ignore[arg-type]
    except expected_error:
        return True
    return False


wrong_magic = b"OggX" + empty_page[4:]
wrong_version = empty_page[:4] + b"\x01" + empty_page[5:]
missing_segment_table = empty_page[:-1] + b"\x01"
wrong_checksum = bytearray(data_page)
wrong_checksum[22] ^= 1
wrong_body = bytearray(data_page)
wrong_body[-1] ^= 1

split_page = build_ogg_page(lacing_values=(1, 2), body=b"abc")
wrong_lacing = bytearray(split_page)
wrong_lacing[27], wrong_lacing[28] = wrong_lacing[28], wrong_lacing[27]

invalid_cases = (
    (bytearray(empty_page), TypeError),
    (empty_page[:-1], OggPageError),
    (bytes(65_308), OggPageError),
    (wrong_magic, OggPageError),
    (wrong_version, OggPageError),
    (missing_segment_table, OggPageError),
    (data_page[:-1], OggPageError),
    (data_page + b"\x00", OggPageError),
    (bytes(wrong_checksum), OggPageError),
    (bytes(wrong_body), OggPageError),
    (bytes(wrong_lacing), OggPageError),
)
assert all(rejected_as(candidate, error) for candidate, error in invalid_cases)
```

## Trade-offs and Limitations

Parsing and checksum verification take `O(N)` time for an `N`-byte page. The
lookup table has 256 entries; the returned body slice and lacing tuple copy at
most 65,025 and 255 values respectively. The protocol's own limits remove the
need for an arbitrary local page-size policy.

A zero segment count is accepted by this structural parser, and a lacing value
of zero represents a valid nil packet. A final value of 255 means that a packet
continues on a later page; this parser deliberately returns lacing and body
bytes rather than claiming that it reconstructed packets. Header-type bits
outside `0x01`, `0x02`, and `0x04` are preserved without interpretation because
RFC 3533 does not state that they must be zero.

The parser does not search for a capture pattern, validate adjacent sequence
numbers, enforce beginning/end/continuation relationships, interpret granule
positions, identify codecs, or decode payloads. The CRC profile is also not
zlib CRC-32. The RFC's CRC description is known to be underspecified; the
current distinction is recorded in
[reported erratum 8824](https://www.rfc-editor.org/errata/eid8824), while this
recipe follows the explicit Xiph framing parameters.

## Related Snippets

<!-- catalog:related:start -->
- [Parse One Bounded Classic PCAP Version 2.4 Capture](parse-one-bounded-classic-pcap-version-2-4-capture.md)
- [Inspect Bounded PNG Critical-Chunk Structure without Decompressing Pixels](../configuration-serialization/inspect-bounded-png-critical-chunk-structure-without-decompressing-pixels.md)
- [Combine zlib CRC-32 Values for Concatenated Byte Segments](../storage-databases/combine-zlib-crc-32-values-for-concatenated-byte-segments.md)
<!-- catalog:related:end -->
