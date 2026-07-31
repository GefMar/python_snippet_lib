---
title: "Split One Bounded IPFIX v10 Message into Raw Set Envelopes"
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
  - parse-one-bounded-classic-pcap-version-2-4-capture.md
  - parse-one-checksum-valid-bounded-ipv4-packet.md
  - unwrap-one-uint32-serial-around-an-explicit-absolute-reference.md
---

# Split One Bounded IPFIX v10 Message into Raw Set Envelopes

## Idea and Problem

Split exactly one length-consistent IPFIX version 10 message into immutable raw Set envelopes without guessing record or padding boundaries.

[RFC 7011](https://www.rfc-editor.org/rfc/rfc7011) defines a 16-byte
network-order message header followed by zero or more Sets. The message and
each Set carry their own inclusive byte length. Set IDs `0` and `1` are unused,
`2` and `3` identify Template and Options Template Sets, `4..255` are currently
unassigned in the [IANA registry](https://www.iana.org/assignments/ipfix/ipfix.xhtml#ipfix-set-ids),
and `256..65535` identify Data Sets.

Record boundaries inside a Set depend on templates and variable-length field
encodings that this parser does not own. It therefore returns each complete
post-header body unchanged, including any padding, while rejecting lengths
that escape their enclosing message.

## When to Use

Use this parser at a bounded capture/import boundary, in protocol fixtures, or
before dispatching raw Sets to a stateful IPFIX template decoder. It provides
a deterministic outer framing check without silently treating opaque record
bytes as one invented schema.

Use a maintained IPFIX collector when templates, withdrawals, transport
sessions, record fields, or sequence-based loss detection matter. Flow data
can be private or identifying; successful framing does not make returned Set
bodies safe to log, retain, or publish.

## Implementation

```python
from dataclasses import dataclass
from struct import Struct

_IPFIX_VERSION = 10
_IPFIX_MESSAGE_HEADER_BYTES = 16
_IPFIX_SET_HEADER_BYTES = 4
_MAX_IPFIX_MESSAGE_BYTES = (1 << 16) - 1
_IPFIX_MESSAGE_HEADER = Struct("!HHIII")
_IPFIX_SET_HEADER = Struct("!HH")


class IpfixMessageError(ValueError):
    """Raised when bytes violate the supported IPFIX v10 outer framing."""


@dataclass(frozen=True, slots=True)
class IpfixRawSet:
    set_id: int
    body: bytes


@dataclass(frozen=True, slots=True)
class IpfixMessage:
    export_time: int
    sequence_number: int
    observation_domain_id: int
    sets: tuple[IpfixRawSet, ...]


def _is_current_ipfix_set_id(set_id: int) -> bool:
    return set_id in (2, 3) or set_id >= 256


def parse_ipfix_v10_message(encoded: bytes) -> IpfixMessage:
    """Split one complete IPFIX v10 message into raw Set envelopes."""
    if type(encoded) is not bytes:
        raise TypeError("encoded must be exact immutable bytes")
    if not _IPFIX_MESSAGE_HEADER_BYTES <= len(encoded) <= _MAX_IPFIX_MESSAGE_BYTES:
        raise IpfixMessageError("message length is outside 16..65,535 bytes")

    (
        version,
        declared_length,
        export_time,
        sequence_number,
        observation_domain_id,
    ) = _IPFIX_MESSAGE_HEADER.unpack_from(encoded)

    if version != _IPFIX_VERSION:
        raise IpfixMessageError("only IPFIX version 10 is supported")
    if declared_length != len(encoded):
        raise IpfixMessageError("declared message length must match the exact input")

    sets: list[IpfixRawSet] = []
    offset = _IPFIX_MESSAGE_HEADER_BYTES

    while offset < len(encoded):
        if offset + _IPFIX_SET_HEADER_BYTES > len(encoded):
            raise IpfixMessageError("message ends inside a Set header")

        set_id, set_length = _IPFIX_SET_HEADER.unpack_from(encoded, offset)
        if not _is_current_ipfix_set_id(set_id):
            raise IpfixMessageError("Set ID is not assigned by this IPFIX v10 profile")
        if set_length < _IPFIX_SET_HEADER_BYTES:
            raise IpfixMessageError("Set length is smaller than its four-byte header")

        set_end = offset + set_length
        if set_end > len(encoded):
            raise IpfixMessageError("Set length escapes the enclosing message")

        sets.append(
            IpfixRawSet(
                set_id=set_id,
                body=encoded[offset + _IPFIX_SET_HEADER_BYTES : set_end],
            )
        )
        offset = set_end

    return IpfixMessage(
        export_time=export_time,
        sequence_number=sequence_number,
        observation_domain_id=observation_domain_id,
        sets=tuple(sets),
    )
```

## Example

```python
def make_raw_set(set_id: int, body: bytes) -> bytes:
    """Build a Set envelope with manual network-order integer encoding."""
    set_length = 4 + len(body)
    if not 0 <= set_id <= 0xFFFF:
        raise ValueError("fixture Set ID must fit 16 bits")
    if set_length > 0xFFFF:
        raise ValueError("fixture Set length must fit 16 bits")
    return set_id.to_bytes(2, "big") + set_length.to_bytes(2, "big") + body


def make_ipfix_message(
    *raw_sets: bytes,
    version: int = 10,
    export_time: int = 1_700_000_000,
    sequence_number: int = 0,
    observation_domain_id: int = 1,
) -> bytes:
    """Build a message without using the production Struct objects."""
    body = b"".join(raw_sets)
    message_length = 16 + len(body)
    if message_length > 0xFFFF:
        raise ValueError("fixture message length must fit 16 bits")
    return (
        version.to_bytes(2, "big")
        + message_length.to_bytes(2, "big")
        + export_time.to_bytes(4, "big")
        + sequence_number.to_bytes(4, "big")
        + observation_domain_id.to_bytes(4, "big")
        + body
    )


def replace_u16(encoded: bytes, offset: int, value: int) -> bytes:
    return encoded[:offset] + value.to_bytes(2, "big") + encoded[offset + 2 :]


def rejected_as(candidate: object, expected_error: type[Exception]) -> bool:
    try:
        parse_ipfix_v10_message(candidate)  # type: ignore[arg-type]
    except expected_error:
        return True
    return False


header_only = make_ipfix_message(
    export_time=0,
    sequence_number=0xFFFF_FFFF,
    observation_domain_id=0,
)
assert len(header_only) == 16
assert parse_ipfix_v10_message(header_only) == IpfixMessage(
    export_time=0,
    sequence_number=0xFFFF_FFFF,
    observation_domain_id=0,
    sets=(),
)

template_record = (
    (256).to_bytes(2, "big")
    + (1).to_bytes(2, "big")
    + (8).to_bytes(2, "big")
    + (4).to_bytes(2, "big")
)
options_record = (
    (257).to_bytes(2, "big")
    + (2).to_bytes(2, "big")
    + (1).to_bytes(2, "big")
    + (149).to_bytes(2, "big")
    + (4).to_bytes(2, "big")
    + (1).to_bytes(2, "big")
    + (8).to_bytes(2, "big")
)
mixed_bytes = make_ipfix_message(
    make_raw_set(2, template_record),
    make_raw_set(3, options_record),
    make_raw_set(256, b"\xc0\x00\x02\x01"),
    make_raw_set(65_535, b"opaque\x00\x00"),
    export_time=1_700_000_123,
    sequence_number=42,
    observation_domain_id=7,
)
mixed = parse_ipfix_v10_message(mixed_bytes)
assert mixed == IpfixMessage(
    export_time=1_700_000_123,
    sequence_number=42,
    observation_domain_id=7,
    sets=(
        IpfixRawSet(2, template_record),
        IpfixRawSet(3, options_record),
        IpfixRawSet(256, b"\xc0\x00\x02\x01"),
        IpfixRawSet(65_535, b"opaque\x00\x00"),
    ),
)

maximum_body = b"x" * 65_515
maximum_message = make_ipfix_message(make_raw_set(256, maximum_body))
assert len(maximum_message) == 65_535
maximum_parsed = parse_ipfix_v10_message(maximum_message)
assert maximum_parsed.sets == (IpfixRawSet(256, maximum_body),)

empty_raw_set = make_raw_set(256, b"")
maximum_set_count_message = make_ipfix_message(
    *(empty_raw_set for _ in range(16_379)),
)
assert len(maximum_set_count_message) == 65_532
assert len(parse_ipfix_v10_message(maximum_set_count_message).sets) == 16_379

wrong_version = make_ipfix_message(version=9)
declared_short = replace_u16(mixed_bytes, 2, len(mixed_bytes) - 1)
declared_long = replace_u16(mixed_bytes, 2, len(mixed_bytes) + 1)
partial_set_header = make_ipfix_message(b"\x01")
short_set = make_ipfix_message(b"\x01\x00\x00\x03")
escaping_set = make_ipfix_message(b"\x01\x00\x00\x08x")
concatenated = header_only + header_only

invalid_cases = (
    (bytearray(header_only), TypeError),
    (bytes(15), IpfixMessageError),
    (bytes(65_536), IpfixMessageError),
    (wrong_version, IpfixMessageError),
    (declared_short, IpfixMessageError),
    (declared_long, IpfixMessageError),
    (partial_set_header, IpfixMessageError),
    (short_set, IpfixMessageError),
    (escaping_set, IpfixMessageError),
    (concatenated, IpfixMessageError),
    (make_ipfix_message(make_raw_set(0, b"")), IpfixMessageError),
    (make_ipfix_message(make_raw_set(1, b"")), IpfixMessageError),
    (make_ipfix_message(make_raw_set(4, b"")), IpfixMessageError),
    (make_ipfix_message(make_raw_set(255, b"")), IpfixMessageError),
)
rejected = sum(
    rejected_as(candidate, expected_error) for candidate, expected_error in invalid_cases
)

assert rejected == len(invalid_cases) == 14 and len(mixed.sets) == 4
```

## Trade-offs and Limitations

Parsing takes `O(N)` time and memory for an `N`-byte message because the raw
Set bodies are copied. The protocol's 16-bit message length inherently caps
one exact message at 65,535 bytes and at most 16,379 four-byte Set envelopes;
there is no additional local size or count policy.

The returned body begins immediately after its Set header and can contain
records, optional padding, or bytes that are semantically invalid. Without an
applicable Template, the parser cannot safely separate records from padding,
validate field encodings, or even prove that a Data Set contains one complete
record. Template and Options Template bodies are likewise not decoded.

The accepted non-Data Set IDs follow the current IANA registry. If a future
Standards Action assigns an ID in `4..255`, this closed profile must be reviewed
and updated before accepting it.

The function accepts one exact message, not a UDP datagram containing another
protocol or a TCP/SCTP stream containing multiple messages. It does not manage
template lifetimes or withdrawals, interpret Export Time across wraparound,
track sequence numbers, detect loss, authenticate exporters, or apply privacy
policy to flow data.

## Related Snippets

<!-- catalog:related:start -->
- [Parse One Bounded Classic PCAP Version 2.4 Capture](parse-one-bounded-classic-pcap-version-2-4-capture.md)
- [Parse One Checksum-Valid Bounded IPv4 Packet](parse-one-checksum-valid-bounded-ipv4-packet.md)
- [Unwrap One uint32 Serial Around an Explicit Absolute Reference](unwrap-one-uint32-serial-around-an-explicit-absolute-reference.md)
<!-- catalog:related:end -->
