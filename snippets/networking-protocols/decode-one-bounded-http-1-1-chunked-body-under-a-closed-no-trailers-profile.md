---
title: "Decode One Bounded HTTP/1.1 Chunked Body under a Closed No-Trailers Profile"
snippet_type: recipe
use_cases:
  - networking
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - encode-a-bounded-http-1-1-chunked-body-for-protocol-tests.md
  - classify-one-http-1-response-body-framing-from-validated-metadata.md
  - parse-one-bounded-printable-ascii-http-1-field-section-with-exact-crlf-framing.md
---

# Decode One Bounded HTTP/1.1 Chunked Body under a Closed No-Trailers Profile

## Idea and Problem

Decode one complete HTTP/1.1 chunked body from bytes while enforcing a deliberately small, fail-closed protocol profile and explicit resource limits.

Each nonzero size line is followed by exactly that many arbitrary octets and a
CRLF. A zero last-chunk followed by one empty trailer line must consume the
whole input. The decoder preserves wire chunk boundaries instead of joining a
second full copy of the content.

## When to Use

Use this decoder for bounded protocol fixtures, conformance probes, or a narrow
adapter that has already selected chunked transfer coding and has isolated one
complete body. It accepts the RFC 9112 hexadecimal size and last-chunk forms,
including either letter case and leading zeros, while intentionally rejecting
extensions and trailer fields.

Use a maintained HTTP implementation for live clients, servers, proxies, or
security gateways. General message parsing also needs request or response
semantics, header validation, streaming backpressure, timeouts, connection
reuse rules, and careful handling of multiple messages.

## Implementation

```python
from dataclasses import dataclass

_MAX_WIRE_BYTES = 5 * 1024 * 1024
_MAX_CONTENT_BYTES = 4 * 1024 * 1024
_MAX_CHUNKS = 1_024
_HEXADECIMAL = frozenset(b"0123456789abcdefABCDEF")


@dataclass(frozen=True, slots=True)
class DecodedChunkedBody:
    chunks: tuple[bytes, ...]
    content_length: int


def decode_chunked_body(
    wire: bytes,
    *,
    max_chunks: int = 128,
    max_content_bytes: int = 1024 * 1024,
) -> DecodedChunkedBody:
    if type(wire) is not bytes:
        raise TypeError("wire must be exact bytes")
    if not 5 <= len(wire) <= _MAX_WIRE_BYTES:
        raise ValueError("wire size is outside 5 bytes..5 MiB")
    if type(max_chunks) is not int:
        raise TypeError("max_chunks must be an exact integer")
    if not 0 <= max_chunks <= _MAX_CHUNKS:
        raise ValueError("max_chunks is outside 0..1024")
    if type(max_content_bytes) is not int:
        raise TypeError("max_content_bytes must be an exact integer")
    if not 0 <= max_content_bytes <= _MAX_CONTENT_BYTES:
        raise ValueError("max_content_bytes is outside 0..4 MiB")

    chunks: list[bytes] = []
    content_length = 0
    position = 0

    while True:
        line_end = wire.find(
            b"\r\n",
            position,
            min(len(wire), position + 18),
        )
        if line_end < 0:
            raise ValueError("size line is missing CRLF or exceeds 16 digits")
        size_text = wire[position:line_end]
        if not 1 <= len(size_text) <= 16 or any(
            byte not in _HEXADECIMAL for byte in size_text
        ):
            raise ValueError("size line must contain 1..16 ASCII hexadecimal digits")
        size = int(size_text, 16)
        position = line_end + 2

        if size == 0:
            if wire[position : position + 2] != b"\r\n":
                raise ValueError("last-chunk must have an empty trailer section")
            position += 2
            if position != len(wire):
                raise ValueError("trailing bytes follow the chunked body")
            return DecodedChunkedBody(tuple(chunks), content_length)

        if len(chunks) >= max_chunks:
            raise ValueError("nonzero chunk count exceeds max_chunks")
        if size > max_content_bytes - content_length:
            raise ValueError("declared content exceeds max_content_bytes")

        data_end = position + size
        if data_end + 2 > len(wire):
            raise ValueError("chunk data is truncated")
        if wire[data_end : data_end + 2] != b"\r\n":
            raise ValueError("chunk data is not followed by CRLF")
        chunks.append(wire[position:data_end])
        content_length += size
        position = data_end + 2
```

## Example

```python
def reference_encode(chunks: tuple[bytes, ...]) -> bytes:
    framed = bytearray()
    for chunk in chunks:
        framed.extend(format(len(chunk), "x").encode("ascii"))
        framed.extend(b"\r\n")
        framed.extend(chunk)
        framed.extend(b"\r\n")
    framed.extend(b"0\r\n\r\n")
    return bytes(framed)


def check_tiny_chunk_tuples() -> int:
    from itertools import product

    payload_options = (b"x", b"\x00", b"\r\n")
    checked = 0
    for count in range(4):
        for chunks in product(payload_options, repeat=count):
            decoded = decode_chunked_body(
                reference_encode(chunks),
                max_chunks=count,
                max_content_bytes=sum(map(len, chunks)),
            )
            assert decoded == DecodedChunkedBody(chunks, sum(map(len, chunks)))
            checked += 1
    return checked


checked = check_tiny_chunk_tuples()

mixed_case = decode_chunked_body(
    b"000a\r\n0123456789\r\nB\r\nhello world\r\n000\r\n\r\n",
    max_chunks=2,
    max_content_bytes=21,
)
valid = reference_encode((b"alpha", b"beta"))
prefixes_rejected = 0
for end in range(len(valid)):
    try:
        decode_chunked_body(valid[:end])
    except ValueError:
        prefixes_rejected += 1

invalid_wires = (
    b"1\nX\n0\n\n",
    b"+1\r\nX\r\n0\r\n\r\n",
    b"0x1\r\nX\r\n0\r\n\r\n",
    b"1;name=value\r\nX\r\n0\r\n\r\n",
    b"1234567890abcdef0\r\n",
    b"1\r\nX0\r\n\r\n",
    b"1\r\nX\r\n0\r\nfield: value\r\n\r\n",
    b"1\r\nX\r\n0\r\n\r\nextra",
)
invalid_rejected = 0
for invalid in invalid_wires:
    try:
        decode_chunked_body(invalid)
    except ValueError:
        invalid_rejected += 1

try:
    decode_chunked_body(b"100000\r\n", max_content_bytes=1024)
except ValueError:
    declaration_cap_enforced = True
else:
    declaration_cap_enforced = False

assert (
    checked == 40
    and decode_chunked_body(b"0\r\n\r\n", max_chunks=0, max_content_bytes=0)
    == DecodedChunkedBody((), 0)
    and mixed_case
    == DecodedChunkedBody((b"0123456789", b"hello world"), 21)
    and prefixes_rejected == len(valid)
    and invalid_rejected == len(invalid_wires)
    and declaration_cap_enforced
)
```

## Trade-offs and Limitations

Decoding takes `O(W)` time for `W` wire bytes and stores `O(C + B)` bytes for
`C` chunk objects and `B` accepted content bytes. Each accepted payload is a
bytes slice, so the original wire buffer can be released without invalidating
the result, but the decoder is intentionally not a zero-copy streaming parser.

The 16-digit size-line limit bounds integer parsing independently of the body
cap. A declared over-limit size is rejected before its payload is sliced. This
closed profile rejects valid general-protocol features—chunk extensions and
nonempty trailers—to keep its trust boundary explicit. It does not select
message framing, decode another transfer coding, decompress content, recover
from malformed input, or implement HTTP/2 or HTTP/3 framing.

## Related Snippets

<!-- catalog:related:start -->
- [Encode a Bounded HTTP/1.1 Chunked Body for Protocol Tests](encode-a-bounded-http-1-1-chunked-body-for-protocol-tests.md)
- [Classify One HTTP/1 Response Body Framing from Validated Metadata](classify-one-http-1-response-body-framing-from-validated-metadata.md)
- [Parse One Bounded Printable-ASCII HTTP/1 Field Section with Exact CRLF Framing](parse-one-bounded-printable-ascii-http-1-field-section-with-exact-crlf-framing.md)
<!-- catalog:related:end -->
