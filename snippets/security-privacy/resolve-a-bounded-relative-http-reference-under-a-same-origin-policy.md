---
title: "Resolve a Bounded Relative HTTP Reference under a Same-Origin Policy"
snippet_type: recipe
use_cases:
  - networking
  - security
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../networking-protocols/build-a-canonical-http-origin-key.md
  - ../networking-protocols/build-a-bounded-http-request-target-from-typed-segments.md
  - ../networking-protocols/parse-a-bounded-host-and-port-with-bracketed-ipv6.md
---

# Resolve a Bounded Relative HTTP Reference under a Same-Origin Policy

## Idea and Problem

Resolve one bounded relative HTTP reference only after excluding authority-changing forms, then verify the resulting origin again.

`urllib.parse.urljoin` deliberately accepts absolute and scheme-relative
references. That general behavior is unsafe when an untrusted value is
supposed to select only a path or query below a known HTTP origin. This helper
places a narrow grammar before the join and a canonical origin comparison
after it, so both assumptions are executable.

## When to Use

Use this recipe when a protocol accepts an ASCII relative reference beside one
already validated HTTP or HTTPS base URL. It permits empty, query-only,
root-relative, and ordinary relative-path references, including dot-segment
resolution, while keeping the base scheme, host, and effective port.

Apply a separate resource-authorization rule to the resulting path and query.
Use a maintained URL/security layer when browser-equivalent URL parsing,
Unicode hostnames, redirect handling, DNS policy, or a complete SSRF defense is
required.

## Implementation

```python
import re
from ipaddress import IPv6Address, ip_address
from urllib.parse import urljoin, urlsplit

_MAX_INPUT_CHARACTERS = 2_048
_MAX_OUTPUT_CHARACTERS = 3_072
_MAX_DNS_CHARACTERS = 253
_HEX_DIGITS = frozenset("0123456789abcdefABCDEF")
_DNS_LABEL = re.compile(
    r"[a-z0-9](?:[a-z0-9-]{0,61}[a-z0-9])?",
    re.ASCII,
)


class URLResolutionError(ValueError):
    """The URL text or same-origin policy is invalid."""


def _validate_url_text(
    value: object,
    *,
    name: str,
    minimum: int,
    maximum: int,
) -> str:
    if type(value) is not str:
        raise TypeError(f"{name} must be an exact string")
    if not minimum <= len(value) <= maximum:
        raise URLResolutionError(f"{name} length is outside the supported range")
    if value and any(not 0x21 <= ord(character) <= 0x7E for character in value):
        raise URLResolutionError(f"{name} must contain visible ASCII")
    if "\\" in value:
        raise URLResolutionError(f"{name} must not contain backslashes")
    for index, character in enumerate(value):
        if character != "%":
            continue
        if (
            index + 2 >= len(value)
            or value[index + 1] not in _HEX_DIGITS
            or value[index + 2] not in _HEX_DIGITS
        ):
            raise URLResolutionError(f"{name} contains a malformed percent escape")
    return value


def _canonical_origin(url: str) -> tuple[str, str, int]:
    if "#" in url:
        raise URLResolutionError("URL fragments are not supported")
    try:
        parsed = urlsplit(url, allow_fragments=False)
        host = parsed.hostname
        port = parsed.port
    except ValueError as error:
        raise URLResolutionError("URL has an invalid authority") from error

    if parsed.scheme not in {"http", "https"}:
        raise URLResolutionError("URL must use lowercase http or https")
    if not url.startswith(f"{parsed.scheme}://"):
        raise URLResolutionError("URL scheme spelling must be canonical")
    if not parsed.netloc or host is None:
        raise URLResolutionError("URL must be absolute and contain a host")
    if "@" in parsed.netloc:
        raise URLResolutionError("URL credentials are not supported")
    if parsed.netloc.endswith(":"):
        raise URLResolutionError("an explicit port must not be empty")
    if parsed.path and not parsed.path.startswith("/"):
        raise URLResolutionError("an absolute HTTP URL path must begin with a slash")
    if "%" in host:
        raise URLResolutionError("host zone identifiers are not supported")

    try:
        address = ip_address(host)
    except ValueError:
        address = None
    if address is None:
        if not 1 <= len(host) <= _MAX_DNS_CHARACTERS:
            raise URLResolutionError("DNS host length is outside the supported range")
        labels = host.split(".")
        if any(_DNS_LABEL.fullmatch(label) is None for label in labels):
            raise URLResolutionError("DNS host is not in canonical lowercase form")
        if all(label.isdecimal() for label in labels):
            raise URLResolutionError("ambiguous all-decimal DNS hosts are not supported")
        canonical_host = host
    else:
        canonical_host = str(address)
        if isinstance(address, IPv6Address):
            canonical_host = f"[{canonical_host}]"

    if port is not None and not 1 <= port <= 65_535:
        raise URLResolutionError("port is outside the supported range")
    port_suffix = "" if port is None else f":{port}"
    if parsed.netloc != f"{canonical_host}{port_suffix}":
        raise URLResolutionError("URL authority spelling is not canonical")

    effective_port = port
    if effective_port is None:
        effective_port = 80 if parsed.scheme == "http" else 443
    return parsed.scheme, canonical_host, effective_port


def resolve_same_origin_http_reference(
    base_url: str,
    reference: str,
) -> str:
    base_url = _validate_url_text(
        base_url,
        name="base URL",
        minimum=1,
        maximum=_MAX_INPUT_CHARACTERS,
    )
    reference = _validate_url_text(
        reference,
        name="reference",
        minimum=0,
        maximum=_MAX_INPUT_CHARACTERS,
    )
    base_origin = _canonical_origin(base_url)

    if "#" in reference:
        raise URLResolutionError("reference fragments are not supported")
    if reference.startswith("//"):
        raise URLResolutionError("scheme-relative references are not supported")
    try:
        parsed_reference = urlsplit(reference, allow_fragments=False)
    except ValueError as error:
        raise URLResolutionError("reference cannot be parsed") from error
    if parsed_reference.scheme:
        raise URLResolutionError("reference must not contain a scheme")
    if parsed_reference.netloc:
        raise URLResolutionError("reference must not contain an authority")

    try:
        resolved = urljoin(base_url, reference, allow_fragments=False)
    except ValueError as error:
        raise URLResolutionError("reference cannot be resolved") from error
    resolved = _validate_url_text(
        resolved,
        name="resolved URL",
        minimum=1,
        maximum=_MAX_OUTPUT_CHARACTERS,
    )
    if _canonical_origin(resolved) != base_origin:
        raise URLResolutionError("resolved URL changes the canonical origin")
    return resolved
```

## Example

```python
base = "https://example.test/a/index"
resolved = (
    resolve_same_origin_http_reference(base, "../asset?q=1"),
    resolve_same_origin_http_reference(base, "?page=2"),
    resolve_same_origin_http_reference(base, "/health"),
    resolve_same_origin_http_reference(base, ""),
    resolve_same_origin_http_reference(
        "http://[2001:db8::1]:8080/a/",
        "./status",
    ),
)

invalid_references = (
    "//other.test/x",
    "//",
    "//?query",
    "///path",
    "https://example.test/x",
    "#fragment",
    "../\\other.test/x",
    "%2",
)
rejected = []
for invalid_reference in invalid_references:
    try:
        resolve_same_origin_http_reference(base, invalid_reference)
    except URLResolutionError:
        rejected.append(invalid_reference)

long_prefix = "https://a.test/"
long_base = (
    long_prefix
    + "a" * (_MAX_INPUT_CHARACTERS - len(long_prefix) - 1)
    + "/"
)
try:
    resolve_same_origin_http_reference(
        long_base,
        "b" * _MAX_INPUT_CHARACTERS,
    )
except URLResolutionError:
    output_limit_rejected = True
else:
    output_limit_rejected = False

assert resolved == (
    "https://example.test/asset?q=1",
    "https://example.test/a/index?page=2",
    "https://example.test/health",
    base,
    "http://[2001:db8::1]:8080/a/status",
)
assert tuple(rejected) == invalid_references
assert output_limit_rejected
```

## Trade-offs and Limitations

For base, reference, and resolved-output lengths `B`, `R`, and `O`, the helper
uses `O(B + R + O)` time and state. The output cap is lower than the sum of the
two input caps, so it remains an independently reachable rejection boundary.

This is a syntactic resolution policy, not a complete SSRF defense. It performs
no DNS lookup and cannot control DNS rebinding, proxies, TLS validation,
authentication, transport behavior, or a client's redirects. Reapply
destination policy after every redirect rather than assuming that the first
resolved URL constrains later requests.

The grammar intentionally excludes credentials, fragments, Unicode/IDNA
hosts, trailing DNS dots, zone identifiers, noncanonical IP literals, and
noncanonical port spellings. Percent escapes are checked but never decoded.
Downstream normalization, double decoding, filesystem mapping, and
application-specific path authorization remain outside this helper and can
change the meaning of otherwise same-origin text.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Canonical HTTP Origin Key](../networking-protocols/build-a-canonical-http-origin-key.md)
- [Build a Bounded HTTP Request Target from Typed Segments](../networking-protocols/build-a-bounded-http-request-target-from-typed-segments.md)
- [Parse a Bounded Host and Port with Bracketed IPv6](../networking-protocols/parse-a-bounded-host-and-port-with-bracketed-ipv6.md)
<!-- catalog:related:end -->
