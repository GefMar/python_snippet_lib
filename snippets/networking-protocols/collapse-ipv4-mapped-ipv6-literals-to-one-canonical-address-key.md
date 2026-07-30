---
title: "Collapse IPv4-Mapped IPv6 Literals to One Canonical Address Key"
snippet_type: recipe
use_cases:
  - data-transformation
  - networking
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - parse-a-bounded-host-and-port-with-bracketed-ipv6.md
  - classify-a-pre-resolved-ip-against-a-bounded-cidr-zone-snapshot.md
  - subtract-a-bounded-disjoint-cidr-exclusion-set-from-one-parent-network.md
---

# Collapse IPv4-Mapped IPv6 Literals to One Canonical Address Key

## Idea and Problem

Normalize an exact IP literal into one text key while deliberately treating IPv4-mapped IPv6 addresses as equivalent to their embedded IPv4 address.

Dual-stack APIs can report the same IPv4 endpoint as `192.0.2.128`,
`::ffff:192.0.2.128`, or `::ffff:c000:280`. `IPv6Address.ipv4_mapped`
identifies precisely the mapped `::ffff:0:0/96` form. Collapsing that subset
avoids three keys for the same address without merging ordinary IPv6 values.

## When to Use

Use this normalization only when a cache, counter, or deduplication boundary
has an explicit policy that IPv4 and IPv4-mapped IPv6 observations represent
the same address identity. Pass one already-separated ASCII address literal,
not an authority, URL, hostname, or user-facing label.

Keep the original observation when family and spelling are audit-relevant. DNS
resolution, endpoint authorization, network classification, and trust decisions
must happen in separate components with their own policies.

## Implementation

```python
from ipaddress import IPv6Address, ip_address

_MAX_IP_LITERAL_LENGTH = 45


def canonical_address_key(literal: str) -> str:
    if type(literal) is not str:
        raise TypeError("literal must be an exact string")
    if not 1 <= len(literal) <= _MAX_IP_LITERAL_LENGTH:
        raise ValueError("literal length is outside the supported range")
    try:
        literal.encode("ascii")
    except UnicodeEncodeError:
        raise ValueError("literal must contain only ASCII") from None
    if literal != literal.strip():
        raise ValueError("literal must not contain surrounding whitespace")
    if any(marker in literal for marker in "[]%"):
        raise ValueError("brackets and scoped addresses are not supported")

    try:
        address = ip_address(literal)
    except ValueError:
        raise ValueError("value must be one exact IP address literal") from None

    if isinstance(address, IPv6Address):
        mapped = address.ipv4_mapped
        if mapped is not None:
            return mapped.compressed
    return address.compressed


```

## Example

```python
equivalent = {
    canonical_address_key("192.0.2.128"),
    canonical_address_key("::ffff:192.0.2.128"),
    canonical_address_key("::ffff:c000:0280"),
}
assert equivalent == {"192.0.2.128"}
assert canonical_address_key(next(iter(equivalent))) == "192.0.2.128"

assert canonical_address_key("2001:0db8:0:0:0:0:0:1") == "2001:db8::1"
assert canonical_address_key("::192.0.2.128") == "::c000:280"
assert (
    canonical_address_key("ffff:ffff:ffff:ffff:ffff:ffff:255.255.255.255")
    == "ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff"
)

invalid_values = (
    "",
    "[2001:db8::1]",
    "192.0.2.128:443",
    "fe80::1%eth0",
    "example.test",
    " 192.0.2.128",
    "192.0.2.128 ",
    "2001\uff1adb8::1",
    "1" * 46,
)
for invalid in invalid_values:
    try:
        canonical_address_key(invalid)
    except ValueError:
        pass
    else:
        raise AssertionError(f"invalid literal was accepted: {invalid!r}")

assert canonical_address_key("2001:db8::2") == "2001:db8::2"
```

## Trade-offs and Limitations

The cross-family collision for IPv4 and IPv4-mapped IPv6 is intentional and is
the only added equivalence. IPv4-compatible IPv6 forms such as `::192.0.2.128`
remain IPv6 keys. NAT64, 6to4, and Teredo addresses are not collapsed either.
Canonical compression also discards the input spelling, so it must not replace
raw evidence when representation matters.

This parser rejects brackets, ports, scope-zone suffixes, hostnames, non-ASCII
text, and surrounding whitespace and performs no DNS lookup. A valid bare IPv6
literal can end in a decimal-looking hextet; no literal-only parser can infer
that a caller intended that hextet as a port. The returned key provides neither
ACL membership nor proof of peer identity, route, reachability, ownership, or
transport security.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Bounded Host and Port with Bracketed IPv6](parse-a-bounded-host-and-port-with-bracketed-ipv6.md)
- [Classify a Pre-Resolved IP Against a Bounded CIDR-Zone Snapshot](classify-a-pre-resolved-ip-against-a-bounded-cidr-zone-snapshot.md)
- [Subtract a Bounded Disjoint CIDR Exclusion Set from One Parent Network](subtract-a-bounded-disjoint-cidr-exclusion-set-from-one-parent-network.md)
<!-- catalog:related:end -->
