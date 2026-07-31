---
title: "Reassemble One Bounded WebSocket Message from Decoded Data Fragments"
snippet_type: algorithm
use_cases:
  - data-transformation
  - interoperability
  - networking
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - parse-one-complete-unfragmented-websocket-data-frame-under-role-masking-rules.md
  - parse-and-render-one-bounded-websocket-close-payload.md
  - ../testing-tooling/enumerate-every-contiguous-chunking-of-bounded-bytes-for-stream-tests.md
---

# Reassemble One Bounded WebSocket Message from Decoded Data Fragments

## Idea and Problem

Reassemble one complete WebSocket text or binary message from already decoded data fragments without decoding split UTF-8 scalars too early.

A final text or binary frame carries a complete one-frame message. Otherwise,
the first frame selects text or binary, and continuation frames contribute the
remaining application bytes until exactly one final continuation. Validating
this sequence prevents a continuation from starting a message, a second data
opcode from changing its kind, or bytes after completion from being hidden.

Text is decoded once, after all payload parts have been joined. A multibyte
UTF-8 scalar may therefore cross any fragment boundary without making either
fragment invalid on its own. Binary messages retain arbitrary bytes.

## When to Use

Use this algorithm after an extension-aware WebSocket frame layer has already
removed masking, interpreted negotiated extensions, enforced wire lengths and
reserved-bit rules, and produced application payload bytes. Pass only decoded
data fragments in their original relative order. An upstream connection state
machine may process and remove interleaved control frames without disturbing
that data-fragment order.

Use a maintained WebSocket implementation for a live connection. This bounded
function is useful for protocol fixtures, message-level adapters, and focused
state-machine tests; it does not read a socket, parse frame headers, negotiate
extensions, or own handshake, control-frame, close, timeout, or transport
lifecycle.

## Implementation

```python
from dataclasses import dataclass
from enum import StrEnum

_CONTINUATION_OPCODE = 0x0
_TEXT_OPCODE = 0x1
_BINARY_OPCODE = 0x2
_MAX_FRAGMENT_COUNT = 64
_MAX_MESSAGE_BYTES = 1_048_576


class WebSocketMessageKind(StrEnum):
    TEXT = "text"
    BINARY = "binary"


class WebSocketFragmentSequenceError(ValueError):
    pass


class WebSocketTextPayloadError(ValueError):
    pass


@dataclass(frozen=True, slots=True)
class DecodedWebSocketDataFragment:
    fin: bool
    opcode: int
    payload: bytes


@dataclass(frozen=True, slots=True)
class ReassembledWebSocketMessage:
    kind: WebSocketMessageKind
    payload: bytes
    text: str | None


def reassemble_websocket_message(
    fragments: tuple[DecodedWebSocketDataFragment, ...],
) -> ReassembledWebSocketMessage:
    """Validate and reassemble exactly one decoded WebSocket data message."""
    if type(fragments) is not tuple:
        raise TypeError("fragments must be an exact tuple")
    if not 1 <= len(fragments) <= _MAX_FRAGMENT_COUNT:
        raise ValueError("fragment count is outside the supported range")

    payload_parts: list[bytes] = []
    total_payload_bytes = 0
    message_kind: WebSocketMessageKind | None = None

    for index, fragment in enumerate(fragments):
        if type(fragment) is not DecodedWebSocketDataFragment:
            raise TypeError(f"fragments[{index}] must be an exact decoded fragment")
        if type(fragment.fin) is not bool:
            raise TypeError(f"fragments[{index}].fin must be an exact boolean")
        if type(fragment.opcode) is not int:
            raise TypeError(f"fragments[{index}].opcode must be an exact integer")
        if type(fragment.payload) is not bytes:
            raise TypeError(f"fragments[{index}].payload must be exact bytes")
        if fragment.opcode not in (
            _CONTINUATION_OPCODE,
            _TEXT_OPCODE,
            _BINARY_OPCODE,
        ):
            raise WebSocketFragmentSequenceError(
                f"fragments[{index}] has an unsupported data opcode"
            )

        if index == 0:
            if fragment.opcode == _CONTINUATION_OPCODE:
                raise WebSocketFragmentSequenceError("a continuation cannot start a message")
            message_kind = (
                WebSocketMessageKind.TEXT
                if fragment.opcode == _TEXT_OPCODE
                else WebSocketMessageKind.BINARY
            )
        elif fragment.opcode != _CONTINUATION_OPCODE:
            raise WebSocketFragmentSequenceError(
                "a fragmented message cannot contain a second data start"
            )

        if fragment.fin and index != len(fragments) - 1:
            raise WebSocketFragmentSequenceError(
                "data fragments follow an already completed message"
            )
        if len(fragment.payload) > _MAX_MESSAGE_BYTES - total_payload_bytes:
            raise ValueError("aggregate payload exceeds the one MiB limit")

        total_payload_bytes += len(fragment.payload)
        payload_parts.append(fragment.payload)

    if not fragments[-1].fin:
        raise WebSocketFragmentSequenceError("the message has no final data fragment")
    if message_kind is None:
        raise AssertionError("a non-empty fragment tuple must select a message kind")

    payload = b"".join(payload_parts)
    text: str | None = None
    if message_kind is WebSocketMessageKind.TEXT:
        try:
            text = payload.decode("utf-8", errors="strict")
        except UnicodeDecodeError as error:
            raise WebSocketTextPayloadError(
                "the complete text message is not strict UTF-8"
            ) from error

    return ReassembledWebSocketMessage(
        kind=message_kind,
        payload=payload,
        text=text,
    )
```

## Example

```python
def contiguous_non_empty_chunkings(payload: bytes) -> tuple[tuple[bytes, ...], ...]:
    chunkings: list[tuple[bytes, ...]] = []
    for cut_mask in range(1 << (len(payload) - 1)):
        chunks: list[bytes] = []
        start = 0
        for position in range(1, len(payload)):
            if cut_mask & (1 << (position - 1)):
                chunks.append(payload[start:position])
                start = position
        chunks.append(payload[start:])
        chunkings.append(tuple(chunks))
    return tuple(chunkings)


def decoded_fragments(
    opcode: int,
    chunks: tuple[bytes, ...],
) -> tuple[DecodedWebSocketDataFragment, ...]:
    if len(chunks) == 1:
        return (DecodedWebSocketDataFragment(True, opcode, chunks[0]),)
    return (
        DecodedWebSocketDataFragment(False, opcode, chunks[0]),
        *(
            DecodedWebSocketDataFragment(False, _CONTINUATION_OPCODE, chunk)
            for chunk in chunks[1:-1]
        ),
        DecodedWebSocketDataFragment(True, _CONTINUATION_OPCODE, chunks[-1]),
    )


def rejected_as(
    fragments: object,
    expected_error: type[Exception],
) -> bool:
    try:
        reassemble_websocket_message(fragments)  # type: ignore[arg-type]
    except expected_error:
        return True
    return False


text_value = "A\u00a2\u20ac\U0001f600"
text_payload = text_value.encode("utf-8")
text_chunkings = contiguous_non_empty_chunkings(text_payload)
for chunks in text_chunkings:
    message = reassemble_websocket_message(decoded_fragments(_TEXT_OPCODE, chunks))
    assert message == ReassembledWebSocketMessage(
        WebSocketMessageKind.TEXT,
        text_payload,
        text_value,
    )

empty_parts = reassemble_websocket_message(
    (
        DecodedWebSocketDataFragment(False, _TEXT_OPCODE, b""),
        DecodedWebSocketDataFragment(False, _CONTINUATION_OPCODE, b"A"),
        DecodedWebSocketDataFragment(True, _CONTINUATION_OPCODE, b""),
    )
)
binary = reassemble_websocket_message(decoded_fragments(_BINARY_OPCODE, (b"\x00", b"\xff\x80")))
maximum_count = reassemble_websocket_message(
    (
        DecodedWebSocketDataFragment(False, _BINARY_OPCODE, b"x"),
        *(DecodedWebSocketDataFragment(False, _CONTINUATION_OPCODE, b"") for _ in range(62)),
        DecodedWebSocketDataFragment(True, _CONTINUATION_OPCODE, b""),
    )
)
maximum_size = reassemble_websocket_message(
    (DecodedWebSocketDataFragment(True, _BINARY_OPCODE, b"x" * _MAX_MESSAGE_BYTES),)
)

too_many = (
    DecodedWebSocketDataFragment(False, _BINARY_OPCODE, b""),
    *(DecodedWebSocketDataFragment(False, _CONTINUATION_OPCODE, b"") for _ in range(63)),
    DecodedWebSocketDataFragment(True, _CONTINUATION_OPCODE, b""),
)
invalid_cases = (
    ((), ValueError),
    (too_many, ValueError),
    (
        (
            DecodedWebSocketDataFragment(False, _BINARY_OPCODE, b"x" * _MAX_MESSAGE_BYTES),
            DecodedWebSocketDataFragment(True, _CONTINUATION_OPCODE, b"x"),
        ),
        ValueError,
    ),
    (
        (DecodedWebSocketDataFragment(True, _CONTINUATION_OPCODE, b"x"),),
        WebSocketFragmentSequenceError,
    ),
    (
        (
            DecodedWebSocketDataFragment(False, _TEXT_OPCODE, b"a"),
            DecodedWebSocketDataFragment(True, _BINARY_OPCODE, b"b"),
        ),
        WebSocketFragmentSequenceError,
    ),
    (
        (DecodedWebSocketDataFragment(True, 0x3, b"x"),),
        WebSocketFragmentSequenceError,
    ),
    (
        (
            DecodedWebSocketDataFragment(True, _TEXT_OPCODE, b"a"),
            DecodedWebSocketDataFragment(True, _CONTINUATION_OPCODE, b"b"),
        ),
        WebSocketFragmentSequenceError,
    ),
    (
        (DecodedWebSocketDataFragment(False, _TEXT_OPCODE, b"unfinished"),),
        WebSocketFragmentSequenceError,
    ),
    (
        (DecodedWebSocketDataFragment(True, _TEXT_OPCODE, b"\xff"),),
        WebSocketTextPayloadError,
    ),
    (
        (DecodedWebSocketDataFragment(1, _TEXT_OPCODE, b"x"),),
        TypeError,
    ),
    (
        (DecodedWebSocketDataFragment(True, True, b"x"),),
        TypeError,
    ),
    (
        (DecodedWebSocketDataFragment(True, _BINARY_OPCODE, bytearray(b"x")),),
        TypeError,
    ),
)

assert (
    len(text_chunkings),
    empty_parts,
    binary,
    len(maximum_count.payload),
    len(maximum_size.payload),
    sum(rejected_as(candidate, expected) for candidate, expected in invalid_cases),
) == (
    512,
    ReassembledWebSocketMessage(WebSocketMessageKind.TEXT, b"A", "A"),
    ReassembledWebSocketMessage(WebSocketMessageKind.BINARY, b"\x00\xff\x80", None),
    1,
    _MAX_MESSAGE_BYTES,
    len(invalid_cases),
)
```

## Trade-offs and Limitations

For `f` fragments and `p` aggregate payload bytes, validation takes `O(f)`
steps and reassembly takes `O(p)` time and space. The input payload objects,
the temporary parts list, and the joined message coexist. A text result also
materializes its decoded string. The byte cap is checked against the remaining
budget for each fragment before `bytes.join` allocates the complete payload.

The first fragment must use text opcode `1` or binary opcode `2`. A non-final
start requires only opcode-`0` continuations, and the final tuple element must
be the sole final data fragment. Empty payload parts remain valid. API shape
violations raise `TypeError`, count and aggregate-size bounds raise
`ValueError`, sequence violations raise `WebSocketFragmentSequenceError`, and
invalid complete text raises `WebSocketTextPayloadError`.

The payload cap applies to application bytes after masking removal and any
negotiated extension-aware decoding. This function does not validate wire
lengths, RSV bits, masking direction, extension state, or control frames.
Removing interleaved control frames upstream must preserve data-fragment order.
It handles exactly one completely materialized message, not multiple messages,
incremental connection state, sockets, handshakes, closing behavior, or
application-level trust.

## Related Snippets

<!-- catalog:related:start -->
- [Parse One Complete Unfragmented WebSocket Data Frame Under Role Masking Rules](parse-one-complete-unfragmented-websocket-data-frame-under-role-masking-rules.md)
- [Parse and Render One Bounded WebSocket Close Payload](parse-and-render-one-bounded-websocket-close-payload.md)
- [Enumerate Every Contiguous Chunking of Bounded Bytes for Stream Tests](../testing-tooling/enumerate-every-contiguous-chunking-of-bounded-bytes-for-stream-tests.md)
<!-- catalog:related:end -->
