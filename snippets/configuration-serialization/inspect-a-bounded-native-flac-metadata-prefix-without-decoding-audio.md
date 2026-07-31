---
title: "Inspect a Bounded Native FLAC Metadata Prefix without Decoding Audio"
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
  - extract-bounded-uncompressed-pcm-frames-from-an-in-memory-wav.md
  - inspect-bounded-png-critical-chunk-structure-without-decompressing-pixels.md
  - ../networking-protocols/parse-one-checksum-valid-bounded-ogg-page.md
---

# Inspect a Bounded Native FLAC Metadata Prefix without Decoding Audio

## Idea and Problem

Locate and inventory a complete native FLAC metadata prefix inside a bounded byte window without interpreting metadata payloads or decoding audio frames.

[RFC 9639](https://www.rfc-editor.org/rfc/rfc9639) places the `fLaC`
marker before one or more length-framed metadata blocks. Each four-byte block
header carries a last-block flag, a seven-bit type, and a 24-bit big-endian
payload length. The first block must be the unique 34-byte Streaminfo block;
the byte after the block marked last is the first possible audio-frame byte.

The inspector returns immutable copies of the raw block payloads and that
audio offset. Reserved block types remain visible for forward compatibility,
while the forbidden type `127`, incomplete framing, repeated Streaminfo, and a
prefix that does not finish inside the caller's byte window fail closed.

## When to Use

Use this recipe for bounded upload preflight, deterministic fixtures, metadata
routing, or an adapter that needs to locate native FLAC audio before handing
the stream to a maintained decoder. The input window may contain bytes after
the metadata prefix; they are deliberately ignored once the last block is
framed.

Use a FLAC library when metadata contents, frame checksums, decoded samples,
or whole-stream validity matter. Use an Ogg demultiplexer for FLAC carried in
Ogg, whose identification and metadata packets have different outer framing.

## Implementation

```python
from dataclasses import dataclass

_FLAC_MARKER = b"fLaC"
_FLAC_METADATA_HEADER_BYTES = 4
_FLAC_STREAMINFO_TYPE = 0
_FLAC_STREAMINFO_BYTES = 34
_FLAC_FORBIDDEN_BLOCK_TYPE = 127
_MIN_FLAC_WINDOW_BYTES = 42
_MAX_FLAC_WINDOW_BYTES = 1 * 1_024 * 1_024
_MAX_FLAC_METADATA_BLOCKS = 256


class FlacMetadataPrefixError(ValueError):
    """Raised when a byte window does not contain the supported FLAC prefix."""


@dataclass(frozen=True, slots=True)
class FlacMetadataBlock:
    block_type: int
    is_last: bool
    payload: bytes


@dataclass(frozen=True, slots=True)
class FlacMetadataPrefix:
    blocks: tuple[FlacMetadataBlock, ...]
    audio_offset: int


def inspect_flac_metadata_prefix(window: bytes) -> FlacMetadataPrefix:
    """Inspect one complete native FLAC metadata prefix in a bounded window."""
    if type(window) is not bytes:
        raise TypeError("window must be exact immutable bytes")
    if not _MIN_FLAC_WINDOW_BYTES <= len(window) <= _MAX_FLAC_WINDOW_BYTES:
        raise FlacMetadataPrefixError("window length is outside 42..1,048,576 bytes")
    if not window.startswith(_FLAC_MARKER):
        raise FlacMetadataPrefixError("native FLAC marker must be fLaC")

    blocks: list[FlacMetadataBlock] = []
    offset = len(_FLAC_MARKER)

    while True:
        if len(blocks) == _MAX_FLAC_METADATA_BLOCKS:
            raise FlacMetadataPrefixError("metadata block count exceeds 256")
        if offset + _FLAC_METADATA_HEADER_BYTES > len(window):
            raise FlacMetadataPrefixError("window ends inside a metadata block header")

        first_header_byte = window[offset]
        is_last = bool(first_header_byte & 0x80)
        block_type = first_header_byte & 0x7F
        payload_length = int.from_bytes(window[offset + 1 : offset + 4], "big")

        if block_type == _FLAC_FORBIDDEN_BLOCK_TYPE:
            raise FlacMetadataPrefixError("metadata block type 127 is forbidden")

        payload_start = offset + _FLAC_METADATA_HEADER_BYTES
        payload_end = payload_start + payload_length
        if payload_end > len(window):
            raise FlacMetadataPrefixError("window ends inside a metadata block payload")

        if not blocks:
            if block_type != _FLAC_STREAMINFO_TYPE:
                raise FlacMetadataPrefixError("the first metadata block must be Streaminfo")
            if payload_length != _FLAC_STREAMINFO_BYTES:
                raise FlacMetadataPrefixError(
                    "the Streaminfo payload must contain exactly 34 bytes"
                )
        elif block_type == _FLAC_STREAMINFO_TYPE:
            raise FlacMetadataPrefixError("Streaminfo must not be repeated")

        blocks.append(
            FlacMetadataBlock(
                block_type=block_type,
                is_last=is_last,
                payload=window[payload_start:payload_end],
            )
        )
        offset = payload_end

        if is_last:
            return FlacMetadataPrefix(tuple(blocks), offset)
```

## Example

```python
def make_streaminfo() -> bytes:
    """Build plausible Streaminfo bytes independently of the inspector."""
    sample_rate = 44_100
    channels_minus_one = 1
    bits_per_sample_minus_one = 15
    total_samples = 1
    packed_audio_fields = (
        (sample_rate << 44)
        | (channels_minus_one << 41)
        | (bits_per_sample_minus_one << 36)
        | total_samples
    )
    return (
        (16).to_bytes(2, "big")
        + (16).to_bytes(2, "big")
        + bytes(6)
        + packed_audio_fields.to_bytes(8, "big")
        + bytes(16)
    )


def make_metadata_block(
    block_type: int,
    payload: bytes,
    *,
    is_last: bool,
) -> bytes:
    """Build a raw metadata block without using inspector internals."""
    if not 0 <= block_type <= 127:
        raise ValueError("fixture block type must fit seven bits")
    if len(payload) >= 1 << 24:
        raise ValueError("fixture payload does not fit the wire length")
    flags_and_type = (0x80 if is_last else 0) | block_type
    return bytes((flags_and_type,)) + len(payload).to_bytes(3, "big") + payload


def make_window(*blocks: bytes, following: bytes = b"") -> bytes:
    return b"fLaC" + b"".join(blocks) + following


def rejected_as(candidate: object, expected_error: type[Exception]) -> bool:
    try:
        inspect_flac_metadata_prefix(candidate)  # type: ignore[arg-type]
    except expected_error:
        return True
    return False


streaminfo = make_streaminfo()
assert len(streaminfo) == 34

minimal_prefix = make_window(
    make_metadata_block(0, streaminfo, is_last=True),
)
minimal_window = minimal_prefix + b"\xff\xf8audio-frame-prefix"
minimal = inspect_flac_metadata_prefix(minimal_window)
assert minimal == FlacMetadataPrefix(
    blocks=(FlacMetadataBlock(0, True, streaminfo),),
    audio_offset=42,
)
assert minimal_window[minimal.audio_offset :] == b"\xff\xf8audio-frame-prefix"

multi_window = make_window(
    make_metadata_block(0, streaminfo, is_last=False),
    make_metadata_block(1, bytes(3), is_last=False),
    make_metadata_block(2, b"TESTpayload", is_last=True),
    following=b"ignored-audio-window",
)
multi = inspect_flac_metadata_prefix(multi_window)
assert tuple(block.block_type for block in multi.blocks) == (0, 1, 2)
assert tuple(block.is_last for block in multi.blocks) == (False, False, True)
assert multi.blocks[2].payload == b"TESTpayload"
assert multi_window[multi.audio_offset :] == b"ignored-audio-window"

reserved_window = make_window(
    make_metadata_block(0, streaminfo, is_last=False),
    make_metadata_block(126, b"future", is_last=True),
)
reserved = inspect_flac_metadata_prefix(reserved_window)
assert reserved.blocks[-1] == FlacMetadataBlock(126, True, b"future")

maximum_window = minimal_prefix + bytes(_MAX_FLAC_WINDOW_BYTES - len(minimal_prefix))
assert len(maximum_window) == 1_048_576
assert inspect_flac_metadata_prefix(maximum_window).audio_offset == 42

maximum_block_count = make_window(
    make_metadata_block(0, streaminfo, is_last=False),
    *(make_metadata_block(1, b"", is_last=False) for _ in range(254)),
    make_metadata_block(1, b"", is_last=True),
)
assert len(inspect_flac_metadata_prefix(maximum_block_count).blocks) == 256

over_block_count = make_window(
    make_metadata_block(0, streaminfo, is_last=False),
    *(make_metadata_block(1, b"", is_last=False) for _ in range(255)),
    make_metadata_block(1, b"", is_last=True),
)
repeated_streaminfo = make_window(
    make_metadata_block(0, streaminfo, is_last=False),
    make_metadata_block(0, streaminfo, is_last=True),
)
forbidden_type = make_window(
    make_metadata_block(0, streaminfo, is_last=False),
    make_metadata_block(127, b"", is_last=True),
)
missing_last = make_window(
    make_metadata_block(0, streaminfo, is_last=False),
)
truncated_header = missing_last + b"\x81\x00"
truncated_payload = missing_last + b"\x81\x00\x00\x05abcd"
wrong_first_type = make_window(
    make_metadata_block(1, bytes(34), is_last=True),
)
wrong_streaminfo_length = make_window(
    make_metadata_block(0, bytes(33), is_last=True),
    following=b"x",
)

invalid_cases = (
    (bytearray(minimal_window), TypeError),
    (bytes(41), FlacMetadataPrefixError),
    (b"FAIL" + minimal_window[4:], FlacMetadataPrefixError),
    (wrong_first_type, FlacMetadataPrefixError),
    (wrong_streaminfo_length, FlacMetadataPrefixError),
    (repeated_streaminfo, FlacMetadataPrefixError),
    (forbidden_type, FlacMetadataPrefixError),
    (missing_last, FlacMetadataPrefixError),
    (truncated_header, FlacMetadataPrefixError),
    (truncated_payload, FlacMetadataPrefixError),
    (over_block_count, FlacMetadataPrefixError),
    (maximum_window + b"x", FlacMetadataPrefixError),
)
rejected = sum(
    rejected_as(candidate, expected_error) for candidate, expected_error in invalid_cases
)

assert rejected == len(invalid_cases) == 12 and minimal.audio_offset == 42
```

## Trade-offs and Limitations

Inspection takes `O(M + B)` time and memory for `M` metadata payload bytes and
`B` blocks because every returned payload is copied. The one-mebibyte window
and 256-block limits are local safety policy, not FLAC wire limits; one block's
24-bit length alone can describe almost 16 MiB. Bytes after the last metadata
block are neither copied nor validated.

The function checks only metadata-envelope invariants. It does not validate
the fields inside Streaminfo or any other metadata payload, require an audio
frame after the prefix, verify frame CRCs or the decoded-audio MD5, follow
picture URIs, or establish that the containing FLAC stream is valid.

Only a native stream beginning directly with `fLaC` is accepted. ID3
preambles, FLAC-in-Ogg identification packets, incremental reads, and metadata
prefixes larger than the caller's bounded window require a different layer.

## Related Snippets

<!-- catalog:related:start -->
- [Extract Bounded Uncompressed PCM Frames from an In-Memory WAV](extract-bounded-uncompressed-pcm-frames-from-an-in-memory-wav.md)
- [Inspect Bounded PNG Critical-Chunk Structure without Decompressing Pixels](inspect-bounded-png-critical-chunk-structure-without-decompressing-pixels.md)
- [Parse One Checksum-Valid Bounded Ogg Page](../networking-protocols/parse-one-checksum-valid-bounded-ogg-page.md)
<!-- catalog:related:end -->
