---
title: "Parse One Checksum-Valid Bounded IPv4 Packet"
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
  - compute-a-16-bit-internet-checksum-across-bounded-byte-segments.md
  - reassemble-one-bounded-ipv4-payload-from-decoded-fragments.md
  - parse-one-bounded-proxy-protocol-version-two-tcp-header-without-tlvs.md
  - classify-a-pre-resolved-ip-against-a-bounded-cidr-zone-snapshot.md
---

# Parse One Checksum-Valid Bounded IPv4 Packet

## Idea and Problem

Parse one complete IPv4 packet into immutable fields while enforcing its structural byte boundaries.

The four-bit Internet Header Length counts 32-bit words, the 16-bit total
length covers both header and payload, and the three flag bits share a word
with the 13-bit fragment offset. The header is accepted only when its
one's-complement checksum, including the stored checksum field, folds to all
ones. The returned fragment offset remains in its encoded eight-byte units.

The Identification field is preserved without assigning it stronger meaning.
In particular, RFC 6864 does not make it a unique datagram identifier for an
atomic datagram with `DF` set, `MF` clear, and offset zero.

## When to Use

Use this parser for one already isolated packet image in bounded protocol
fixtures, offline inspection, capture adapters, or a deliberately small
decoder that needs native `IPv4Address` values. It accepts fragmented packets
so that grouping and reassembly can remain a separate policy with its own
resource limits and overlap rules.

Use the operating-system IP stack or a maintained packet-processing library
for live traffic. Incremental capture framing, reassembly, option semantics,
transport checksums, routing policy, authentication, and normalization across
encapsulation layers are outside this parser.

## Implementation

```python
from dataclasses import dataclass
from ipaddress import IPv4Address

_MIN_IPV4_PACKET_BYTES = 20
_MAX_IPV4_PACKET_BYTES = 65_535
_MIN_IPV4_IHL_WORDS = 5
_MAX_IPV4_IHL_WORDS = 15
_IPV4_RESERVED_FLAG = 1 << 15
_IPV4_DONT_FRAGMENT_FLAG = 1 << 14
_IPV4_MORE_FRAGMENTS_FLAG = 1 << 13
_IPV4_FRAGMENT_OFFSET_MASK = (1 << 13) - 1


class IPv4PacketError(ValueError):
    """Raised when bytes are not one complete checksum-valid IPv4 packet."""


@dataclass(frozen=True, slots=True)
class IPv4Packet:
    dscp: int
    ecn: int
    identification: int
    dont_fragment: bool
    more_fragments: bool
    fragment_offset: int
    ttl: int
    protocol: int
    checksum: int
    source: IPv4Address
    destination: IPv4Address
    options: bytes
    payload: bytes


def _has_valid_ipv4_header_checksum(header: bytes) -> bool:
    word_sum = sum(
        (header[offset] << 8) | header[offset + 1] for offset in range(0, len(header), 2)
    )
    while word_sum >> 16:
        word_sum = (word_sum & 0xFFFF) + (word_sum >> 16)
    return word_sum == 0xFFFF


def parse_ipv4_packet(packet: bytes) -> IPv4Packet:
    """Parse exactly one complete checksum-valid IPv4 packet."""
    if type(packet) is not bytes:
        raise TypeError("packet must be exact immutable bytes")
    if not _MIN_IPV4_PACKET_BYTES <= len(packet) <= _MAX_IPV4_PACKET_BYTES:
        raise IPv4PacketError("packet length is outside the supported IPv4 range")

    version_ihl = packet[0]
    if version_ihl >> 4 != 4:
        raise IPv4PacketError("packet does not use IPv4")

    ihl_words = version_ihl & 0x0F
    if not _MIN_IPV4_IHL_WORDS <= ihl_words <= _MAX_IPV4_IHL_WORDS:
        raise IPv4PacketError("Internet Header Length is outside the IPv4 range")
    header_length = ihl_words * 4
    if header_length > len(packet):
        raise IPv4PacketError("packet is shorter than its declared IPv4 header")

    total_length = int.from_bytes(packet[2:4], "big")
    if total_length < header_length:
        raise IPv4PacketError("total length is shorter than the IPv4 header")
    if total_length != len(packet):
        raise IPv4PacketError("total length does not match the complete packet bytes")

    flags_and_offset = int.from_bytes(packet[6:8], "big")
    if flags_and_offset & _IPV4_RESERVED_FLAG:
        raise IPv4PacketError("the reserved IPv4 flag must be zero")

    header = packet[:header_length]
    if not _has_valid_ipv4_header_checksum(header):
        raise IPv4PacketError("IPv4 header checksum is invalid")

    traffic_class = packet[1]
    return IPv4Packet(
        dscp=traffic_class >> 2,
        ecn=traffic_class & 0x03,
        identification=int.from_bytes(packet[4:6], "big"),
        dont_fragment=bool(flags_and_offset & _IPV4_DONT_FRAGMENT_FLAG),
        more_fragments=bool(flags_and_offset & _IPV4_MORE_FRAGMENTS_FLAG),
        fragment_offset=flags_and_offset & _IPV4_FRAGMENT_OFFSET_MASK,
        ttl=packet[8],
        protocol=packet[9],
        checksum=int.from_bytes(packet[10:12], "big"),
        source=IPv4Address(packet[12:16]),
        destination=IPv4Address(packet[16:20]),
        options=packet[20:header_length],
        payload=packet[header_length:],
    )
```

## Example

```python
def reference_header_checksum(header_with_zero_checksum: bytes) -> int:
    assert len(header_with_zero_checksum) % 2 == 0
    word_sum = sum(
        int.from_bytes(header_with_zero_checksum[offset : offset + 2], "big")
        for offset in range(0, len(header_with_zero_checksum), 2)
    )
    while word_sum > 0xFFFF:
        word_sum = (word_sum >> 16) + (word_sum & 0xFFFF)
    return word_sum ^ 0xFFFF


def build_ipv4_packet(
    *,
    dscp: int = 0,
    ecn: int = 0,
    identification: int = 0,
    flag_bits: int = 0,
    fragment_offset: int = 0,
    ttl: int = 64,
    protocol: int = 17,
    source: str = "192.0.2.1",
    destination: str = "198.51.100.9",
    options: bytes = b"",
    payload: bytes = b"",
) -> bytes:
    assert len(options) <= 40 and len(options) % 4 == 0
    assert 0 <= dscp < 64 and 0 <= ecn < 4
    assert 0 <= identification <= 0xFFFF
    assert 0 <= flag_bits < 8 and 0 <= fragment_offset < (1 << 13)
    assert 0 <= ttl <= 0xFF and 0 <= protocol <= 0xFF

    header_length = 20 + len(options)
    total_length = header_length + len(payload)
    assert total_length <= 65_535
    header = bytearray(header_length)
    header[0] = (4 << 4) | (header_length // 4)
    header[1] = (dscp << 2) | ecn
    header[2:4] = total_length.to_bytes(2, "big")
    header[4:6] = identification.to_bytes(2, "big")
    header[6:8] = ((flag_bits << 13) | fragment_offset).to_bytes(2, "big")
    header[8] = ttl
    header[9] = protocol
    header[12:16] = IPv4Address(source).packed
    header[16:20] = IPv4Address(destination).packed
    header[20:] = options
    header[10:12] = reference_header_checksum(bytes(header)).to_bytes(2, "big")
    return bytes(header) + payload


minimum_bytes = build_ipv4_packet()
minimum_packet = parse_ipv4_packet(minimum_bytes)
assert len(minimum_bytes) == 20
assert minimum_packet.options == minimum_packet.payload == b""

plain_bytes = build_ipv4_packet(
    dscp=46,
    ecn=3,
    identification=0x1234,
    flag_bits=0b010,
    protocol=6,
    payload=b"hello",
)
plain = parse_ipv4_packet(plain_bytes)
assert plain == IPv4Packet(
    dscp=46,
    ecn=3,
    identification=0x1234,
    dont_fragment=True,
    more_fragments=False,
    fragment_offset=0,
    ttl=64,
    protocol=6,
    checksum=int.from_bytes(plain_bytes[10:12], "big"),
    source=IPv4Address("192.0.2.1"),
    destination=IPv4Address("198.51.100.9"),
    options=b"",
    payload=b"hello",
)

maximum_options = bytes(range(40))
option_packet = parse_ipv4_packet(
    build_ipv4_packet(options=maximum_options, payload=b"option payload")
)
assert option_packet.options == maximum_options
assert option_packet.payload == b"option payload"

first_fragment = parse_ipv4_packet(
    build_ipv4_packet(
        identification=7,
        flag_bits=0b001,
        fragment_offset=0,
        payload=b"12345678abcdefgh",
    )
)
last_fragment = parse_ipv4_packet(
    build_ipv4_packet(
        identification=7,
        fragment_offset=2,
        payload=b"tail",
    )
)
assert first_fragment.more_fragments and first_fragment.fragment_offset == 0
assert not last_fragment.more_fragments and last_fragment.fragment_offset == 2

unusual_fragment = parse_ipv4_packet(
    build_ipv4_packet(flag_bits=0b011, fragment_offset=7, payload=b"fragment")
)
assert unusual_fragment.dont_fragment and unusual_fragment.more_fragments
assert unusual_fragment.fragment_offset == 7

maximum_bytes = build_ipv4_packet(payload=b"x" * (65_535 - 20))
maximum_packet = parse_ipv4_packet(maximum_bytes)
assert len(maximum_bytes) == 65_535
assert len(maximum_packet.payload) == 65_535 - 20

payload_mutation = bytearray(plain_bytes)
payload_mutation[-1] ^= 1
assert parse_ipv4_packet(bytes(payload_mutation)).payload == b"helln"

header_corruption = bytearray(plain_bytes)
header_corruption[8] ^= 1
invalid_version = bytes([0x65]) + plain_bytes[1:]
invalid_ihl = bytes([0x44]) + plain_bytes[1:]
unavailable_header = bytes([0x4F]) + plain_bytes[1:]
short_total = bytearray(plain_bytes)
short_total[2:4] = (19).to_bytes(2, "big")
reserved_flag = build_ipv4_packet(flag_bits=0b100)


def rejected_as(candidate: object, expected_error: type[Exception]) -> bool:
    try:
        parse_ipv4_packet(candidate)  # type: ignore[arg-type]
    except expected_error:
        return True
    return False


invalid_cases = (
    (bytearray(plain_bytes), TypeError),
    (b"\x00" * 19, IPv4PacketError),
    (b"\x00" * 65_536, IPv4PacketError),
    (plain_bytes[:-1], IPv4PacketError),
    (plain_bytes + b"\x00", IPv4PacketError),
    (invalid_version, IPv4PacketError),
    (invalid_ihl, IPv4PacketError),
    (unavailable_header, IPv4PacketError),
    (bytes(short_total), IPv4PacketError),
    (reserved_flag, IPv4PacketError),
    (bytes(header_corruption), IPv4PacketError),
)
assert all(rejected_as(candidate, error) for candidate, error in invalid_cases)
```

## Trade-offs and Limitations

Validation scans at most the 60-byte IPv4 header. Returning `options` and
`payload` slices takes `O(N)` time and memory for an `N`-byte packet; all other
work is bounded by the header size. The exact-length rule is intentionally
strict: callers that receive link-layer padding or multiple packets must
isolate the declared packet bytes before calling this function.

This parser does not interpret options, reject unusual but structurally
representable flag combinations, reassemble fragments, or decide whether an
address, TTL, protocol number, DSCP, ECN, or Identification value is acceptable
for a particular application. The fragment offset is returned in eight-byte
units rather than converted to a byte position.

The IPv4 checksum covers only the header. It detects some accidental changes
but is not collision resistant, does not authenticate the sender, and says
nothing about payload integrity, as the payload-mutation example demonstrates.

## Related Snippets

<!-- catalog:related:start -->
- [Compute a 16-Bit Internet Checksum Across Bounded Byte Segments](compute-a-16-bit-internet-checksum-across-bounded-byte-segments.md)
- [Reassemble One Bounded IPv4 Payload from Decoded Fragments](reassemble-one-bounded-ipv4-payload-from-decoded-fragments.md)
- [Parse One Bounded PROXY Protocol Version Two TCP Header Without TLVs](parse-one-bounded-proxy-protocol-version-two-tcp-header-without-tlvs.md)
- [Classify a Pre-Resolved IP Against a Bounded CIDR-Zone Snapshot](classify-a-pre-resolved-ip-against-a-bounded-cidr-zone-snapshot.md)
<!-- catalog:related:end -->
