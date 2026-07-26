---
title: "Read and Write Size-Capped Varint Frames"
snippet_type: recipe
use_cases:
  - networking
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../python-language/read-fixed-size-blocks-with-iter-sentinel.md
  - ../data-processing/batch-items-by-estimated-byte-size.md
---

# Read and Write Size-Capped Varint Frames

## Idea and Problem

Frame byte payloads on a blocking stream with a canonical unsigned LEB128 length while rejecting oversized declarations before reading their bodies.

Exact-read and write-all loops handle streams that make only partial progress.
The length prefix is limited to an unsigned 64-bit value and its shortest
encoding, so peers have one unambiguous wire representation.

## When to Use

Use this recipe when both peers explicitly share this framing specification and
a blocking byte stream supplies bounded reads and writes. Choose a conservative
`max_frame_size` before accepting untrusted input. Use an established protocol
or serialization library when compatibility, schema evolution, multiplexing,
authentication, or async transport integration matters.

## Implementation

```python
from typing import Protocol


MAX_U64 = (1 << 64) - 1


class FrameError(Exception):
    pass


class FrameProtocolError(FrameError):
    pass


class FrameTooLarge(FrameError):
    pass


class BlockingByteStream(Protocol):
    def read(self, size: int) -> bytes:
        ...

    def write(self, data: bytes | memoryview) -> int:
        ...


def _non_negative_integer(value: int, *, name: str) -> int:
    if isinstance(value, bool) or not isinstance(value, int):
        raise TypeError(f"{name} must be an integer")
    if value < 0:
        raise ValueError(f"{name} must be non-negative")
    return value


def encode_u64_leb128(value: int) -> bytes:
    value = _non_negative_integer(value, name="value")
    if value > MAX_U64:
        raise ValueError("value does not fit in an unsigned 64-bit integer")

    encoded = bytearray()
    while True:
        payload = value & 0x7F
        value >>= 7
        encoded.append(payload | (0x80 if value else 0))
        if not value:
            return bytes(encoded)


def read_exact(stream: BlockingByteStream, size: int) -> bytes:
    size = _non_negative_integer(size, name="size")
    chunks = []
    remaining = size
    while remaining:
        chunk = stream.read(remaining)
        if not isinstance(chunk, bytes):
            raise TypeError("stream.read() must return bytes")
        if not chunk:
            raise EOFError("stream ended before the requested bytes were read")
        if len(chunk) > remaining:
            raise FrameProtocolError("stream.read() returned more bytes than requested")
        chunks.append(chunk)
        remaining -= len(chunk)
    return b"".join(chunks)


def write_all(stream: BlockingByteStream, data: bytes) -> None:
    if not isinstance(data, bytes):
        raise TypeError("data must be bytes")
    view = memoryview(data)
    offset = 0
    while offset < len(view):
        written = stream.write(view[offset:])
        if isinstance(written, bool) or not isinstance(written, int):
            raise TypeError("stream.write() must return an integer byte count")
        if written <= 0 or written > len(view) - offset:
            raise FrameProtocolError("stream.write() made invalid progress")
        offset += written


def read_u64_leb128(stream: BlockingByteStream) -> int:
    raw = bytearray()
    value = 0
    for index in range(10):
        byte = read_exact(stream, 1)[0]
        raw.append(byte)
        payload = byte & 0x7F
        if index == 9 and payload > 1:
            raise FrameProtocolError("length prefix exceeds unsigned 64-bit range")
        value |= payload << (7 * index)

        if byte & 0x80 == 0:
            if bytes(raw) != encode_u64_leb128(value):
                raise FrameProtocolError("length prefix is not canonical")
            return value

    raise FrameProtocolError("length prefix is overlong")


def write_frame(
    stream: BlockingByteStream,
    payload: bytes,
    *,
    max_frame_size: int,
) -> None:
    max_frame_size = _non_negative_integer(
        max_frame_size,
        name="max_frame_size",
    )
    if not isinstance(payload, bytes):
        raise TypeError("payload must be bytes")
    if len(payload) > max_frame_size:
        raise FrameTooLarge("payload exceeds max_frame_size")
    write_all(stream, encode_u64_leb128(len(payload)))
    write_all(stream, payload)


def read_frame(
    stream: BlockingByteStream,
    *,
    max_frame_size: int,
) -> bytes:
    max_frame_size = _non_negative_integer(
        max_frame_size,
        name="max_frame_size",
    )
    size = read_u64_leb128(stream)
    if size > max_frame_size:
        raise FrameTooLarge("declared frame exceeds max_frame_size")
    return read_exact(stream, size)
```

## Example

```python
class PartialMemoryStream:
    def __init__(
        self,
        data: bytes = b"",
        *,
        max_read: int = 5,
        max_write: int = 3,
    ) -> None:
        self.buffer = bytearray(data)
        self.max_read = max_read
        self.max_write = max_write

    def read(self, size: int) -> bytes:
        count = min(size, self.max_read, len(self.buffer))
        chunk = bytes(self.buffer[:count])
        del self.buffer[:count]
        return chunk

    def write(self, data: bytes | memoryview) -> int:
        count = min(len(data), self.max_write)
        self.buffer.extend(data[:count])
        return count


stream = PartialMemoryStream(max_write=1)
frames = [b"", b"a" * 127, b"b" * 128]
for frame in frames:
    write_frame(stream, frame, max_frame_size=128)
decoded = [read_frame(stream, max_frame_size=128) for _ in frames]

largest = PartialMemoryStream(encode_u64_leb128(MAX_U64), max_read=1)
largest_value = read_u64_leb128(largest)

try:
    read_u64_leb128(PartialMemoryStream(b"\x80\x00", max_read=1))
except FrameProtocolError:
    noncanonical_rejected = True
else:
    noncanonical_rejected = False

try:
    read_frame(
        PartialMemoryStream(encode_u64_leb128(129), max_read=1),
        max_frame_size=128,
    )
except FrameTooLarge:
    oversized_rejected = True
else:
    oversized_rejected = False

try:
    read_frame(
        PartialMemoryStream(encode_u64_leb128(3) + b"ab", max_read=1),
        max_frame_size=3,
    )
except EOFError:
    truncated_rejected = True
else:
    truncated_rejected = False

assert (
    decoded,
    largest_value,
    noncanonical_rejected,
    oversized_rejected,
    truncated_rejected,
) == (frames, MAX_U64, True, True, True)
```

## Trade-offs and Limitations

The wire format is useful only when every peer agrees on canonical unsigned
LEB128 and the same frame-size policy. It provides no timeout, cancellation,
checksum, authentication, encryption, compression, schema, multiplexing, or
resynchronization after an error. An oversized prefix is rejected before body
allocation but remains consumed, and a failed write can leave a partial frame;
the caller should normally close or reset that stream. The helper is blocking
and not safe for concurrent access without external ownership.

## Related Snippets

<!-- catalog:related:start -->
- [Read Fixed-Size Blocks with iter() and a Sentinel](../python-language/read-fixed-size-blocks-with-iter-sentinel.md)
- [Batch Items by Estimated Byte Size](../data-processing/batch-items-by-estimated-byte-size.md)
<!-- catalog:related:end -->
