---
title: "Extract Bounded Uncompressed PCM Frames from an In-Memory WAV"
snippet_type: recipe
use_cases:
  - interoperability
  - parsing
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - encode-and-decode-one-exact-size-big-endian-binary-record-with-struct.md
  - ../data-processing/extract-a-finite-2d-bounding-box-from-bounded-wkb.md
  - ../networking-protocols/read-and-write-size-capped-varint-frames.md
---

# Extract Bounded Uncompressed PCM Frames from an In-Memory WAV

## Idea and Problem

Extract metadata and canonical raw little-endian PCM frame bytes from one bounded in-memory WAV using the standard library's format parser.

The helper validates channel, sample-width, sample-rate, frame-count, and exact
payload budgets before one bounded read. The returned frames remain interleaved
encoded bytes: they are not decoded numeric samples.

## When to Use

Use this for a small generated fixture, protocol attachment, test artifact, or
offline transformation that accepts only mono or stereo uncompressed PCM.
Choose tighter byte, rate, and frame limits when the surrounding format has a
smaller profile.

Use an audio library when you need sample arrays, channel-layout interpretation,
format conversion, resampling, normalization, playback, streaming, or
compressed codecs. Treat format acceptance separately from trusting what an
audio recording contains.

## Implementation

```python
import sys
import wave
from dataclasses import dataclass
from io import BytesIO

_MAX_WAV_BYTES = 1_048_576
_MAX_FRAMES = 262_144
_MAX_FRAME_BYTES = 1_048_576
_MAX_FRAME_RATE = 192_000


class PcmWavError(ValueError):
    """Raised when input is outside the bounded PCM WAV profile."""


@dataclass(frozen=True, slots=True)
class PcmWav:
    channels: int
    sample_width: int
    frame_rate: int
    frame_count: int
    frames: bytes


def _reverse_sample_bytes(
    frames: bytes,
    sample_width: int,
) -> bytes:
    return b"".join(
        frames[offset : offset + sample_width][::-1]
        for offset in range(0, len(frames), sample_width)
    )


def extract_pcm_wav(data: bytes) -> PcmWav:
    """Return bounded PCM metadata and canonical little-endian frame bytes."""
    if type(data) is not bytes:
        raise TypeError("data must be exact bytes")
    if len(data) > _MAX_WAV_BYTES:
        raise PcmWavError("WAV input exceeds the byte limit")

    try:
        with wave.open(BytesIO(data), "rb") as reader:
            channels = reader.getnchannels()
            sample_width = reader.getsampwidth()
            frame_rate = reader.getframerate()
            frame_count = reader.getnframes()

            if reader.getcomptype() != "NONE":
                raise PcmWavError(
                    "WAV must contain uncompressed PCM"
                )
            if channels not in {1, 2}:
                raise PcmWavError(
                    "channel count is outside the supported profile"
                )
            if not 1 <= sample_width <= 4:
                raise PcmWavError(
                    "sample width is outside the supported profile"
                )
            if not 1 <= frame_rate <= _MAX_FRAME_RATE:
                raise PcmWavError(
                    "frame rate is outside the supported profile"
                )
            if not 0 <= frame_count <= _MAX_FRAMES:
                raise PcmWavError(
                    "frame count exceeds the supported profile"
                )

            expected_bytes = channels * sample_width * frame_count
            if expected_bytes > _MAX_FRAME_BYTES:
                raise PcmWavError(
                    "PCM payload exceeds the byte limit"
                )
            frames = reader.readframes(frame_count + 1)
    except PcmWavError:
        raise
    except (wave.Error, EOFError, RuntimeError, ValueError):
        raise PcmWavError(
            "data is not a supported bounded PCM WAV"
        ) from None

    if len(frames) != expected_bytes:
        raise PcmWavError(
            "PCM frame data is truncated or partial"
        )
    if sys.byteorder == "big" and sample_width > 1:
        frames = _reverse_sample_bytes(frames, sample_width)

    return PcmWav(
        channels=channels,
        sample_width=sample_width,
        frame_rate=frame_rate,
        frame_count=frame_count,
        frames=frames,
    )
```

## Example

```python
def build_pcm_wav(
    *,
    channels: int,
    sample_width: int,
    frame_rate: int,
    frames: bytes,
) -> bytes:
    output = BytesIO()
    with wave.open(output, "wb") as writer:
        writer.setnchannels(channels)
        writer.setsampwidth(sample_width)
        writer.setframerate(frame_rate)
        writer.writeframes(frames)
    return output.getvalue()


samples = b"\x00\x80\x00\x00\xff\x7f"
payload = build_pcm_wav(
    channels=1,
    sample_width=2,
    frame_rate=8_000,
    frames=samples,
)
decoded = extract_pcm_wav(payload)
with_trailing_bytes = extract_pcm_wav(
    payload + b"ignored trailing bytes"
)
empty = extract_pcm_wav(
    build_pcm_wav(
        channels=1,
        sample_width=1,
        frame_rate=8_000,
        frames=b"",
    )
)

partial = bytearray(payload[:-1])
partial[4:8] = (len(partial) - 8).to_bytes(4, "little")
data_offset = partial.index(b"data")
partial[data_offset + 4 : data_offset + 8] = (3).to_bytes(
    4,
    "little",
)

malformed_chunk = bytearray(payload)
malformed_chunk[16:20] = (18).to_bytes(4, "little")

invalid_payloads = (
    bytes(partial),
    bytes(malformed_chunk),
    build_pcm_wav(
        channels=3,
        sample_width=2,
        frame_rate=8_000,
        frames=b"\x00" * 6,
    ),
    payload + b"\x00" * _MAX_WAV_BYTES,
)
rejected = 0
for invalid in invalid_payloads:
    try:
        extract_pcm_wav(invalid)
    except PcmWavError:
        rejected += 1

assert (
    decoded
    == PcmWav(
        channels=1,
        sample_width=2,
        frame_rate=8_000,
        frame_count=3,
        frames=samples,
    )
    and with_trailing_bytes == decoded
    and empty.frame_count == 0
    and empty.frames == b""
    and _reverse_sample_bytes(
        b"\x01\x02\x03\x04",
        2,
    )
    == b"\x02\x01\x04\x03"
    and rejected == 4
)
```

## Trade-offs and Limitations

Let `B` be the captured input and `P` the returned PCM payload. The
bounded parser, input buffer, and immutable result use `O(B + P)` time and
memory. On big-endian hosts, `wave` exposes multi-byte samples in native
order, so the helper reverses each sample-width chunk back to canonical WAV
little-endian order.

`readframes(frame_count + 1)` catches truncation and a trailing partial
frame by requiring exactly `channels * sample_width * frame_count` bytes.
The frame count is still the parser's interpretation of the data chunk, not an
independently authenticated fact. PCM in WAVE_FORMAT_EXTENSIBLE is accepted
only when the tested standard library reports compression type `NONE`.

Unknown ancillary chunks and bytes after the data chunk are deliberately
ignored inside the bounded capture. The function therefore does not validate
the whole RIFF container, infer a channel mask, decode sample values, verify
audio semantics, or provide a streaming interface.

## Related Snippets

<!-- catalog:related:start -->
- [Encode and Decode One Exact-Size Big-Endian Binary Record with struct.Struct](encode-and-decode-one-exact-size-big-endian-binary-record-with-struct.md)
- [Extract a Finite 2D Bounding Box from Bounded WKB](../data-processing/extract-a-finite-2d-bounding-box-from-bounded-wkb.md)
- [Read and Write Size-Capped Varint Frames](../networking-protocols/read-and-write-size-capped-varint-frames.md)
<!-- catalog:related:end -->
