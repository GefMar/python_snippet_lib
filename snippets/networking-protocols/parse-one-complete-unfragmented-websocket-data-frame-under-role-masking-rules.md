---
title: "Parse One Complete Unfragmented WebSocket Data Frame Under Role Masking Rules"
snippet_type: recipe
use_cases:
  - interoperability
  - networking
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - parse-and-render-one-bounded-websocket-close-payload.md
  - encode-and-decode-one-canonical-size-capped-netstring.md
  - read-and-write-size-capped-varint-frames.md
---

# Parse One Complete Unfragmented WebSocket Data Frame Under Role Masking Rules

## Idea and Problem

Decode exactly one complete WebSocket text or binary frame while enforcing the sender's masking direction and the canonical wire length.

The parser admits a deliberately closed no-extensions profile: `FIN` is set,
reserved bits are clear, and the opcode is text or binary. It validates the
complete header and declared payload before allocating the unmasked result.
Client frames require a four-byte mask; server frames must not carry one.

## When to Use

Use this recipe in protocol tests, diagnostic tooling, or a small boundary that
already knows the sender role and receives exactly one materialized data frame.
It handles the RFC 6455 length encodings through a locally capped 64 KiB
payload and validates a complete text message as strict UTF-8.

Use a maintained WebSocket implementation for a live connection. Real peers
need HTTP upgrade handling, incremental reads, fragmentation, control-frame
interleaving, extension negotiation, close state, timeouts, and transport
security.

## Implementation

```python
from dataclasses import dataclass
from enum import StrEnum

_MAX_PAYLOAD_BYTES = 65_536
_MAX_FRAME_BYTES = 2 + 8 + 4 + _MAX_PAYLOAD_BYTES


class FrameSender(StrEnum):
    CLIENT = "client"
    SERVER = "server"


class DataFrameKind(StrEnum):
    TEXT = "text"
    BINARY = "binary"


class WebSocketDataFrameError(ValueError):
    pass


@dataclass(frozen=True, slots=True)
class WebSocketDataFrame:
    kind: DataFrameKind
    payload: bytes
    text: str | None


def _frame_error(message: str) -> WebSocketDataFrameError:
    return WebSocketDataFrameError(message)


def parse_websocket_data_frame(
    frame: bytes,
    *,
    sender: FrameSender,
) -> WebSocketDataFrame:
    """Parse one complete unfragmented text or binary frame."""
    if type(frame) is not bytes:
        raise TypeError("frame must be exact bytes")
    if not 2 <= len(frame) <= _MAX_FRAME_BYTES:
        raise ValueError("frame length is outside the supported range")
    if type(sender) is not FrameSender:
        raise TypeError("sender must be an exact FrameSender")

    first, second = frame[0], frame[1]
    if first & 0x80 == 0:
        raise _frame_error("fragmented messages are outside this profile")
    if first & 0x70:
        raise _frame_error("reserved bits require an unsupported extension")

    opcode = first & 0x0F
    if opcode == 0x1:
        kind = DataFrameKind.TEXT
    elif opcode == 0x2:
        kind = DataFrameKind.BINARY
    else:
        raise _frame_error("opcode is not an admitted text or binary frame")

    masked = bool(second & 0x80)
    if sender is FrameSender.CLIENT and not masked:
        raise _frame_error("a client frame must be masked")
    if sender is FrameSender.SERVER and masked:
        raise _frame_error("a server frame must not be masked")

    length_marker = second & 0x7F
    cursor = 2
    if length_marker <= 125:
        payload_length = length_marker
    elif length_marker == 126:
        if len(frame) < cursor + 2:
            raise _frame_error("the 16-bit extended length is truncated")
        payload_length = int.from_bytes(frame[cursor : cursor + 2], "big")
        cursor += 2
        if payload_length < 126:
            raise _frame_error("payload length does not use its shortest encoding")
    else:
        if len(frame) < cursor + 8:
            raise _frame_error("the 64-bit extended length is truncated")
        encoded_length = frame[cursor : cursor + 8]
        cursor += 8
        if encoded_length[0] & 0x80:
            raise _frame_error("the 64-bit payload length has its high bit set")
        payload_length = int.from_bytes(encoded_length, "big")
        if payload_length < 65_536:
            raise _frame_error("payload length does not use its shortest encoding")

    if payload_length > _MAX_PAYLOAD_BYTES:
        raise _frame_error("payload exceeds the local 64 KiB limit")

    masking_key: bytes | None = None
    if masked:
        if len(frame) < cursor + 4:
            raise _frame_error("the masking key is truncated")
        masking_key = frame[cursor : cursor + 4]
        cursor += 4

    expected_length = cursor + payload_length
    if len(frame) < expected_length:
        raise _frame_error("the payload is truncated")
    if len(frame) > expected_length:
        raise _frame_error("bytes follow the one declared frame")

    wire_payload = frame[cursor:expected_length]
    if masking_key is None:
        payload = wire_payload
    else:
        payload = bytes(value ^ masking_key[index % 4] for index, value in enumerate(wire_payload))

    text: str | None = None
    if kind is DataFrameKind.TEXT:
        try:
            text = payload.decode("utf-8", errors="strict")
        except UnicodeDecodeError as error:
            raise _frame_error("text payload is not strict UTF-8") from error

    return WebSocketDataFrame(kind=kind, payload=payload, text=text)


```

## Example

```python
# RFC 6455 Section 5.7 single-frame "Hello" examples.
server_hello = bytes.fromhex("81 05 48 65 6c 6c 6f")
client_hello = bytes.fromhex("81 85 37 fa 21 3d 7f 9f 4d 51 58")

parsed_server = parse_websocket_data_frame(
    server_hello,
    sender=FrameSender.SERVER,
)
parsed_client = parse_websocket_data_frame(
    client_hello,
    sender=FrameSender.CLIENT,
)

large_payload = b"x" * 65_536
large_frame = b"\x82\x7f" + len(large_payload).to_bytes(8, "big") + large_payload
parsed_large = parse_websocket_data_frame(
    large_frame,
    sender=FrameSender.SERVER,
)

invalid_frames = (
    (b"\x81\x05Hello", FrameSender.CLIENT),  # Missing client mask.
    (b"\x81\x7e\x00\x7d" + b"x" * 125, FrameSender.SERVER),  # Non-minimal.
    (b"\x81\x01\xff", FrameSender.SERVER),  # Invalid UTF-8.
    (server_hello + b"x", FrameSender.SERVER),  # Trailing byte.
)
rejected = 0
for invalid, sender in invalid_frames:
    try:
        parse_websocket_data_frame(invalid, sender=sender)
    except WebSocketDataFrameError:
        rejected += 1

assert (
    parsed_server,
    parsed_client,
    parsed_large.kind,
    len(parsed_large.payload),
    rejected,
) == (
    WebSocketDataFrame(DataFrameKind.TEXT, b"Hello", "Hello"),
    WebSocketDataFrame(DataFrameKind.TEXT, b"Hello", "Hello"),
    DataFrameKind.BINARY,
    65_536,
    4,
)
```

## Trade-offs and Limitations

Parsing and optional unmasking take `O(p)` time for payload size `p`. Unmasked
server bytes can be reused directly, while a client frame requires a new
bounded byte string. The complete input and output are both materialized.

The accepted length encodings are canonical: values through 125 stay in the
base header, values 126 through 65,535 use the 16-bit extension, and 65,536
uses the 64-bit extension. Larger payload declarations fail before payload
slicing. All admitted wire, masking, opcode, and UTF-8 violations use
`WebSocketDataFrameError`; API type and outer-size violations remain
`TypeError` or `ValueError`.

Because only final text frames are accepted, validating the payload itself as
UTF-8 is equivalent to validating the complete message. The parser does not
accept continuation or control frames, fragments, negotiated reserved bits,
per-message compression, multiple frames, incremental transport reads, or
connection-state transitions. Successful parsing says nothing about the
message's trustworthiness or application semantics.

## Related Snippets

<!-- catalog:related:start -->
- [Parse and Render One Bounded WebSocket Close Payload](parse-and-render-one-bounded-websocket-close-payload.md)
- [Encode and Decode One Canonical Size-Capped Netstring](encode-and-decode-one-canonical-size-capped-netstring.md)
- [Read and Write Size-Capped Varint Frames](read-and-write-size-capped-varint-frames.md)
<!-- catalog:related:end -->
