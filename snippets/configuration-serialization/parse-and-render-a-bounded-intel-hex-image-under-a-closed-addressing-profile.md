---
title: "Parse and Render a Bounded Intel HEX Image Under a Closed Addressing Profile"
snippet_type: recipe
use_cases:
  - interoperability
  - parsing
  - serialization
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - ../algorithms-data-structures/read-a-bounded-range-from-non-overlapping-byte-segments.md
  - ../data-processing/decode-bounded-fixed-width-ascii-records-across-arbitrary-byte-chunks.md
  - encode-and-decode-one-exact-size-big-endian-binary-record-with-struct.md
---

# Parse and Render a Bounded Intel HEX Image Under a Closed Addressing Profile

## Idea and Problem

Validate a bounded Intel HEX32 text and normalize its data records into sorted non-overlapping byte segments that can be rendered canonically.

Each record carries a byte count, 16-bit address offset, type, payload, and an
8-bit two's-complement checksum. An extended-linear-address record supplies the
upper 16 address bits for following data records. This closed profile accepts
only data (`00`), end-of-file (`01`), and extended linear address (`04`)
records, which is enough for a sparse 32-bit memory image without execution
metadata.

Parsing separates wire-level validation from the resulting memory model.
Records may arrive out of address order, but overlapping bytes fail closed and
adjacent regions are coalesced. Rendering then uses one uppercase, LF-terminated
spelling with fixed-size data records.

## When to Use

Use this profile when both producer and consumer agree that an Intel HEX file
contains only linear-addressed data plus a final EOF marker. It is suitable for
offline inspection, deterministic fixtures, checksummed interchange, and
converting a small sparse image into range-readable segments before any device
operation.

Use a device-specific tool when segment addressing, start addresses, comments,
vendor extensions, memory-region policy, flashing, or a complete Intel HEX
dialect is required. Parsing a checksum-valid image does not establish that its
addresses are safe for a particular target.

## Implementation

```python
from dataclasses import dataclass
from random import Random

_MAX_INTEL_HEX_TEXT = 1_048_576
_MAX_INTEL_HEX_RECORDS = 4_096
_MAX_INTEL_HEX_DATA = 262_144
_MAX_32_BIT_ADDRESS = (1 << 32) - 1
_RENDER_DATA_BYTES = 128
_HEX_DIGITS = frozenset("0123456789abcdefABCDEF")


@dataclass(frozen=True, slots=True)
class MemorySegment:
    address: int
    data: bytes


def _intel_hex_lines(text: str) -> list[str]:
    if type(text) is not str:
        raise TypeError("text must be an exact string")
    if not 1 <= len(text) <= _MAX_INTEL_HEX_TEXT:
        raise ValueError("text length is outside 1..1,048,576")
    try:
        text.encode("ascii")
    except UnicodeEncodeError as error:
        raise ValueError("Intel HEX text must be ASCII") from error

    normalized = text.replace("\r\n", "\n")
    if "\r" in normalized:
        raise ValueError("bare carriage returns are not accepted")
    if normalized.endswith("\n"):
        normalized = normalized[:-1]
    lines = normalized.split("\n")
    if any(not line for line in lines):
        raise ValueError("blank records are not accepted")
    if len(lines) > _MAX_INTEL_HEX_RECORDS:
        raise ValueError("record count exceeds 4,096")
    return lines


def _parse_intel_hex_record(line: str, line_number: int) -> tuple[int, int, bytes]:
    if not line.startswith(":"):
        raise ValueError(f"line {line_number} does not start with a colon")
    encoded = line[1:]
    if len(encoded) < 10 or len(encoded) % 2:
        raise ValueError(f"line {line_number} has an invalid encoded length")
    if any(character not in _HEX_DIGITS for character in encoded):
        raise ValueError(f"line {line_number} contains a non-hex character")

    record = bytes.fromhex(encoded)
    byte_count = record[0]
    if len(record) != byte_count + 5:
        raise ValueError(f"line {line_number} byte count does not match its length")
    if sum(record) & 0xFF:
        raise ValueError(f"line {line_number} checksum is invalid")

    address = int.from_bytes(record[1:3], "big")
    record_type = record[3]
    return address, record_type, record[4:-1]


def parse_intel_hex_image(text: str) -> tuple[MemorySegment, ...]:
    """Parse canonical record shapes from the closed 00/01/04 profile."""
    lines = _intel_hex_lines(text)
    upper_address = 0
    total_data = 0
    pieces: list[MemorySegment] = []

    for index, line in enumerate(lines):
        address, record_type, data = _parse_intel_hex_record(line, index + 1)
        if record_type == 0x00:
            if not data:
                raise ValueError("data records must contain at least one byte")
            if address + len(data) > 1 << 16:
                raise ValueError("a data record crosses its 64-KiB offset window")
            absolute_address = (upper_address << 16) + address
            if absolute_address + len(data) > 1 << 32:
                raise ValueError("data extends beyond the 32-bit address space")
            total_data += len(data)
            if total_data > _MAX_INTEL_HEX_DATA:
                raise ValueError("decoded data exceeds 262,144 bytes")
            pieces.append(MemorySegment(absolute_address, data))
        elif record_type == 0x01:
            if address != 0 or data:
                raise ValueError("EOF must have address zero and no data")
            if index != len(lines) - 1:
                raise ValueError("EOF must be the final record")
        elif record_type == 0x04:
            if address != 0 or len(data) != 2:
                raise ValueError("extended linear address must have address zero and two bytes")
            upper_address = int.from_bytes(data, "big")
        else:
            raise ValueError(f"record type {record_type:02X} is outside the closed profile")

    if _parse_intel_hex_record(lines[-1], len(lines))[1] != 0x01:
        raise ValueError("one final EOF record is required")

    ordered = sorted(pieces, key=lambda segment: segment.address)
    merged: list[MemorySegment] = []
    group_address: int | None = None
    group_end = 0
    group_parts: list[bytes] = []
    for segment in ordered:
        if group_address is None:
            group_address = segment.address
            group_end = segment.address + len(segment.data)
            group_parts = [segment.data]
            continue
        if segment.address < group_end:
            raise ValueError("data records overlap")
        if segment.address == group_end:
            group_parts.append(segment.data)
            group_end += len(segment.data)
            continue
        merged.append(MemorySegment(group_address, b"".join(group_parts)))
        group_address = segment.address
        group_end = segment.address + len(segment.data)
        group_parts = [segment.data]
    if group_address is not None:
        merged.append(MemorySegment(group_address, b"".join(group_parts)))
    return tuple(merged)


def _render_intel_hex_record(address: int, record_type: int, data: bytes) -> str:
    body = (
        bytes((len(data),))
        + address.to_bytes(2, "big")
        + bytes((record_type,))
        + data
    )
    checksum = (-sum(body)) & 0xFF
    return ":" + (body + bytes((checksum,))).hex().upper()


def render_intel_hex_image(segments: tuple[MemorySegment, ...]) -> str:
    """Render already canonical sparse segments under the fixed HEX32 profile."""
    if type(segments) is not tuple:
        raise TypeError("segments must be an exact tuple")

    total_data = 0
    previous_end = -1
    for index, segment in enumerate(segments):
        if type(segment) is not MemorySegment:
            raise TypeError(f"segments[{index}] must be an exact MemorySegment")
        if type(segment.address) is not int:
            raise TypeError(f"segments[{index}].address must be an exact integer")
        if type(segment.data) is not bytes:
            raise TypeError(f"segments[{index}].data must be exact bytes")
        if not segment.data:
            raise ValueError(f"segments[{index}].data must not be empty")
        if not 0 <= segment.address <= _MAX_32_BIT_ADDRESS:
            raise ValueError(f"segments[{index}].address is outside 32-bit range")
        end = segment.address + len(segment.data)
        if end > 1 << 32:
            raise ValueError(f"segments[{index}] extends beyond 32-bit range")
        if segment.address <= previous_end:
            raise ValueError("segments must be sorted, separated, and adjacency-coalesced")
        previous_end = end
        total_data += len(segment.data)
        if total_data > _MAX_INTEL_HEX_DATA:
            raise ValueError("segment data exceeds 262,144 bytes")

    lines: list[str] = []
    current_upper: int | None = None
    for segment in segments:
        consumed = 0
        while consumed < len(segment.data):
            absolute_address = segment.address + consumed
            upper_address = absolute_address >> 16
            offset = absolute_address & 0xFFFF
            if upper_address != current_upper:
                lines.append(
                    _render_intel_hex_record(0, 0x04, upper_address.to_bytes(2, "big"))
                )
                current_upper = upper_address
            chunk_size = min(
                _RENDER_DATA_BYTES,
                len(segment.data) - consumed,
                (1 << 16) - offset,
            )
            lines.append(
                _render_intel_hex_record(
                    offset,
                    0x00,
                    segment.data[consumed : consumed + chunk_size],
                )
            )
            consumed += chunk_size

    lines.append(_render_intel_hex_record(0, 0x01, b""))
    if len(lines) > _MAX_INTEL_HEX_RECORDS:
        raise ValueError("canonical rendering would exceed 4,096 records")
    rendered = "\n".join(lines) + "\n"
    if len(rendered) > _MAX_INTEL_HEX_TEXT:
        raise ValueError("canonical rendering would exceed the text limit")
    return rendered
```

## Example

```python
def independent_record(address: int, record_type: int, data: bytes) -> str:
    fields = [len(data), address >> 8, address & 0xFF, record_type, *data]
    checksum = (-sum(fields)) % 256
    return ":" + "".join(f"{value:02X}" for value in (*fields, checksum))


fixture_text = "\r\n".join(
    (
        independent_record(0, 0x04, b"\x00\x02"),
        independent_record(0, 0x00, b"gamma"),
        independent_record(0, 0x04, b"\x00\x01"),
        independent_record(0xFFF8, 0x00, b"abcdefgh"),
        independent_record(0, 0x04, b"\x00\x02"),
        independent_record(5, 0x00, b"delta"),
        independent_record(0, 0x01, b""),
    )
) + "\r\n"
fixture_segments = parse_intel_hex_image(fixture_text)
assert fixture_segments == (
    MemorySegment(0x1FFF8, b"abcdefghgammadelta"),
)
assert parse_intel_hex_image(render_intel_hex_image(fixture_segments)) == fixture_segments
assert render_intel_hex_image(()) == ":00000001FF\n"
assert parse_intel_hex_image(":00000001ff") == ()

representative = independent_record(0, 0x00, bytes.fromhex("FEEFFFF0"))
eof = independent_record(0, 0x01, b"")
for index in range(1, len(representative)):
    original = representative[index]
    replacement = "0" if original != "0" else "1"
    mutated = representative[:index] + replacement + representative[index + 1 :]
    try:
        parse_intel_hex_image(mutated + "\n" + eof)
    except ValueError:
        pass
    else:
        raise AssertionError("accepted a one-digit record mutation")

invalid_images = (
    representative,
    independent_record(0, 0x02, b"\x10\x00") + "\n" + eof,
    independent_record(0xFFFF, 0x00, b"ab") + "\n" + eof,
    independent_record(0, 0x00, b"ab")
    + "\n"
    + independent_record(1, 0x00, b"cd")
    + "\n"
    + eof,
    eof + "\n" + representative,
)
for invalid in invalid_images:
    try:
        parse_intel_hex_image(invalid)
    except ValueError:
        pass
    else:
        raise AssertionError("accepted an out-of-profile image")

rng = Random(29)
for _ in range(200):
    first_address = rng.randrange(0, 1 << 31)
    first = bytes(rng.randrange(256) for _ in range(rng.randrange(1, 300)))
    gap = rng.randrange(1, 500)
    second_address = first_address + len(first) + gap
    second = bytes(rng.randrange(256) for _ in range(rng.randrange(1, 300)))
    segments = (
        MemorySegment(first_address, first),
        MemorySegment(second_address, second),
    )
    assert parse_intel_hex_image(render_intel_hex_image(segments)) == segments

maximum_data = bytes(range(256)) * 1_024
maximum_segments = (MemorySegment(0, maximum_data),)
maximum_text = render_intel_hex_image(maximum_segments)
maximum_round_trip = parse_intel_hex_image(maximum_text)

assert (
    len(maximum_text) <= _MAX_INTEL_HEX_TEXT,
    maximum_round_trip == maximum_segments,
    len(maximum_round_trip[0].data),
) == (True, True, _MAX_INTEL_HEX_DATA)
```

## Trade-offs and Limitations

Parsing costs `O(text + R log R)` time for `R` data records because the sparse
pieces are sorted before overlap detection and coalescing. Rendering and
checksum work are linear in emitted text. Both directions materialize the
bounded text and decoded bytes, so this is not a streaming implementation.

The parser accepts LF and CRLF record endings, an optional final newline, and
upper- or lowercase hex digits. The renderer deliberately emits uppercase hex,
LF endings, 128-byte data records, an explicit type-04 record for each used
upper address, and a final newline. Valid files with different record grouping
therefore normalize to the same spelling.

Only record types `00`, `01`, and `04` are supported. Segment-address records,
start-address records, comments, blank lines, zero-length data records,
overlaps, and data records that cross a 64-KiB offset window are rejected. A
checksum detects many accidental record errors, but collisions are possible
and it is not authentication. A valid memory image is not authorization to
program any device or address.

## Related Snippets

<!-- catalog:related:start -->
- [Read a Bounded Range from Non-Overlapping Byte Segments](../algorithms-data-structures/read-a-bounded-range-from-non-overlapping-byte-segments.md)
- [Decode Bounded Fixed-Width ASCII Records Across Arbitrary Byte Chunks](../data-processing/decode-bounded-fixed-width-ascii-records-across-arbitrary-byte-chunks.md)
- [Encode and Decode One Exact-Size Big-Endian Binary Record with struct.Struct](encode-and-decode-one-exact-size-big-endian-binary-record-with-struct.md)
<!-- catalog:related:end -->
