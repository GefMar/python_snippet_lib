---
title: "Parse One Bounded HTTP/2 SETTINGS Frame"
snippet_type: recipe
use_cases:
  - interoperability
  - networking
  - parsing
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - parse-one-complete-unfragmented-websocket-data-frame-under-role-masking-rules.md
  - parse-one-bounded-proxy-protocol-version-two-tcp-header-without-tlvs.md
  - parse-and-format-a-strict-w3c-traceparent-version-00-value.md
---

# Parse One Bounded HTTP/2 SETTINGS Frame

## Idea and Problem

Parse exactly one complete HTTP/2 SETTINGS frame while preserving every setting in wire order.

[RFC 9113 Section 6.5](https://www.rfc-editor.org/rfc/rfc9113.html#section-6.5)
processes settings in their declared order, so a later occurrence of an
identifier replaces an earlier one. A parser must therefore not collapse the
payload into a mapping. Unknown identifiers are also retained for diagnostics,
although an endpoint applying the result must ignore unsupported settings.

The receive path deliberately ignores the reserved stream bit and all flag
bits except `ACK`, as the protocol requires. Each occurrence of a core RFC 9113
setting is still validated before the immutable ordered result is returned.

## When to Use

Use this recipe in protocol tests, capture analysis, or a bounded HTTP/2
boundary that has already isolated one complete frame and knows the sender's
client-or-server role. Pass the frame-size limit currently enforced by the
receiver; the default represents the mandatory initial 16,384-byte limit.

Use a maintained HTTP/2 implementation for a live connection. A complete
endpoint must manage the connection preface, incremental reads, setting
application and acknowledgement order, extension semantics, flow-control
state, stream lifecycle, timeouts, and transport security.

## Implementation

```python
from dataclasses import dataclass

_FRAME_HEADER_BYTES = 9
_SETTINGS_FRAME_TYPE = 0x04
_ACK_FLAG = 0x01
_STREAM_IDENTIFIER_MASK = (1 << 31) - 1
_SETTING_BYTES = 6
_MAX_SETTINGS = 4_096
_MINIMUM_FRAME_SIZE_LIMIT = 1 << 14
_MAXIMUM_FRAME_SIZE_LIMIT = (1 << 24) - 1


class HTTP2SettingsFrameError(ValueError):
    """Raised when bytes violate the bounded SETTINGS receive contract."""


@dataclass(frozen=True, slots=True)
class HTTP2Setting:
    identifier: int
    value: int


@dataclass(frozen=True, slots=True)
class HTTP2SettingsFrame:
    acknowledgement: bool
    settings: tuple[HTTP2Setting, ...]


def _validate_core_setting(
    identifier: int,
    value: int,
    *,
    sender_is_client: bool,
) -> None:
    if identifier in (0x01, 0x03, 0x06):
        return
    if identifier == 0x02:
        if value not in (0, 1):
            raise HTTP2SettingsFrameError("SETTINGS_ENABLE_PUSH must be zero or one")
        if not sender_is_client and value == 1:
            raise HTTP2SettingsFrameError("a server sender must not enable push")
        return
    if identifier == 0x04:
        if value > (1 << 31) - 1:
            raise HTTP2SettingsFrameError("SETTINGS_INITIAL_WINDOW_SIZE is too large")
        return
    if identifier == 0x05 and not (1 << 14) <= value <= (1 << 24) - 1:
        raise HTTP2SettingsFrameError("SETTINGS_MAX_FRAME_SIZE is outside its range")


def parse_http2_settings_frame(
    frame: bytes,
    *,
    sender_is_client: bool,
    maximum_frame_size: int = 16_384,
) -> HTTP2SettingsFrame:
    """Parse one exact SETTINGS frame under the supplied receive limit."""
    if type(frame) is not bytes:
        raise TypeError("frame must be exact bytes")
    if type(sender_is_client) is not bool:
        raise TypeError("sender_is_client must be an exact bool")
    if type(maximum_frame_size) is not int:
        raise TypeError("maximum_frame_size must be an exact integer")
    if not _MINIMUM_FRAME_SIZE_LIMIT <= maximum_frame_size <= _MAXIMUM_FRAME_SIZE_LIMIT:
        raise ValueError("maximum_frame_size is outside the HTTP/2 range")
    if len(frame) < _FRAME_HEADER_BYTES:
        raise HTTP2SettingsFrameError("frame is shorter than the fixed header")

    payload_length = int.from_bytes(frame[0:3], "big")
    if payload_length > maximum_frame_size:
        raise HTTP2SettingsFrameError("payload exceeds the receiver frame-size limit")
    if frame[3] != _SETTINGS_FRAME_TYPE:
        raise HTTP2SettingsFrameError("frame type is not SETTINGS")

    stream_identifier = int.from_bytes(frame[5:9], "big") & _STREAM_IDENTIFIER_MASK
    if stream_identifier != 0:
        raise HTTP2SettingsFrameError("a SETTINGS frame must use stream zero")

    expected_length = _FRAME_HEADER_BYTES + payload_length
    if len(frame) != expected_length:
        raise HTTP2SettingsFrameError("input length does not match the declared frame length")

    acknowledgement = bool(frame[4] & _ACK_FLAG)
    if acknowledgement and payload_length:
        raise HTTP2SettingsFrameError("an acknowledged SETTINGS frame must be empty")
    if payload_length % _SETTING_BYTES:
        raise HTTP2SettingsFrameError("SETTINGS payload length must be a multiple of six")

    setting_count = payload_length // _SETTING_BYTES
    if setting_count > _MAX_SETTINGS:
        raise HTTP2SettingsFrameError("setting count exceeds the local limit")

    settings: list[HTTP2Setting] = []
    for offset in range(_FRAME_HEADER_BYTES, expected_length, _SETTING_BYTES):
        identifier = int.from_bytes(frame[offset : offset + 2], "big")
        value = int.from_bytes(frame[offset + 2 : offset + 6], "big")
        _validate_core_setting(
            identifier,
            value,
            sender_is_client=sender_is_client,
        )
        settings.append(HTTP2Setting(identifier, value))

    return HTTP2SettingsFrame(acknowledgement, tuple(settings))
```

## Example

```python
def render_settings_frame(
    settings: tuple[tuple[int, int], ...],
    *,
    flags: int = 0,
    stream_word: int = 0,
) -> bytes:
    payload = b"".join(
        identifier.to_bytes(2, "big") + value.to_bytes(4, "big") for identifier, value in settings
    )
    return (
        len(payload).to_bytes(3, "big")
        + bytes((_SETTINGS_FRAME_TYPE, flags))
        + stream_word.to_bytes(4, "big")
        + payload
    )


ordered = (
    (0x03, 100),
    (0xBEEF, 7),
    (0x03, 200),
    (0x01, 0xFFFF_FFFF),
    (0x04, 0x7FFF_FFFF),
    (0x05, 16_384),
    (0x05, 0x00FF_FFFF),
    (0x06, 0xFFFF_FFFF),
    (0x02, 1),
)
parsed = parse_http2_settings_frame(
    render_settings_frame(
        ordered,
        flags=0xFE,  # Every unused flag bit is set; ACK remains clear.
        stream_word=0x8000_0000,  # The ignored reserved bit is set.
    ),
    sender_is_client=True,
)
acknowledgement = parse_http2_settings_frame(
    render_settings_frame((), flags=0xFF, stream_word=0x8000_0000),
    sender_is_client=False,
)

empty_frame = render_settings_frame(())
accepted_limit_boundaries = tuple(
    parse_http2_settings_frame(
        empty_frame,
        sender_is_client=True,
        maximum_frame_size=limit,
    )
    for limit in (_MINIMUM_FRAME_SIZE_LIMIT, _MAXIMUM_FRAME_SIZE_LIMIT)
)

at_setting_limit = render_settings_frame(
    tuple((0xCAFE, index) for index in range(_MAX_SETTINGS)),
)
accepted_with_larger_limit = parse_http2_settings_frame(
    at_setting_limit,
    sender_is_client=True,
    maximum_frame_size=len(at_setting_limit) - _FRAME_HEADER_BYTES,
)


def is_rejected(
    candidate: bytes,
    *,
    sender_is_client: bool = True,
    maximum_frame_size: int = 16_384,
) -> bool:
    try:
        parse_http2_settings_frame(
            candidate,
            sender_is_client=sender_is_client,
            maximum_frame_size=maximum_frame_size,
        )
    except (TypeError, ValueError):
        return True
    return False


wrong_type = bytearray(render_settings_frame(()))
wrong_type[3] = 0x03
odd_payload = b"x"
odd_length_frame = (
    len(odd_payload).to_bytes(3, "big")
    + bytes((_SETTINGS_FRAME_TYPE, 0))
    + b"\x00\x00\x00\x00"
    + odd_payload
)
too_many_settings = render_settings_frame(
    tuple((0xCAFE, index) for index in range(_MAX_SETTINGS + 1)),
)
truncated_frame = render_settings_frame(((0x01, 0),))[:-1]

invalid_known_settings = (
    (render_settings_frame(((0x02, 2),)), True),
    (render_settings_frame(((0x02, 1),)), False),
    (render_settings_frame(((0x04, 1 << 31),)), True),
    (render_settings_frame(((0x05, (1 << 14) - 1),)), True),
    (render_settings_frame(((0x05, 1 << 24),)), True),
    (render_settings_frame(((0x05, 0), (0x05, 1 << 14))), True),
)
rejected = sum(
    is_rejected(candidate, sender_is_client=sender_is_client)
    for candidate, sender_is_client in invalid_known_settings
)
rejected += sum(
    (
        is_rejected(bytes(wrong_type)),
        is_rejected(render_settings_frame((), stream_word=1)),
        is_rejected(render_settings_frame(((0x01, 0),), flags=_ACK_FLAG)),
        is_rejected(odd_length_frame),
        is_rejected(render_settings_frame(()) + b"x"),
        is_rejected(truncated_frame),
        is_rejected(at_setting_limit),
        is_rejected(
            too_many_settings,
            maximum_frame_size=len(too_many_settings) - _FRAME_HEADER_BYTES,
        ),
        is_rejected(
            empty_frame,
            maximum_frame_size=_MINIMUM_FRAME_SIZE_LIMIT - 1,
        ),
        is_rejected(
            empty_frame,
            maximum_frame_size=_MAXIMUM_FRAME_SIZE_LIMIT + 1,
        ),
    )
)

assert (
    parsed,
    acknowledgement,
    accepted_limit_boundaries,
    len(accepted_with_larger_limit.settings),
    rejected,
) == (
    HTTP2SettingsFrame(
        False,
        tuple(HTTP2Setting(identifier, value) for identifier, value in ordered),
    ),
    HTTP2SettingsFrame(True, ()),
    (HTTP2SettingsFrame(False, ()), HTTP2SettingsFrame(False, ())),
    4_096,
    16,
)
```

## Trade-offs and Limitations

Parsing takes `O(s)` time and space for `s` settings. The complete input and
the immutable result are materialized, while the explicit frame-size and
4,096-setting limits bound retained work. Every core setting occurrence is
validated, including one that a later duplicate would replace.

Ignoring unused flags and the reserved stream bit is receive-side behavior,
not permission to emit them. They are deliberately absent from the normalized
result. A sender must leave those bits clear. The parser preserves unknown
settings so diagnostics do not lose wire evidence, but an endpoint applying
the result must ignore unsupported identifiers.

This recipe does not apply settings, reduce duplicates to their last values,
track outstanding acknowledgements, validate extension-specific semantics, or
translate failures into HTTP/2 connection error codes. It also does not read
from a stream or parse adjacent frames. The caller remains responsible for
connection state, sender-role accuracy, the current receive limit, and all
effects of accepted settings.

## Related Snippets

<!-- catalog:related:start -->
- [Parse One Complete Unfragmented WebSocket Data Frame Under Role Masking Rules](parse-one-complete-unfragmented-websocket-data-frame-under-role-masking-rules.md)
- [Parse One Bounded PROXY Protocol Version Two TCP Header Without TLVs](parse-one-bounded-proxy-protocol-version-two-tcp-header-without-tlvs.md)
- [Parse and Format a Strict W3C traceparent Version 00 Value](parse-and-format-a-strict-w3c-traceparent-version-00-value.md)
<!-- catalog:related:end -->
