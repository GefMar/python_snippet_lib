---
title: "Parse One Bounded PROXY Protocol Version Two TCP Header Without TLVs"
snippet_type: recipe
use_cases:
  - networking
  - parsing
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - parse-one-bounded-proxy-protocol-version-one-line.md
  - parse-a-bounded-host-and-port-with-bracketed-ipv6.md
  - collapse-ipv4-mapped-ipv6-literals-to-one-canonical-address-key.md
---

# Parse One Bounded PROXY Protocol Version Two TCP Header Without TLVs

## Idea and Problem

Parse one isolated PROXY protocol version 2 binary header under an exact TCP-over-IPv4-or-IPv6 profile that admits no TLV or trailing bytes.

The fixed signature, version-command byte, family-protocol byte, and declared
address length select one of two packed network-byte-order layouts. Requiring
the complete input length to equal that layout prevents optional metadata or
application data from being mistaken for part of this deliberately narrow
result.

## When to Use

Use this parser only on a dedicated PROXY-protocol listener after an
authenticated or explicitly allowlisted transport peer has supplied exactly
one already-isolated header. Retain the actual socket peer identity separately:
the advertised source and destination are assertions made by that trusted
peer, not facts authenticated by this parser.

Select this profile through listener configuration rather than by guessing from
arbitrary application traffic. Use a maintained protocol implementation when
socket reads, deadlines, incremental framing, `LOCAL`, datagrams, Unix sockets,
TLVs, or general version 2 interoperability are required.

## Implementation

```python
from dataclasses import dataclass
from enum import StrEnum
from ipaddress import IPv4Address, IPv6Address

_PROXY_V2_SIGNATURE = b"\r\n\r\n\x00\r\nQUIT\n"
_FIXED_HEADER_BYTES = 16
_VERSION_TWO_PROXY_COMMAND = 0x21
_TCP_OVER_IPV4 = 0x11
_TCP_OVER_IPV6 = 0x21
_IPV4_ADDRESS_BYTES = 12
_IPV6_ADDRESS_BYTES = 36


class ProxyV2HeaderError(ValueError):
    """Raised when bytes are outside the closed PROXY v2 TCP profile."""


class ProxyV2TcpFamily(StrEnum):
    IPV4 = "TCP4"
    IPV6 = "TCP6"


@dataclass(frozen=True, slots=True)
class ProxyV2TcpHeader:
    family: ProxyV2TcpFamily
    source_address: IPv4Address | IPv6Address
    destination_address: IPv4Address | IPv6Address
    source_port: int
    destination_port: int


def parse_proxy_protocol_v2_tcp_header(header: bytes) -> ProxyV2TcpHeader:
    """Parse exactly one complete TCP address header without TLVs."""
    if type(header) is not bytes:
        raise TypeError("header must be exact immutable bytes")
    if len(header) < _FIXED_HEADER_BYTES:
        raise ProxyV2HeaderError("header is shorter than the fixed version 2 header")
    if header[:12] != _PROXY_V2_SIGNATURE:
        raise ProxyV2HeaderError("header has an invalid version 2 signature")
    if header[12] != _VERSION_TWO_PROXY_COMMAND:
        raise ProxyV2HeaderError("header must use version 2 with the PROXY command")

    family_protocol = header[13]
    if family_protocol == _TCP_OVER_IPV4:
        family = ProxyV2TcpFamily.IPV4
        expected_address_bytes = _IPV4_ADDRESS_BYTES
    elif family_protocol == _TCP_OVER_IPV6:
        family = ProxyV2TcpFamily.IPV6
        expected_address_bytes = _IPV6_ADDRESS_BYTES
    else:
        raise ProxyV2HeaderError("header must describe TCP over IPv4 or IPv6")

    declared_address_bytes = int.from_bytes(header[14:16], "big")
    if declared_address_bytes != expected_address_bytes:
        raise ProxyV2HeaderError("declared length must contain only the TCP address block")
    if len(header) != _FIXED_HEADER_BYTES + declared_address_bytes:
        raise ProxyV2HeaderError("header length does not match its exact declared length")

    address_block = header[_FIXED_HEADER_BYTES:]
    if family is ProxyV2TcpFamily.IPV4:
        source_address = IPv4Address(address_block[0:4])
        destination_address = IPv4Address(address_block[4:8])
        ports_offset = 8
    else:
        source_address = IPv6Address(address_block[0:16])
        destination_address = IPv6Address(address_block[16:32])
        ports_offset = 32

    return ProxyV2TcpHeader(
        family=family,
        source_address=source_address,
        destination_address=destination_address,
        source_port=int.from_bytes(address_block[ports_offset : ports_offset + 2], "big"),
        destination_port=int.from_bytes(
            address_block[ports_offset + 2 : ports_offset + 4],
            "big",
        ),
    )


```

## Example

```python
tcp4_bytes = bytes.fromhex(
    "0d0a0d0a000d0a515549540a"  # signature
    "21"  # version 2, PROXY command
    "11"  # TCP over IPv4
    "000c"  # 12 address bytes
    "c0000201"  # 192.0.2.1
    "c6336409"  # 198.51.100.9
    "3039"  # port 12345
    "01bb"  # port 443
)
tcp6_bytes = bytes.fromhex(
    "0d0a0d0a000d0a515549540a"
    "21"
    "21"  # TCP over IPv6
    "0024"  # 36 address bytes
    "00000000000000000000ffffc0000201"
    "20010db8000000000000000000000002"
    "0000"
    "ffff"
)

tcp4 = parse_proxy_protocol_v2_tcp_header(tcp4_bytes)
tcp6 = parse_proxy_protocol_v2_tcp_header(tcp6_bytes)

assert tcp4 == ProxyV2TcpHeader(
    family=ProxyV2TcpFamily.IPV4,
    source_address=IPv4Address("192.0.2.1"),
    destination_address=IPv4Address("198.51.100.9"),
    source_port=12_345,
    destination_port=443,
)
assert tcp6 == ProxyV2TcpHeader(
    family=ProxyV2TcpFamily.IPV6,
    source_address=IPv6Address("::ffff:192.0.2.1"),
    destination_address=IPv6Address("2001:db8::2"),
    source_port=0,
    destination_port=65_535,
)
assert type(tcp6.source_address) is IPv6Address

with_tlv = bytearray(tcp6_bytes + b"\x04\x00\x00")
with_tlv[14:16] = (39).to_bytes(2, "big")
try:
    parse_proxy_protocol_v2_tcp_header(bytes(with_tlv))
except ProxyV2HeaderError:
    tlv_rejected = True
else:
    tlv_rejected = False

assert tlv_rejected
```

## Trade-offs and Limitations

Parsing performs constant work over exactly 28 or 52 bytes and allocates one
immutable result plus two standard-library address objects. Packed addresses
avoid text-spelling ambiguity, ports preserve the complete unsigned 16-bit
range, and an IPv4-mapped IPv6 address remains an `IPv6Address`.

This is a closed subset, not a complete PROXY protocol receiver. It rejects
`LOCAL`, `UNSPEC`, datagram and Unix-socket families, every TLV, incomplete
input, and bytes following the exact header. It neither reads from a socket nor
returns unconsumed application bytes, chooses a deadline, auto-detects version
1 or 2, or verifies that a configured peer sent the header.

The protocol carries asserted connection metadata and provides no
authentication by itself. Accepting these addresses from an arbitrary client
can enable source-address spoofing and corrupt authorization or audit data.
Trust establishment, listener separation, actual-peer retention, and policy
for advertised addresses remain mandatory caller responsibilities.

## Related Snippets

<!-- catalog:related:start -->
- [Parse One Bounded PROXY Protocol Version 1 Line](parse-one-bounded-proxy-protocol-version-one-line.md)
- [Parse a Bounded Host and Port with Bracketed IPv6](parse-a-bounded-host-and-port-with-bracketed-ipv6.md)
- [Collapse IPv4-Mapped IPv6 Literals to One Canonical Address Key](collapse-ipv4-mapped-ipv6-literals-to-one-canonical-address-key.md)
<!-- catalog:related:end -->
