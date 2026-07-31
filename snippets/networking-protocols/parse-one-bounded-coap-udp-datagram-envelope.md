---
title: "Parse One Bounded CoAP UDP Datagram Envelope"
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
  - parse-one-bounded-http-2-settings-frame.md
  - parse-one-bounded-proxy-protocol-version-two-tcp-header-without-tlvs.md
  - decode-one-bounded-http-1-1-chunked-body-under-a-closed-no-trailers-profile.md
---

# Parse One Bounded CoAP UDP Datagram Envelope

## Idea and Problem

Parse exactly one size-capped CoAP version 1 UDP datagram into an immutable raw envelope.

[RFC 7252 Section 3](https://www.rfc-editor.org/rfc/rfc7252.html#section-3)
places a token and delta-encoded ordered options after a four-byte header. A
`0xFF` byte starts the payload only where another option could begin, so the
same byte inside an option value is ordinary data. The parser retains repeated
and unknown options without assigning application meaning to them.

The result also enforces the RFC 7252 message-type matrix: requests use
Confirmable or Non-confirmable messages, responses may additionally use an
Acknowledgement, Reset messages are empty, and an empty Non-confirmable
message is invalid. Code classes 1, 3, 6, and 7 are reserved by this profile.

## When to Use

Use this recipe after one complete UDP datagram has already been received and
the caller needs a bounded structural envelope for protocol tests, capture
inspection, or a deliberately small endpoint boundary. The 1,152-byte message
and 1,024-byte payload caps follow the conservative upper bounds in
[RFC 7252 Section 4.6](https://www.rfc-editor.org/rfc/rfc7252.html#section-4.6).

Use a maintained CoAP implementation for a live endpoint. Parsing alone does
not implement retransmission, duplicate suppression, request-response
correlation, option registries, block-wise transfer, multicast rules, DTLS,
or OSCORE. A token and Message ID are correlation fields, not authentication.

## Implementation

```python
from dataclasses import dataclass
from enum import IntEnum

_HEADER_BYTES = 4
_COAP_VERSION = 1
_PAYLOAD_MARKER = 0xFF
_MAX_DATAGRAM_BYTES = 1_152
_MAX_PAYLOAD_BYTES = 1_024
_MAX_TOKEN_BYTES = 8
_MAX_OPTIONS = 32
_MAX_OPTION_VALUE_BYTES = 1_024
_MAX_OPTION_NUMBER = 65_535


class CoapDatagramError(ValueError):
    """Raised when bytes violate the bounded CoAP UDP envelope profile."""


class CoapMessageType(IntEnum):
    CONFIRMABLE = 0
    NON_CONFIRMABLE = 1
    ACKNOWLEDGEMENT = 2
    RESET = 3


@dataclass(frozen=True, slots=True)
class CoapOption:
    number: int
    value: bytes


@dataclass(frozen=True, slots=True)
class CoapUdpDatagram:
    message_type: CoapMessageType
    code_class: int
    code_detail: int
    message_id: int
    token: bytes
    options: tuple[CoapOption, ...]
    payload: bytes


def _decode_option_argument(
    nibble: int,
    datagram: bytes,
    offset: int,
    *,
    field: str,
) -> tuple[int, int]:
    if nibble <= 12:
        return nibble, offset
    if nibble == 13:
        if offset >= len(datagram):
            raise CoapDatagramError(f"truncated extended option {field}")
        return 13 + datagram[offset], offset + 1
    if nibble == 14:
        if offset + 2 > len(datagram):
            raise CoapDatagramError(f"truncated extended option {field}")
        extension = int.from_bytes(datagram[offset : offset + 2], "big")
        return 269 + extension, offset + 2
    raise CoapDatagramError(f"reserved option {field} nibble")


def _validate_message_shape(
    message_type: CoapMessageType,
    code_class: int,
    code_detail: int,
    *,
    token_length: int,
    datagram_length: int,
) -> bool:
    if code_class not in (0, 2, 4, 5):
        raise CoapDatagramError("code class is reserved by RFC 7252")

    if code_class == 0 and code_detail == 0:
        if message_type is CoapMessageType.NON_CONFIRMABLE:
            raise CoapDatagramError("a Non-confirmable message must not be empty")
        if token_length != 0 or datagram_length != _HEADER_BYTES:
            raise CoapDatagramError("an empty message must contain only its header")
        return True

    if code_class == 0:
        if message_type not in (
            CoapMessageType.CONFIRMABLE,
            CoapMessageType.NON_CONFIRMABLE,
        ):
            raise CoapDatagramError("a request must be Confirmable or Non-confirmable")
    elif message_type is CoapMessageType.RESET:
        raise CoapDatagramError("a Reset message must be empty")
    return False


def parse_coap_udp_datagram(datagram: bytes) -> CoapUdpDatagram:
    """Parse one complete CoAP version 1 message from one UDP datagram."""
    if type(datagram) is not bytes:
        raise TypeError("datagram must be exact immutable bytes")
    if not _HEADER_BYTES <= len(datagram) <= _MAX_DATAGRAM_BYTES:
        raise CoapDatagramError("datagram length is outside 4..1,152 bytes")

    first = datagram[0]
    version = first >> 6
    if version != _COAP_VERSION:
        raise CoapDatagramError("datagram must use CoAP version 1")

    message_type = CoapMessageType((first >> 4) & 0b11)
    token_length = first & 0b1111
    if token_length > _MAX_TOKEN_BYTES:
        raise CoapDatagramError("token length is reserved or too large")

    code = datagram[1]
    code_class = code >> 5
    code_detail = code & 0b1_1111
    message_id = int.from_bytes(datagram[2:4], "big")
    is_empty = _validate_message_shape(
        message_type,
        code_class,
        code_detail,
        token_length=token_length,
        datagram_length=len(datagram),
    )
    if is_empty:
        return CoapUdpDatagram(message_type, 0, 0, message_id, b"", (), b"")

    offset = _HEADER_BYTES
    token_end = offset + token_length
    if token_end > len(datagram):
        raise CoapDatagramError("datagram ends inside its token")
    token = datagram[offset:token_end]
    offset = token_end

    options: list[CoapOption] = []
    previous_option_number = 0
    option_value_bytes = 0
    payload = b""

    while offset < len(datagram):
        option_header = datagram[offset]
        offset += 1
        if option_header == _PAYLOAD_MARKER:
            payload = datagram[offset:]
            if not payload:
                raise CoapDatagramError("payload marker must precede a payload")
            if len(payload) > _MAX_PAYLOAD_BYTES:
                raise CoapDatagramError("payload exceeds 1,024 bytes")
            offset = len(datagram)
            break

        if len(options) == _MAX_OPTIONS:
            raise CoapDatagramError("option count exceeds 32")
        option_delta, offset = _decode_option_argument(
            option_header >> 4,
            datagram,
            offset,
            field="delta",
        )
        option_length, offset = _decode_option_argument(
            option_header & 0b1111,
            datagram,
            offset,
            field="length",
        )
        option_number = previous_option_number + option_delta
        if option_number > _MAX_OPTION_NUMBER:
            raise CoapDatagramError("cumulative option number exceeds 65,535")
        if option_length > _MAX_OPTION_VALUE_BYTES - option_value_bytes:
            raise CoapDatagramError("cumulative option values exceed 1,024 bytes")

        value_end = offset + option_length
        if value_end > len(datagram):
            raise CoapDatagramError("datagram ends inside an option value")
        options.append(CoapOption(option_number, datagram[offset:value_end]))
        option_value_bytes += option_length
        previous_option_number = option_number
        offset = value_end

    return CoapUdpDatagram(
        message_type,
        code_class,
        code_detail,
        message_id,
        token,
        tuple(options),
        payload,
    )
```

## Example

```python
# RFC 7252 Appendix A, Figure 16: GET /temperature and its response.
published_get = bytes.fromhex("40017d34bb74656d7065726174757265")
published_response = bytes.fromhex("60457d34ff32322e332043")

request = parse_coap_udp_datagram(published_get)
response = parse_coap_udp_datagram(published_response)

assert request == CoapUdpDatagram(
    CoapMessageType.CONFIRMABLE,
    0,
    1,
    0x7D34,
    b"",
    (CoapOption(11, b"temperature"),),
    b"",
)
assert response == CoapUdpDatagram(
    CoapMessageType.ACKNOWLEDGEMENT,
    2,
    5,
    0x7D34,
    b"",
    (),
    b"22.3 C",
)

# Delta 13, a repeated option, delta 269, and 0xFF inside an option value.
constructed = bytes.fromhex(
    "41011234a5"  # CON GET, token a5
    "d100ff"  # option 13, value ff
    "026f6b"  # repeated option 13, value 'ok'
    "e00000"  # option 282, empty value
    "ff626f6479"  # payload 'body'
)
parsed = parse_coap_udp_datagram(constructed)
assert parsed.token == b"\xa5"
assert parsed.options == (
    CoapOption(13, b"\xff"),
    CoapOption(13, b"ok"),
    CoapOption(282, b""),
)
assert parsed.payload == b"body"

empty_types = tuple(
    parse_coap_udp_datagram(bytes((first, 0, 0xCA, 0xFE))).message_type
    for first in (0x40, 0x60, 0x70)
)
accepted_shapes = tuple(
    parse_coap_udp_datagram(bytes.fromhex(value))
    for value in (
        "40010001",  # CON request
        "50010001",  # NON request
        "40450001",  # CON response
        "50450001",  # NON response
        "60450001",  # ACK response
    )
)

maximum_option_number = parse_coap_udp_datagram(bytes.fromhex("40010001e0fef2")).options[0].number
maximum_option_value = (
    parse_coap_udp_datagram(b"\x40\x01\x00\x01\x0e\x02\xf3" + b"v" * _MAX_OPTION_VALUE_BYTES)
    .options[0]
    .value
)
maximum_payload = parse_coap_udp_datagram(
    b"\x40\x45\x00\x01\xff" + b"p" * _MAX_PAYLOAD_BYTES
).payload


def is_rejected(candidate: object) -> bool:
    try:
        parse_coap_udp_datagram(candidate)  # type: ignore[arg-type]
    except (TypeError, ValueError):
        return True
    return False


rejected = (
    b"\x40\x01\x00",  # shorter than the header
    b"\x00\x01\x00\x01",  # version zero
    b"\x49\x01\x00\x01",  # reserved token length 9
    b"\x48\x01\x00\x01",  # truncated eight-byte token
    b"\x40\x60\x00\x01",  # reserved code class 3
    b"\x50\x00\x00\x01",  # empty NON
    b"\x60\x01\x00\x01",  # request in ACK
    b"\x70\x45\x00\x01",  # response in RST
    b"\x40\x00\x00\x01\x00",  # non-header bytes in an empty message
    b"\x40\x01\x00\x01\xff",  # marker without payload
    b"\x40\x01\x00\x01\xf0",  # reserved delta nibble
    b"\x40\x01\x00\x01\x02x",  # truncated option value
    b"\x40\x01\x00\x01\xe0\xff\xff",  # option number above 65,535
    b"\x40\x01\x00\x01" + b"\x00" * (_MAX_OPTIONS + 1),
    b"\x40\x01\x00\x01\x0e\x02\xf4" + b"v" * (_MAX_OPTION_VALUE_BYTES + 1),
    b"\x40\x45\x00\x01\xff" + b"p" * (_MAX_PAYLOAD_BYTES + 1),
    b"\x40\x01\x00\x01" + b"\x00" * (_MAX_DATAGRAM_BYTES - 3),
    bytearray(published_get),
)

assert empty_types == (
    CoapMessageType.CONFIRMABLE,
    CoapMessageType.ACKNOWLEDGEMENT,
    CoapMessageType.RESET,
)
assert len(accepted_shapes) == 5
assert maximum_option_number == _MAX_OPTION_NUMBER
assert len(maximum_option_value) == _MAX_OPTION_VALUE_BYTES
assert len(maximum_payload) == _MAX_PAYLOAD_BYTES
assert all(is_rejected(candidate) for candidate in rejected)
```

## Trade-offs and Limitations

Parsing is linear in the datagram length and retains at most 32 immutable
option records plus the token and payload slices. Repeated options remain in
wire order, a delta of zero remains observable, and no mapping silently drops
an occurrence. The complete input is required; this is not an incremental
stream decoder or a UDP receive loop.

The parser checks the generic envelope and the base message-type matrix, not
registered Code or Option semantics. An endpoint must still reject an unknown
critical option, validate each known option's length and repeatability, and
apply the correct method or response rules. A proxy must separately understand
unknown unsafe options before forwarding them.

UDP can lose, duplicate, reorder, or spoof datagrams. Callers remain
responsible for endpoint-bound Message ID reuse rules, token uniqueness and
randomness, replay handling, retransmission deadlines, amplification limits,
and transport or object security. Successfully parsing bytes grants no
authority to process their payload.

## Related Snippets

<!-- catalog:related:start -->
- [Parse One Bounded HTTP/2 SETTINGS Frame](parse-one-bounded-http-2-settings-frame.md)
- [Parse One Bounded PROXY Protocol Version Two TCP Header Without TLVs](parse-one-bounded-proxy-protocol-version-two-tcp-header-without-tlvs.md)
- [Decode One Bounded HTTP/1.1 Chunked Body under a Closed No-Trailers Profile](decode-one-bounded-http-1-1-chunked-body-under-a-closed-no-trailers-profile.md)
<!-- catalog:related:end -->
