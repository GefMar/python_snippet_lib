---
title: "Parse One Bounded PROXY Protocol Version 1 Line"
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
  - parse-a-bounded-host-and-port-with-bracketed-ipv6.md
  - collapse-ipv4-mapped-ipv6-literals-to-one-canonical-address-key.md
  - parse-one-bounded-printable-ascii-http-1-field-section-with-exact-crlf-framing.md
---

# Parse One Bounded PROXY Protocol Version 1 Line

## Idea and Problem

Parse one complete PROXY protocol version 1 line under a small canonical wire profile and return immutable advertised endpoint data.

The parser accepts only the short `UNKNOWN` form or an exact-space six-field
`TCP4` or `TCP6` form. It validates CRLF framing, address-family syntax, and
canonical decimal ports before creating a tagged result. `UNKNOWN` deliberately
contains no advertised endpoint, so callers cannot accidentally reuse ignored
text from that form.

## When to Use

Use this recipe only after a dedicated PROXY-protocol listener has received
one complete line from an authenticated or explicitly allowlisted transport
peer. Keep the actual socket peer identity available for authorization and for
the `UNKNOWN` case; advertised addresses are assertions made by that peer.

Select this parser by listener configuration, never by inspecting arbitrary
application traffic and guessing whether a header is present. Use a maintained
protocol implementation when version 2, noncanonical version 1 input, socket
reads, timeouts, or incremental framing are required.

## Implementation

```python
from dataclasses import dataclass
from enum import StrEnum
from ipaddress import AddressValueError, IPv4Address, IPv6Address

_MIN_LINE_BYTES = 15
_MAX_LINE_BYTES = 107
_UNKNOWN_LINE = b"PROXY UNKNOWN\r\n"
_IPV6_TEXT_BYTES = frozenset(b"0123456789abcdefABCDEF:")


class ProxyV1LineError(ValueError):
    pass


class ProxyV1Kind(StrEnum):
    UNKNOWN = "UNKNOWN"
    TCP4 = "TCP4"
    TCP6 = "TCP6"


@dataclass(frozen=True, slots=True)
class ProxyV1Header:
    kind: ProxyV1Kind
    source_address: IPv4Address | IPv6Address | None
    destination_address: IPv4Address | IPv6Address | None
    source_port: int | None
    destination_port: int | None


def _parse_ipv4(token: bytes) -> IPv4Address:
    octets = token.split(b".")
    if len(octets) != 4:
        raise ProxyV1LineError("an IPv4 address must contain four octets")

    for octet in octets:
        if not 1 <= len(octet) <= 3 or not octet.isdigit():
            raise ProxyV1LineError("IPv4 octets must contain one to three digits")
        if len(octet) > 1 and octet.startswith(b"0"):
            raise ProxyV1LineError("IPv4 octets must not have leading zeroes")
        if int(octet) > 255:
            raise ProxyV1LineError("an IPv4 octet exceeds 255")

    return IPv4Address(token.decode("ascii"))


def _parse_ipv6(token: bytes) -> IPv6Address:
    if not token or any(octet not in _IPV6_TEXT_BYTES for octet in token):
        raise ProxyV1LineError("IPv6 text must contain only hexadecimal digits and colons")
    try:
        return IPv6Address(token.decode("ascii"))
    except AddressValueError as error:
        raise ProxyV1LineError("the IPv6 address is invalid") from error


def _parse_port(token: bytes) -> int:
    if not 1 <= len(token) <= 5 or not token.isdigit():
        raise ProxyV1LineError("ports must contain one to five decimal digits")
    if len(token) > 1 and token.startswith(b"0"):
        raise ProxyV1LineError("ports must not have leading zeroes")
    port = int(token)
    if port > 65_535:
        raise ProxyV1LineError("a port exceeds 65535")
    return port


def parse_proxy_protocol_v1_line(line: bytes) -> ProxyV1Header:
    if type(line) is not bytes:
        raise TypeError("line must be exact immutable bytes")
    if not _MIN_LINE_BYTES <= len(line) <= _MAX_LINE_BYTES:
        raise ProxyV1LineError("line length is outside the supported range")
    if not line.endswith(b"\r\n"):
        raise ProxyV1LineError("line must end with exactly one CRLF")
    if b"\r" in line[:-2] or b"\n" in line[:-2]:
        raise ProxyV1LineError("line must not contain embedded line breaks")

    if line == _UNKNOWN_LINE:
        return ProxyV1Header(ProxyV1Kind.UNKNOWN, None, None, None, None)

    fields = line[:-2].split(b" ")
    if len(fields) != 6 or any(not field for field in fields):
        raise ProxyV1LineError("TCP forms require six fields separated by one space")
    marker, protocol, source_text, destination_text, source_text_port, destination_text_port = (
        fields
    )
    if marker != b"PROXY":
        raise ProxyV1LineError("line must start with the exact PROXY marker")

    if protocol == b"TCP4":
        kind = ProxyV1Kind.TCP4
        source_address = _parse_ipv4(source_text)
        destination_address = _parse_ipv4(destination_text)
    elif protocol == b"TCP6":
        kind = ProxyV1Kind.TCP6
        source_address = _parse_ipv6(source_text)
        destination_address = _parse_ipv6(destination_text)
    else:
        raise ProxyV1LineError("the protocol field must be TCP4 or TCP6")

    return ProxyV1Header(
        kind,
        source_address,
        destination_address,
        _parse_port(source_text_port),
        _parse_port(destination_text_port),
    )


```

## Example

```python
ipv4 = parse_proxy_protocol_v1_line(b"PROXY TCP4 192.0.2.1 198.51.100.9 12345 443\r\n")
ipv6 = parse_proxy_protocol_v1_line(b"PROXY TCP6 2001:db8::1 2001:db8::2 0 65535\r\n")
unknown = parse_proxy_protocol_v1_line(b"PROXY UNKNOWN\r\n")


def is_rejected(candidate: object) -> bool:
    try:
        parse_proxy_protocol_v1_line(candidate)  # type: ignore[arg-type]
    except TypeError:
        return True
    except ProxyV1LineError:
        return True
    return False


invalid = (
    bytearray(b"PROXY UNKNOWN\r\n"),
    b"PROXY UNKNOWN\n",
    b"PROXY UNKNOWN extra\r\n",
    b"PROXY  TCP4 192.0.2.1 198.51.100.9 1 2\r\n",
    b"PROXY TCP4 192.00.2.1 198.51.100.9 1 2\r\n",
    b"PROXY TCP4 2001:db8::1 2001:db8::2 1 2\r\n",
    b"PROXY TCP6 ::ffff:192.0.2.1 2001:db8::2 1 2\r\n",
    b"PROXY TCP6 fe80::1%3 2001:db8::2 1 2\r\n",
    b"PROXY TCP4 192.0.2.1 198.51.100.9 01 2\r\n",
    b"PROXY TCP4 192.0.2.1 198.51.100.9 65536 2\r\n",
    b"PROXY TCP4 192.0.2.1\r 198.51.100.9 1 2\r\n",
)

assert ipv4 == ProxyV1Header(
    ProxyV1Kind.TCP4,
    IPv4Address("192.0.2.1"),
    IPv4Address("198.51.100.9"),
    12_345,
    443,
)
assert ipv6 == ProxyV1Header(
    ProxyV1Kind.TCP6,
    IPv6Address("2001:db8::1"),
    IPv6Address("2001:db8::2"),
    0,
    65_535,
)
assert unknown == ProxyV1Header(ProxyV1Kind.UNKNOWN, None, None, None, None)
assert all(is_rejected(candidate) for candidate in invalid)
```

## Trade-offs and Limitations

Parsing is linear in at most 107 already-materialized bytes and allocates a
small fixed number of tokens plus two immutable address objects. The lower
length bound admits the 15-byte short `UNKNOWN` line; the upper bound is
inclusive. IPv4 and port spellings are canonical decimal without leading
zeroes. IPv6 input permits only hexadecimal digits and colons, then relies on
`IPv6Address` for a complete 128-bit family parse; scope identifiers and dotted
IPv4 tails are excluded.

This is a closed version 1 profile, not a socket reader or a complete PROXY
protocol implementation. It rejects optional text after `UNKNOWN`, version 2,
extra whitespace, incomplete framing, and bytes after the final CRLF. It does
not authenticate the sender, preserve the actual peer endpoint, impose a read
deadline, or decide whether a connection should use the protocol. Those
controls belong at the dedicated listener before advertised endpoint data is
trusted.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Bounded Host and Port with Bracketed IPv6](parse-a-bounded-host-and-port-with-bracketed-ipv6.md)
- [Collapse IPv4-Mapped IPv6 Literals to One Canonical Address Key](collapse-ipv4-mapped-ipv6-literals-to-one-canonical-address-key.md)
- [Parse One Bounded Printable-ASCII HTTP/1 Field Section with Exact CRLF Framing](parse-one-bounded-printable-ascii-http-1-field-section-with-exact-crlf-framing.md)
<!-- catalog:related:end -->
