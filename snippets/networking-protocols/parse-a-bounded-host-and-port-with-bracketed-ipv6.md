---
title: "Parse a Bounded Host and Port with Bracketed IPv6"
snippet_type: recipe
use_cases:
  - networking
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - parse-a-bounded-ascii-media-type-value.md
  - read-and-write-size-capped-varint-frames.md
  - ../data-processing/collect-expected-parse-failures-without-stopping-a-batch.md
---

# Parse a Bounded Host and Port with Bracketed IPv6

## Idea and Problem

Parse one conservative host-and-port value without confusing the colons inside an IPv6 literal with a port separator.

Hostnames and IPv4 addresses may carry an optional `:port`; IPv6 uses brackets
when a port is explicit and may be unbracketed only when the caller supplies the
port. The result normalizes hostnames and IP literals into an immutable value
that serializes back to one unambiguous bracketed form.

## When to Use

Use this parser for bounded configuration that deliberately accepts only a
host and TCP/UDP-style numeric port, not a URL. Keep destination authorization,
DNS, transport, and connection policy outside the parser. Use a URI library and
a scheme-specific validator when credentials, paths, query strings, Unicode
hostnames, service names, or other authority components are part of the input.

## Implementation

```python
import re
from dataclasses import dataclass
from ipaddress import IPv6Address, ip_address


_MAX_ENDPOINT_LENGTH = 300
_MAX_HOSTNAME_LENGTH = 253
_HOST_LABEL = re.compile(
    r"[a-z0-9](?:[a-z0-9-]{0,61}[a-z0-9])?",
    re.ASCII,
)
_PORT_TEXT = re.compile(r"[1-9][0-9]{0,4}", re.ASCII)


def _validate_port(port: int, *, name: str) -> int:
    if isinstance(port, bool) or not isinstance(port, int):
        raise TypeError(f"{name} must be an integer")
    if not 1 <= port <= 65_535:
        raise ValueError(f"{name} is outside the supported range")
    return port


def _parse_port_text(text: str) -> int:
    if _PORT_TEXT.fullmatch(text) is None:
        raise ValueError("port must contain canonical decimal digits")
    return _validate_port(int(text), name="port")


def _normalize_host(host: str) -> str:
    if not isinstance(host, str):
        raise TypeError("host must be text")
    if not host or len(host) > _MAX_HOSTNAME_LENGTH:
        raise ValueError("host length is outside the supported range")
    if "%" in host:
        raise ValueError("scoped IP literals are not supported")

    try:
        address = ip_address(host)
    except ValueError:
        if ":" in host:
            raise ValueError("host contains an invalid IP literal") from None
        if all(character in "0123456789." for character in host):
            raise ValueError("host contains an invalid IPv4 address") from None

        normalized = host.lower()
        labels = normalized.split(".")
        if any(_HOST_LABEL.fullmatch(label) is None for label in labels):
            raise ValueError("host contains an invalid hostname")
        return normalized
    return address.compressed


@dataclass(frozen=True, slots=True)
class HostPort:
    host: str
    port: int

    def __post_init__(self) -> None:
        object.__setattr__(self, "host", _normalize_host(self.host))
        object.__setattr__(
            self,
            "port",
            _validate_port(self.port, name="port"),
        )

    def serialize(self) -> str:
        rendered_host = f"[{self.host}]" if ":" in self.host else self.host
        return f"{rendered_host}:{self.port}"


def parse_host_port(text: str, *, default_port: int) -> HostPort:
    if not isinstance(text, str):
        raise TypeError("host and port must be text")
    if not 1 <= len(text) <= _MAX_ENDPOINT_LENGTH:
        raise ValueError("host and port length is outside the supported range")
    if any(not 0x21 <= ord(character) <= 0x7E for character in text):
        raise ValueError("host and port must contain visible ASCII")
    port = _validate_port(default_port, name="default_port")
    if "://" in text or any(marker in text for marker in "/?#@"):
        raise ValueError("URLs and credentials are not supported")

    if text.startswith("["):
        closing_bracket = text.find("]")
        if closing_bracket <= 1:
            raise ValueError("bracketed IPv6 host is malformed")
        literal = text[1:closing_bracket]
        suffix = text[closing_bracket + 1 :]
        try:
            normalized_ipv6 = IPv6Address(literal).compressed
        except ValueError:
            raise ValueError("brackets must contain an IPv6 address") from None

        if suffix:
            if not suffix.startswith(":") or len(suffix) == 1:
                raise ValueError("unexpected text after bracketed IPv6 host")
            port = _parse_port_text(suffix[1:])
        return HostPort(normalized_ipv6, port)

    if "[" in text or "]" in text:
        raise ValueError("host contains unmatched brackets")

    colon_count = text.count(":")
    if colon_count > 1:
        try:
            normalized_ipv6 = IPv6Address(text).compressed
        except ValueError:
            raise ValueError("unbracketed IPv6 host is malformed") from None
        return HostPort(normalized_ipv6, port)

    if colon_count == 1:
        host, raw_port = text.rsplit(":", 1)
        if not host or not raw_port:
            raise ValueError("host and explicit port must not be empty")
        port = _parse_port_text(raw_port)
    else:
        host = text
    return HostPort(host, port)
```

## Example

```python
parsed = (
    parse_host_port("Example.TEST", default_port=443),
    parse_host_port("192.0.2.10:8080", default_port=443),
    parse_host_port("[2001:0db8::1]:8443", default_port=443),
    parse_host_port("2001:db8::2", default_port=443),
)
serialized = tuple(endpoint.serialize() for endpoint in parsed)
round_trip = tuple(
    parse_host_port(value, default_port=1)
    for value in serialized
)

invalid_values = (
    "",
    "example.test:0",
    "example.test:080",
    "example.test:65536",
    "https://example.test",
    "user@example.test:443",
    "example.test/path",
    "[2001:db8::1",
    "[192.0.2.1]:443",
    "999.0.2.1",
)
rejected = []
for value in invalid_values:
    try:
        parse_host_port(value, default_port=443)
    except ValueError:
        rejected.append(value)

assert (parsed, serialized, round_trip == parsed, tuple(rejected)) == (
    (
        HostPort("example.test", 443),
        HostPort("192.0.2.10", 8080),
        HostPort("2001:db8::1", 8443),
        HostPort("2001:db8::2", 443),
    ),
    (
        "example.test:443",
        "192.0.2.10:8080",
        "[2001:db8::1]:8443",
        "[2001:db8::2]:443",
    ),
    True,
    invalid_values,
)
```

## Trade-offs and Limitations

This is not a general URI parser. It excludes IDNA and Unicode hostnames,
trailing-root dots, service-name ports, IPvFuture, CIDR prefixes, and scoped
IPv6 zone identifiers. An unbracketed multi-colon value is always an IPv6
address using the default port; callers must require brackets when a port is
intended. Syntax validation performs no DNS lookup and establishes no
authorization, SSRF protection, routing, TLS/SNI correctness, reachability, or
ownership. Transport selection and secure-to-insecure fallback are separate
decisions and must never be inferred from this value.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Bounded ASCII Media Type Value](parse-a-bounded-ascii-media-type-value.md)
- [Read and Write Size-Capped Varint Frames](read-and-write-size-capped-varint-frames.md)
- [Collect Expected Parse Failures Without Stopping a Batch](../data-processing/collect-expected-parse-failures-without-stopping-a-batch.md)
<!-- catalog:related:end -->
