---
title: "Parse One Bounded RFC 8489 STUN Binding Request Envelope"
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
  - parse-one-bounded-proxy-protocol-version-two-tcp-header-without-tlvs.md
  - parse-one-complete-unfragmented-websocket-data-frame-under-role-masking-rules.md
  - compute-a-16-bit-internet-checksum-across-bounded-byte-segments.md
---

# Parse One Bounded RFC 8489 STUN Binding Request Envelope

## Idea and Problem

Parse one complete STUN Binding request into its transaction ID and ordered raw attributes without claiming to process the protocol.

An RFC 8489 message has a fixed 20-byte header followed by four-byte-aligned
type-length-value attributes. The header's type, declared body length, magic
cookie, and transaction ID establish the structural envelope. Attribute
padding is not part of its value and receivers must ignore its contents.

The parser deliberately preserves duplicate and unknown attribute types. Their
meaning belongs to the STUN usage and attribute registry, not to generic wire
framing.

## When to Use

Use this recipe for bounded packet inspection, protocol fixtures, or the first
syntax boundary of a component that expects exactly one already-framed STUN
Binding request. The caller must already own the datagram boundary or have
framed one message from a stream.

Use a maintained STUN, ICE, or TURN implementation for network traversal and
production request handling. Structural success does not authenticate a peer,
validate an attribute, authorize an operation, or establish that a packet is
safe to act upon.

## Implementation

```python
from dataclasses import dataclass
from struct import pack

_STUN_HEADER_BYTES = 20
_STUN_BINDING_REQUEST = 0x0001
_STUN_MAGIC_COOKIE = 0x2112_A442
_MAX_STUN_MESSAGE_BYTES = 65_552
_MAX_ATTRIBUTES = 256


class StunEnvelopeError(ValueError):
    """The bytes are outside the bounded Binding-request envelope."""


@dataclass(frozen=True, slots=True)
class StunAttribute:
    type_code: int
    comprehension_required: bool
    value: bytes


@dataclass(frozen=True, slots=True)
class StunBindingRequest:
    transaction_id: bytes
    attributes: tuple[StunAttribute, ...]


def parse_stun_binding_request(message: bytes) -> StunBindingRequest:
    """Parse one exact RFC 8489 Binding-request envelope."""
    if type(message) is not bytes:
        raise TypeError("message must be exact bytes")
    if not _STUN_HEADER_BYTES <= len(message) <= _MAX_STUN_MESSAGE_BYTES:
        raise StunEnvelopeError("message length is outside the supported range")

    message_type = int.from_bytes(message[0:2], "big")
    if message_type & 0xC000:
        raise StunEnvelopeError("the two most-significant type bits must be zero")
    if message_type != _STUN_BINDING_REQUEST:
        raise StunEnvelopeError("message type is not a Binding request")

    declared_body_length = int.from_bytes(message[2:4], "big")
    if declared_body_length & 0b11:
        raise StunEnvelopeError("declared body length is not four-byte aligned")
    if declared_body_length != len(message) - _STUN_HEADER_BYTES:
        raise StunEnvelopeError("declared body length does not match the message")
    if int.from_bytes(message[4:8], "big") != _STUN_MAGIC_COOKIE:
        raise StunEnvelopeError("magic cookie is invalid")

    transaction_id = message[8:20]
    attributes: list[StunAttribute] = []
    offset = _STUN_HEADER_BYTES
    while offset < len(message):
        if len(attributes) >= _MAX_ATTRIBUTES:
            raise StunEnvelopeError("attribute count exceeds the supported limit")
        if len(message) - offset < 4:
            raise StunEnvelopeError("attribute header is truncated")

        type_code = int.from_bytes(message[offset : offset + 2], "big")
        value_length = int.from_bytes(message[offset + 2 : offset + 4], "big")
        value_start = offset + 4
        value_end = value_start + value_length
        padding_length = (-value_length) % 4
        record_end = value_end + padding_length
        if record_end > len(message):
            raise StunEnvelopeError("attribute value or padding is truncated")

        attributes.append(
            StunAttribute(
                type_code=type_code,
                comprehension_required=type_code < 0x8000,
                value=message[value_start:value_end],
            )
        )
        offset = record_end

    if offset != len(message):
        raise AssertionError("aligned traversal must consume the declared body")
    return StunBindingRequest(
        transaction_id=transaction_id,
        attributes=tuple(attributes),
    )
```

## Example

```python


transaction_id = b"0123456789ab"
body = b"".join(
    (
        pack("!HH", 0x8022, 3) + b"one" + b"\xff",
        pack("!HH", 0x0006, 0),
        pack("!HH", 0x8022, 3) + b"two" + b"\x00",
    )
)
packet = (
    pack(
        "!HHI12s",
        _STUN_BINDING_REQUEST,
        len(body),
        _STUN_MAGIC_COOKIE,
        transaction_id,
    )
    + body
)

parsed = parse_stun_binding_request(packet)

assert parsed == StunBindingRequest(
    transaction_id=transaction_id,
    attributes=(
        StunAttribute(0x8022, False, b"one"),
        StunAttribute(0x0006, True, b""),
        StunAttribute(0x8022, False, b"two"),
    ),
)

try:
    parse_stun_binding_request(packet[:-1])
except StunEnvelopeError:
    truncation_rejected = True
else:
    truncation_rejected = False

wrong_cookie = packet[:4] + b"\x00\x00\x00\x00" + packet[8:]
try:
    parse_stun_binding_request(wrong_cookie)
except StunEnvelopeError:
    cookie_rejected = True
else:
    cookie_rejected = False

assert truncation_rejected and cookie_rejected
```

## Trade-offs and Limitations

Parsing takes `O(n + a)` time for `n` admitted bytes and `a` attributes. The
result copies each raw value, so its retained memory is `O(n + a)`. The total
message and attribute-count caps prevent a packet filled with zero-length
attributes from producing an unbounded object graph.

Non-zero alignment bytes are intentionally ignored rather than rejected.
Attribute order, duplicates, and unknown type codes are preserved because RFC
8489 permits duplicates and assigns their processing to higher layers. The
`comprehension_required` flag reports only the type-code range; it does not say
whether this implementation understands that attribute.

The function accepts only Binding requests and does not decode attribute
values, consult a registry, check placement rules, verify `FINGERPRINT` or
`MESSAGE-INTEGRITY`, derive addresses, authenticate credentials, run ICE or
TURN, retransmit, apply timers, frame TCP/TLS, or perform network I/O. A
successfully parsed request remains untrusted input.

## Related Snippets

<!-- catalog:related:start -->
- [Parse One Bounded PROXY Protocol Version Two TCP Header Without TLVs](parse-one-bounded-proxy-protocol-version-two-tcp-header-without-tlvs.md)
- [Parse One Complete Unfragmented WebSocket Data Frame Under Role Masking Rules](parse-one-complete-unfragmented-websocket-data-frame-under-role-masking-rules.md)
- [Compute a 16-Bit Internet Checksum Across Bounded Byte Segments](compute-a-16-bit-internet-checksum-across-bounded-byte-segments.md)
<!-- catalog:related:end -->
