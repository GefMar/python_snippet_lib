---
title: "Inspect Bounded PNG Critical-Chunk Structure without Decompressing Pixels"
snippet_type: recipe
use_cases:
  - interoperability
  - parsing
  - security
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - encode-and-decode-one-exact-size-big-endian-binary-record-with-struct.md
  - parse-and-render-a-bounded-intel-hex-image-under-a-closed-addressing-profile.md
  - ../storage-databases/combine-zlib-crc-32-values-for-concatenated-byte-segments.md
---

# Inspect Bounded PNG Critical-Chunk Structure without Decompressing Pixels

## Idea and Problem

Inspect the critical-chunk structure and checksums of a bounded PNG byte string without constructing or decompressing a decoded pixel buffer.

[PNG Third Edition](https://www.w3.org/TR/png-3/) stores a fixed signature
followed by length-framed chunks. Every chunk has a four-letter type, data
bytes, and a CRC-32 over the type and data. A useful preflight boundary can
validate that framing, the `IHDR` fields, palette placement, consecutive
`IDAT` chunks, and the final `IEND` before a full image decoder sees the input.

The returned inventory deliberately contains only each chunk's type, declared
data length, and byte offset. It does not retain arbitrary ancillary payloads.
Unknown critical chunks fail closed; unknown ancillary chunks remain visible
in the inventory so a later policy layer can decide whether they matter.

## When to Use

Use this recipe for bounded upload preflight, deterministic binary fixtures,
or offline structural inspection when pixel decoding is a separate trusted
step. It catches truncated framing, corrupt CRCs, impossible `IHDR` profiles,
and several critical ordering errors with predictable resource limits.

Use a maintained PNG decoder when the application must display or transform
the image. A structurally accepted byte string can still contain an invalid
zlib stream, invalid scanline filters, impossible pixel data, or meaningful
ancillary chunks that this inspector intentionally does not interpret.

## Implementation

```python
from binascii import crc32
from dataclasses import dataclass

_PNG_SIGNATURE = b"\x89PNG\r\n\x1a\n"
_MAX_PNG_BYTES = 4 * 1024 * 1024
_MAX_PNG_CHUNKS = 256
_MAX_PNG_CHUNK_DATA = 256 * 1024
_MAX_PNG_DIMENSION = 32_768
_MAX_PNG_PIXELS = 100_000_000
_PNG_DEPTHS_BY_COLOR_TYPE = {
    0: frozenset((1, 2, 4, 8, 16)),
    2: frozenset((8, 16)),
    3: frozenset((1, 2, 4, 8)),
    4: frozenset((8, 16)),
    6: frozenset((8, 16)),
}
_KNOWN_CRITICAL_CHUNKS = frozenset((b"IHDR", b"PLTE", b"IDAT", b"IEND"))


class PngStructureError(ValueError):
    """Raised when bytes violate the bounded PNG structural profile."""


@dataclass(frozen=True, slots=True)
class PngChunkInfo:
    chunk_type: str
    data_length: int
    offset: int


@dataclass(frozen=True, slots=True)
class PngStructure:
    width: int
    height: int
    bit_depth: int
    color_type: int
    interlace: int
    chunks: tuple[PngChunkInfo, ...]


def _is_ascii_chunk_type(chunk_type: bytes) -> bool:
    return len(chunk_type) == 4 and all(
        65 <= character <= 90 or 97 <= character <= 122
        for character in chunk_type
    )


def inspect_png_critical_structure(encoded: bytes) -> PngStructure:
    """Return metadata after checking a bounded PNG critical structure."""
    if type(encoded) is not bytes:
        raise TypeError("encoded must be exact immutable bytes")
    if not 57 <= len(encoded) <= _MAX_PNG_BYTES:
        raise PngStructureError("PNG length is outside 57..4,194,304 bytes")
    if not encoded.startswith(_PNG_SIGNATURE):
        raise PngStructureError("PNG signature is invalid")

    offset = len(_PNG_SIGNATURE)
    chunks: list[PngChunkInfo] = []
    width = height = bit_depth = color_type = interlace = None
    saw_palette = False
    saw_image_data = False
    image_data_finished = False

    while offset < len(encoded):
        if len(chunks) == _MAX_PNG_CHUNKS:
            raise PngStructureError("chunk count exceeds 256")
        if offset + 12 > len(encoded):
            raise PngStructureError("PNG ends inside a chunk frame")

        chunk_offset = offset
        data_length = int.from_bytes(encoded[offset : offset + 4], "big")
        if data_length > _MAX_PNG_CHUNK_DATA:
            raise PngStructureError("chunk data exceeds 262,144 bytes")

        chunk_type = encoded[offset + 4 : offset + 8]
        if not _is_ascii_chunk_type(chunk_type):
            raise PngStructureError("chunk type must contain four ASCII letters")

        data_start = offset + 8
        data_end = data_start + data_length
        chunk_end = data_end + 4
        if chunk_end > len(encoded):
            raise PngStructureError("PNG ends inside a chunk frame")

        data = encoded[data_start:data_end]
        expected_crc = int.from_bytes(encoded[data_end:chunk_end], "big")
        actual_crc = crc32(data, crc32(chunk_type)) & 0xFFFF_FFFF
        if actual_crc != expected_crc:
            raise PngStructureError("chunk CRC does not match its type and data")

        is_critical = chunk_type[0] & 0x20 == 0
        if is_critical and chunk_type not in _KNOWN_CRITICAL_CHUNKS:
            raise PngStructureError("unknown critical chunk is not supported")

        chunks.append(
            PngChunkInfo(chunk_type.decode("ascii"), data_length, chunk_offset)
        )

        if chunk_type == b"IHDR":
            if len(chunks) != 1 or width is not None:
                raise PngStructureError("IHDR must occur exactly once and first")
            if data_length != 13:
                raise PngStructureError("IHDR data must contain exactly 13 bytes")

            width = int.from_bytes(data[0:4], "big")
            height = int.from_bytes(data[4:8], "big")
            bit_depth = data[8]
            color_type = data[9]
            compression, filtering, interlace = data[10:13]
            if not 1 <= width <= _MAX_PNG_DIMENSION:
                raise PngStructureError("width is outside the local 1..32,768 cap")
            if not 1 <= height <= _MAX_PNG_DIMENSION:
                raise PngStructureError("height is outside the local 1..32,768 cap")
            if width * height > _MAX_PNG_PIXELS:
                raise PngStructureError("pixel count exceeds the local 100,000,000 cap")
            if bit_depth not in _PNG_DEPTHS_BY_COLOR_TYPE.get(color_type, ()):
                raise PngStructureError("bit depth is invalid for the color type")
            if compression != 0 or filtering != 0:
                raise PngStructureError("compression and filter methods must be zero")
            if interlace not in (0, 1):
                raise PngStructureError("interlace method must be zero or one")
        elif width is None:
            raise PngStructureError("IHDR must be the first chunk")
        elif chunk_type == b"PLTE":
            if saw_palette:
                raise PngStructureError("PLTE must not be repeated")
            if saw_image_data:
                raise PngStructureError("PLTE must precede IDAT")
            if color_type in (0, 4):
                raise PngStructureError("PLTE is forbidden for this color type")
            if not 3 <= data_length <= 768 or data_length % 3:
                raise PngStructureError("PLTE must contain 1..256 RGB entries")
            palette_entries = data_length // 3
            if color_type == 3 and palette_entries > 1 << bit_depth:
                raise PngStructureError("PLTE has too many entries for indexed depth")
            saw_palette = True
        elif chunk_type == b"IDAT":
            if image_data_finished:
                raise PngStructureError("IDAT chunks must be consecutive")
            if color_type == 3 and not saw_palette:
                raise PngStructureError("indexed-color PNG requires PLTE before IDAT")
            saw_image_data = True
        else:
            if saw_image_data:
                image_data_finished = True
            if chunk_type == b"IEND":
                if data_length != 0:
                    raise PngStructureError("IEND data must be empty")
                if not saw_image_data:
                    raise PngStructureError("at least one IDAT chunk is required")
                if chunk_end != len(encoded):
                    raise PngStructureError("IEND must be the final chunk")
                return PngStructure(
                    width,
                    height,
                    bit_depth,
                    color_type,
                    interlace,
                    tuple(chunks),
                )

        offset = chunk_end

    raise PngStructureError("PNG is missing its final IEND chunk")
```

## Example

```python
def png_chunk(chunk_type: bytes, data: bytes) -> bytes:
    checksum = crc32(data, crc32(chunk_type)) & 0xFFFF_FFFF
    return (
        len(data).to_bytes(4, "big")
        + chunk_type
        + data
        + checksum.to_bytes(4, "big")
    )


def png_header(
    *,
    width: int = 1,
    height: int = 1,
    bit_depth: int = 8,
    color_type: int = 6,
    compression: int = 0,
    filtering: int = 0,
    interlace: int = 0,
) -> bytes:
    return (
        width.to_bytes(4, "big")
        + height.to_bytes(4, "big")
        + bytes((bit_depth, color_type, compression, filtering, interlace))
    )


def make_png(
    header: bytes,
    *middle_chunks: tuple[bytes, bytes],
) -> bytes:
    return _PNG_SIGNATURE + b"".join(
        png_chunk(chunk_type, data)
        for chunk_type, data in ((b"IHDR", header), *middle_chunks, (b"IEND", b""))
    )


minimum = make_png(png_header(), (b"IDAT", b""))
assert len(minimum) == 57
assert inspect_png_critical_structure(minimum) == PngStructure(
    1,
    1,
    8,
    6,
    0,
    (
        PngChunkInfo("IHDR", 13, 8),
        PngChunkInfo("IDAT", 0, 33),
        PngChunkInfo("IEND", 0, 45),
    ),
)

indexed = make_png(
    png_header(width=5, height=7, bit_depth=2, color_type=3, interlace=1),
    (b"PLTE", bytes((0, 0, 0, 255, 255, 255))),
    (b"IDAT", b""),
    (b"IDAT", b"not decoded"),
    (b"vpaa", b"opaque ancillary data"),
)
indexed_structure = inspect_png_critical_structure(indexed)
assert (indexed_structure.width, indexed_structure.height) == (5, 7)
assert tuple(chunk.chunk_type for chunk in indexed_structure.chunks) == (
    "IHDR",
    "PLTE",
    "IDAT",
    "IDAT",
    "vpaa",
    "IEND",
)

valid_depths = {
    0: (1, 2, 4, 8, 16),
    2: (8, 16),
    3: (1, 2, 4, 8),
    4: (8, 16),
    6: (8, 16),
}
for valid_color_type, depths in valid_depths.items():
    for valid_depth in depths:
        middle = []
        if valid_color_type == 3:
            middle.append((b"PLTE", b"\x00\x00\x00"))
        middle.append((b"IDAT", b""))
        candidate = make_png(
            png_header(bit_depth=valid_depth, color_type=valid_color_type),
            *middle,
        )
        assert inspect_png_critical_structure(candidate).bit_depth == valid_depth

bounded_dimensions = make_png(
    png_header(width=32_768, height=3_051),
    (b"IDAT", b""),
)
assert inspect_png_critical_structure(bounded_dimensions).width == 32_768

maximum_palette = make_png(
    png_header(bit_depth=8, color_type=3),
    (b"PLTE", bytes(range(256)) * 3),
    (b"IDAT", b""),
)
assert inspect_png_critical_structure(maximum_palette).chunks[1].data_length == 768

maximum_chunk = make_png(
    png_header(),
    (b"wiDe", b"x" * _MAX_PNG_CHUNK_DATA),
    (b"IDAT", b""),
)
assert inspect_png_critical_structure(maximum_chunk).chunks[1].data_length == (
    _MAX_PNG_CHUNK_DATA
)

maximum_chunk_count = make_png(
    png_header(),
    *((b"saFe", b"") for _ in range(_MAX_PNG_CHUNKS - 3)),
    (b"IDAT", b""),
)
assert len(inspect_png_critical_structure(maximum_chunk_count).chunks) == 256

padding_chunk_count = 16
padding_data_bytes = (
    _MAX_PNG_BYTES - len(minimum) - 12 * padding_chunk_count
)
full_padding_chunks, final_padding_bytes = divmod(
    padding_data_bytes,
    _MAX_PNG_CHUNK_DATA,
)
assert (full_padding_chunks, final_padding_bytes) == (15, 261_895)
padding_chunks = [(b"fiLl", b"x" * _MAX_PNG_CHUNK_DATA)] * full_padding_chunks
padding_chunks.append((b"fiLl", b"x" * final_padding_bytes))
maximum_png = make_png(png_header(), *padding_chunks, (b"IDAT", b""))
assert len(maximum_png) == _MAX_PNG_BYTES
assert inspect_png_critical_structure(maximum_png).width == 1


def is_rejected(candidate: object) -> bool:
    try:
        inspect_png_critical_structure(candidate)  # type: ignore[arg-type]
    except (TypeError, ValueError):
        return True
    return False


bad_crc = bytearray(minimum)
bad_crc[-1] ^= 1
rejected = (
    b"not PNG",
    b"\x88" + minimum[1:],
    bytes(bad_crc),
    minimum[:-1],
    _PNG_SIGNATURE
    + png_chunk(b"text", b"")
    + png_chunk(b"IHDR", png_header())
    + png_chunk(b"IDAT", b"")
    + png_chunk(b"IEND", b""),
    make_png(png_header(), (b"IHDR", png_header()), (b"IDAT", b"")),
    make_png(png_header()[:-1], (b"IDAT", b"")),
    make_png(png_header(bit_depth=4, color_type=6), (b"IDAT", b"")),
    make_png(png_header(compression=1), (b"IDAT", b"")),
    make_png(png_header(filtering=1), (b"IDAT", b"")),
    make_png(png_header(interlace=2), (b"IDAT", b"")),
    make_png(png_header(color_type=3), (b"IDAT", b"")),
    make_png(png_header(color_type=0), (b"PLTE", b"\x00\x00\x00"), (b"IDAT", b"")),
    make_png(
        png_header(bit_depth=1, color_type=3),
        (b"PLTE", b"\x00" * 9),
        (b"IDAT", b""),
    ),
    make_png(
        png_header(color_type=3),
        (b"PLTE", b"\x00\x00\x00"),
        (b"PLTE", b"\xff\xff\xff"),
        (b"IDAT", b""),
    ),
    make_png(
        png_header(color_type=2),
        (b"IDAT", b""),
        (b"PLTE", b"\x00\x00\x00"),
    ),
    make_png(png_header(width=32_769), (b"IDAT", b"")),
    make_png(png_header(width=32_768, height=3_052), (b"IDAT", b"")),
    make_png(png_header(), (b"ABCD", b""), (b"IDAT", b"")),
    make_png(png_header(), (b"te1t", b""), (b"IDAT", b"")),
    make_png(png_header(), (b"IDAT", b""), (b"text", b"x"), (b"IDAT", b"")),
    make_png(png_header(), (b"IDAT", b""), (b"wide", b"x" * (256 * 1024 + 1))),
    make_png(
        png_header(),
        *((b"saFe", b"") for _ in range(_MAX_PNG_CHUNKS - 2)),
        (b"IDAT", b""),
    ),
    _PNG_SIGNATURE
    + png_chunk(b"IHDR", png_header())
    + png_chunk(b"IDAT", b"")
    + png_chunk(b"text", b"x"),
    _PNG_SIGNATURE
    + png_chunk(b"IHDR", png_header())
    + png_chunk(b"IDAT", b"")
    + png_chunk(b"IEND", b"x"),
    minimum + b"\x00",
    maximum_png + b"\x00",
)
assert all(is_rejected(candidate) for candidate in rejected)
assert is_rejected(bytearray(minimum))
```

## Trade-offs and Limitations

The complete input is materialized and capped at 4 MiB. At most 256 chunks are
visited once, each CRC scans at most 256 KiB, and the metadata inventory uses
`O(number of chunks)` storage. A temporary slice copies one bounded chunk
payload at a time. The local dimension and pixel-count caps bound later work;
they are not limits imposed by the PNG specification.

This is a critical-structure inspector, not a full PNG conformance checker or
decoder. In particular, an empty `IDAT` is structurally accepted. The code does
not decompress zlib data, reconstruct scanlines, validate filter bytes or pixel
counts, interpret color values, enforce every ancillary-chunk ordering rule,
or implement APNG. It follows decoder-oriented forward compatibility by not
rejecting an unknown ancillary type solely because its third letter is
lowercase, even though that type bit is reserved by the current PNG version.

CRC-32 detects accidental corruption but is not an authenticity mechanism.
Perform content authentication and application-specific image policy outside
this parser, and pass accepted bytes to a maintained decoder before using any
pixels.

## Related Snippets

<!-- catalog:related:start -->
- [Encode and Decode One Exact-Size Big-Endian Binary Record with struct.Struct](encode-and-decode-one-exact-size-big-endian-binary-record-with-struct.md)
- [Parse and Render a Bounded Intel HEX Image Under a Closed Addressing Profile](parse-and-render-a-bounded-intel-hex-image-under-a-closed-addressing-profile.md)
- [Combine zlib CRC-32 Values for Concatenated Byte Segments](../storage-databases/combine-zlib-crc-32-values-for-concatenated-byte-segments.md)
<!-- catalog:related:end -->
