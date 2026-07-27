---
title: "Encode a Bounded HTTP/1.1 Chunked Body for Protocol Tests"
snippet_type: recipe
use_cases:
  - networking
  - serialization
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - choose-buffered-or-streaming-multipart-encoding-from-bounded-parts.md
  - read-and-write-size-capped-varint-frames.md
  - ../testing-tooling/verify-ordered-http-client-expectations-with-bounded-mismatch-reports.md
---

# Encode a Bounded HTTP/1.1 Chunked Body for Protocol Tests

## Idea and Problem

Build deterministic HTTP/1.1 chunked transfer-coding bytes from bounded payload chunks without opening a connection or relying on a client library's framing policy.

Each non-empty chunk is preceded by its hexadecimal octet length and followed
by CRLF. One zero-size last chunk and the final empty line terminate the body.
Materializing the complete bounded result makes exact parser fixtures and
assertions reproducible.

## When to Use

Use this encoder to test an HTTP/1.1 parser, proxy, fixture decoder, or byte-level
transport adapter with known chunk boundaries. The caller supplies content
chunks rather than already framed wire data, and explicit limits bound input
count and aggregate content before the returned bytes are trusted.

Do not pass the returned bytes as an ordinary body to a high-level HTTP client;
that client might frame them a second time. Use the client's documented
streaming API for integration tests and verify what a controlled server
actually received.

## Implementation

```python
from collections.abc import Iterable


_MAX_CHUNKS = 1_024
_MAX_CHUNK_BYTES = 1024 * 1024
_MAX_CONTENT_BYTES = 4 * 1024 * 1024


def encode_chunked_body(
    chunks: Iterable[bytes],
    *,
    max_chunks: int = 128,
    max_content_bytes: int = 1024 * 1024,
) -> bytes:
    if isinstance(chunks, (str, bytes, bytearray)):
        raise TypeError("chunks must be an iterable of byte values")
    for name, value, upper in (
        ("max_chunks", max_chunks, _MAX_CHUNKS),
        ("max_content_bytes", max_content_bytes, _MAX_CONTENT_BYTES),
    ):
        if isinstance(value, bool) or not isinstance(value, int):
            raise TypeError(f"{name} must be an integer")
        if not 1 <= value <= upper:
            raise ValueError(f"{name} is outside the supported range")

    encoded = bytearray()
    content_bytes = 0
    chunk_count = 0

    for chunk in chunks:
        chunk_count += 1
        if chunk_count > max_chunks:
            raise ValueError("chunk count exceeds max_chunks")
        if not isinstance(chunk, bytes):
            raise TypeError("chunks must contain immutable bytes")
        if not chunk:
            raise ValueError("data chunks must not be empty")
        if len(chunk) > _MAX_CHUNK_BYTES:
            raise ValueError("a chunk exceeds the supported size")

        content_bytes += len(chunk)
        if content_bytes > max_content_bytes:
            raise ValueError("content exceeds max_content_bytes")
        encoded.extend(f"{len(chunk):X}\r\n".encode("ascii"))
        encoded.extend(chunk)
        encoded.extend(b"\r\n")

    encoded.extend(b"0\r\n\r\n")
    return bytes(encoded)
```

## Example

```python
wire_body = encode_chunked_body(
    (b"Wiki", b"pedia", b" in\r\n\r\nchunks."),
    max_chunks=3,
    max_content_bytes=64,
)
empty_body = encode_chunked_body(())

assert (wire_body, empty_body) == (
    b"4\r\nWiki\r\n5\r\npedia\r\nE\r\n in\r\n\r\nchunks.\r\n0\r\n\r\n",
    b"0\r\n\r\n",
)
```

## Trade-offs and Limitations

The helper buffers the complete encoded body, so its memory use is proportional
to the configured content bound plus small per-chunk framing. This is useful
for deterministic tests, not large live uploads. Rejecting empty data chunks
avoids confusing them with the terminal chunk; omit an empty application
fragment before calling the encoder.

This implements only the basic HTTP/1.1 chunked body grammar. It does not emit
chunk extensions or trailers, construct request headers, parse responses, or
validate a peer. HTTP/2 and HTTP/3 use their own framing rather than this
transfer coding. Exact fixture bytes prove the encoder's output, not the
behavior of an operating-system socket, proxy, or high-level client.

## Related Snippets

<!-- catalog:related:start -->
- [Choose Buffered or Streaming Multipart Encoding from Bounded Parts](choose-buffered-or-streaming-multipart-encoding-from-bounded-parts.md)
- [Read and Write Size-Capped Varint Frames](read-and-write-size-capped-varint-frames.md)
- [Verify Ordered HTTP Client Expectations with Bounded Mismatch Reports](../testing-tooling/verify-ordered-http-client-expectations-with-bounded-mismatch-reports.md)
<!-- catalog:related:end -->
