---
title: "Build a Canonical HTTP Origin Key"
snippet_type: recipe
use_cases:
  - networking
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - parse-a-bounded-host-and-port-with-bracketed-ipv6.md
  - ../algorithms-data-structures/build-a-canonical-unicode-caseless-comparison-key.md
  - ../security-privacy/validate-a-conservative-unicode-filename-component.md
---

# Build a Canonical HTTP Origin Key

## Idea and Problem

Convert one strict HTTP or HTTPS origin into a stable key with a normalized host and an explicit port.

Equivalent spellings such as an uppercase DNS name, one trailing DNS root dot,
an omitted default port, or a compressed IPv6 literal should not create
different identifiers. The parser rejects every component that an origin key
would otherwise discard, so credentials, paths, queries, and fragments cannot
silently collapse into the same result.

## When to Use

Use this helper when a bounded configuration value identifies an HTTP origin
and equality must follow one deliberately narrow policy. Inputs may contain
DNS names using Python's built-in IDNA codec, ordinary IPv4 literals, or
bracketed IPv6 literals.
The canonical key always contains `scheme://host:port`, including the default
port, and IPv6 remains bracketed.

Do not use this key as an authorization decision, an SSRF defense, a DNS
pinning mechanism, or proof that two URLs reach the same service. Use an
IDNA 2008 or UTS 46 library when that precise compatibility policy is required.

## Implementation

```python
import re
from ipaddress import IPv4Address, IPv6Address, ip_address
from urllib.parse import urlsplit


_MAX_ORIGIN_LENGTH = 2048
_MAX_DNS_LENGTH = 253
_DNS_LABEL = re.compile(
    r"[a-z0-9](?:[a-z0-9-]{0,61}[a-z0-9])?",
    re.ASCII,
)
_LEGACY_NUMERIC_LABEL = re.compile(r"(?:[0-9]+|0x[0-9a-f]+)", re.ASCII)


def _normalize_origin_host(host: str) -> str:
    if "%" in host:
        raise ValueError("IPv6 zone identifiers are not supported")

    try:
        address = ip_address(host)
    except ValueError:
        if host.endswith(".."):
            raise ValueError("a DNS name may have at most one trailing dot")
        dns_name = host[:-1] if host.endswith(".") else host
        if not dns_name:
            raise ValueError("host must not be empty")
        try:
            normalized = dns_name.encode("idna").decode("ascii").lower()
        except UnicodeError as error:
            raise ValueError("host is not valid under the selected IDNA policy") from error
        if not 1 <= len(normalized) <= _MAX_DNS_LENGTH:
            raise ValueError("DNS name length is outside the supported range")
        labels = normalized.split(".")
        if any(_DNS_LABEL.fullmatch(label) is None for label in labels):
            raise ValueError("host contains an invalid DNS label")
        if all(_LEGACY_NUMERIC_LABEL.fullmatch(label) for label in labels):
            raise ValueError("ambiguous numeric host is not supported")
        return normalized

    if isinstance(address, IPv4Address):
        return str(address)
    if isinstance(address, IPv6Address):
        return f"[{address.compressed}]"
    raise AssertionError("unexpected IP address family")


def canonical_http_origin_key(origin: str) -> str:
    if not isinstance(origin, str):
        raise TypeError("origin must be text")
    if not 1 <= len(origin) <= _MAX_ORIGIN_LENGTH:
        raise ValueError("origin length is outside the supported range")
    if any(ord(character) <= 0x20 or ord(character) == 0x7F for character in origin):
        raise ValueError("origin must not contain whitespace or control characters")
    if "?" in origin or "#" in origin:
        raise ValueError("origin must not contain a query or fragment")

    try:
        parsed = urlsplit(origin)
        port = parsed.port
        host = parsed.hostname
    except ValueError as error:
        raise ValueError("origin has an invalid authority") from error

    scheme = parsed.scheme.lower()
    if scheme not in {"http", "https"}:
        raise ValueError("origin must use http or https")
    if not parsed.netloc or host is None:
        raise ValueError("origin must be absolute and contain a host")
    if parsed.username is not None or parsed.password is not None:
        raise ValueError("origin must not contain credentials")
    if parsed.path:
        raise ValueError("origin must not contain a path")
    if parsed.netloc.endswith(":"):
        raise ValueError("an explicit port must not be empty")

    normalized_host = _normalize_origin_host(host)
    if normalized_host.startswith("[") and not parsed.netloc.startswith("["):
        raise ValueError("an IPv6 origin must use brackets")

    default_port = 80 if scheme == "http" else 443
    normalized_port = default_port if port is None else port
    if not 1 <= normalized_port <= 65_535:
        raise ValueError("port is outside the supported range")
    return f"{scheme}://{normalized_host}:{normalized_port}"
```

## Example

```python
keys = (
    canonical_http_origin_key("HTTPS://BÜCHER.Example.:443"),
    canonical_http_origin_key("https://xn--bcher-kva.example"),
    canonical_http_origin_key("http://[2001:0db8::1]"),
    canonical_http_origin_key("http://example.com:8080"),
)

invalid_origins = (
    "https://" + "reader@" + "example.com",
    "https://example.com/",
    "https://example.com?view=full",
    "https://example.com:",
    "https://example.com:0",
    "https://2001:db8::1",
    "https://[2001:db8::1%25en0]",
    "https://127.1",
)
rejected = []
for invalid_origin in invalid_origins:
    try:
        canonical_http_origin_key(invalid_origin)
    except ValueError:
        rejected.append(invalid_origin)

assert (keys, tuple(rejected)) == (
    (
        "https://xn--bcher-kva.example:443",
        "https://xn--bcher-kva.example:443",
        "http://[2001:db8::1]:80",
        "http://example.com:8080",
    ),
    invalid_origins,
)
```

## Trade-offs and Limitations

The accepted grammar is intentionally narrower than general URI syntax. It
rejects even a root `/` path, empty query or fragment delimiters, user
information, IPvFuture, IPv6 zone identifiers, legacy numeric IPv4 spellings,
and non-HTTP schemes. It removes exactly one trailing DNS root dot and uses the
IDNA behavior supplied by the running Python version; that is not a promise of
IDNA 2008 or browser-equivalent processing.

Parsing and normalization perform no DNS lookup. DNS rebinding, split-horizon
answers, proxy routing, TLS certificate validation, service ownership, and
network reachability remain separate concerns. Explicit default ports collapse
to the same key as omitted ports, while different schemes or non-default ports
remain distinct.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Bounded Host and Port with Bracketed IPv6](parse-a-bounded-host-and-port-with-bracketed-ipv6.md)
- [Build a Canonical Unicode Caseless Comparison Key](../algorithms-data-structures/build-a-canonical-unicode-caseless-comparison-key.md)
- [Validate a Conservative Unicode Filename Component](../security-privacy/validate-a-conservative-unicode-filename-component.md)
<!-- catalog:related:end -->
