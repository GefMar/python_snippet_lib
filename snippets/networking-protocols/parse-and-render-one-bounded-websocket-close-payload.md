---
title: "Parse and Render One Bounded WebSocket Close Payload"
snippet_type: recipe
use_cases:
  - interoperability
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - encode-a-bounded-http-1-1-chunked-body-for-protocol-tests.md
  - read-and-write-size-capped-varint-frames.md
  - yield-bounded-sse-frames-with-serialized-comment-keepalives.md
---

# Parse and Render One Bounded WebSocket Close Payload

## Idea and Problem

Validate the application-data portion of one WebSocket Close control frame and give its empty form an explicit model.

A Close payload is either empty or starts with a two-byte unsigned status code
in network byte order. Any remaining bytes are one strict UTF-8 reason. A
one-byte payload cannot contain a complete code, and the complete control-frame
payload may not exceed 125 bytes.

The status-code policy below is intentionally frozen: it admits the currently
registered codes used by this profile and the private-use range, while
rejecting reserved and unassigned values.

## When to Use

Use this recipe at a small protocol boundary that already separated the Close
frame's payload bytes from its frame header and masking. It is useful in
protocol tests, diagnostic adapters, or a deliberately narrow WebSocket
implementation whose status-code policy must be reviewable.

Use a maintained WebSocket library for a complete connection. Parsing one
payload does not implement the opening handshake, frame state machine,
masking, fragmentation, role rules, or the closing handshake.

## Implementation

```python
from dataclasses import dataclass

_MAX_CLOSE_PAYLOAD_BYTES = 125
_REGISTERED_CODES = frozenset(
    {
        1000,
        1001,
        1002,
        1003,
        1007,
        1008,
        1009,
        1010,
        1011,
        1012,
        1013,
        1014,
        3000,
        3003,
        3008,
    }
)


class WebSocketClosePayloadError(ValueError):
    pass


@dataclass(frozen=True, slots=True)
class WebSocketClosePayload:
    code: int | None
    reason: str


def _validate_close_code(code: object) -> int:
    if type(code) is not int:
        raise TypeError("code must be an exact integer")
    if code not in _REGISTERED_CODES and not 4000 <= code <= 4999:
        raise WebSocketClosePayloadError(
            "status code is outside the frozen wire profile"
        )
    return code


def parse_websocket_close_payload(
    payload: bytes,
) -> WebSocketClosePayload:
    if type(payload) is not bytes:
        raise TypeError("payload must be exact bytes")
    if len(payload) > _MAX_CLOSE_PAYLOAD_BYTES:
        raise ValueError("payload exceeds the control-frame limit")
    if not payload:
        return WebSocketClosePayload(None, "")
    if len(payload) == 1:
        raise WebSocketClosePayloadError(
            "a non-empty payload needs a complete two-byte status code"
        )

    code = _validate_close_code(int.from_bytes(payload[:2], "big"))
    try:
        reason = payload[2:].decode("utf-8", errors="strict")
    except UnicodeDecodeError as error:
        raise WebSocketClosePayloadError(
            "close reason is not strict UTF-8"
        ) from error
    return WebSocketClosePayload(code, reason)


def render_websocket_close_payload(
    code: int | None,
    reason: str,
) -> bytes:
    if type(reason) is not str:
        raise TypeError("reason must be an exact string")
    if code is None:
        if reason:
            raise WebSocketClosePayloadError(
                "a reason cannot be sent without a wire status code"
            )
        return b""

    code = _validate_close_code(code)
    try:
        reason_bytes = reason.encode("utf-8", errors="strict")
    except UnicodeEncodeError as error:
        raise WebSocketClosePayloadError(
            "close reason contains an invalid Unicode scalar value"
        ) from error

    payload = code.to_bytes(2, "big") + reason_bytes
    if len(payload) > _MAX_CLOSE_PAYLOAD_BYTES:
        raise ValueError("encoded close payload exceeds 125 bytes")
    return payload
```

## Example

```python
normal = render_websocket_close_payload(1000, "complete")
private = render_websocket_close_payload(4001, "retry elsewhere")
empty = render_websocket_close_payload(None, "")
parsed_normal = parse_websocket_close_payload(normal)

assert parsed_normal == WebSocketClosePayload(
    1000,
    "complete",
)
assert parse_websocket_close_payload(private).code == 4001
assert parse_websocket_close_payload(empty) == WebSocketClosePayload(None, "")
assert render_websocket_close_payload(
    parsed_normal.code,
    parsed_normal.reason,
) == normal
```

## Trade-offs and Limitations

Parsing and rendering use `O(b)` time and space for at most 125 payload bytes.
The reason limit is measured after UTF-8 encoding, so a 123-character reason
made of multibyte characters may still be too large.

The admitted registry snapshot is deliberately closed: codes 1000-1003,
1007-1014, 3000, 3003, 3008, and private-use codes 4000-4999. Future IANA
registrations require an explicit update. Code 1010 has endpoint-role meaning
that this payload-only function does not enforce, and private-use meanings
must be agreed by the participating application.

An empty payload is represented as `code=None`, not as synthetic status code
1005. The reason is decoded but not treated as trusted, normalized, localized,
or necessarily human-readable. This recipe does not parse the frame header,
check the Close opcode, enforce client masking, prevent fragmentation, operate
a socket, or decide whether the peer closed cleanly.

## Related Snippets

<!-- catalog:related:start -->
- [Encode a Bounded HTTP/1.1 Chunked Body for Protocol Tests](encode-a-bounded-http-1-1-chunked-body-for-protocol-tests.md)
- [Read and Write Size-Capped Varint Frames](read-and-write-size-capped-varint-frames.md)
- [Yield Bounded SSE Frames with Serialized Comment Keepalives](yield-bounded-sse-frames-with-serialized-comment-keepalives.md)
<!-- catalog:related:end -->
