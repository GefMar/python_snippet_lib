---
title: "Collapse a Bounded Same-Family CIDR Set Without Changing Its Address Union"
snippet_type: algorithm
use_cases:
  - networking
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - classify-a-pre-resolved-ip-against-a-bounded-cidr-zone-snapshot.md
  - parse-a-bounded-host-and-port-with-bracketed-ipv6.md
  - ../algorithms-data-structures/coalesce-bounded-half-open-integer-intervals-under-an-explicit-adjacency-policy.md
---

# Collapse a Bounded Same-Family CIDR Set Without Changing Its Address Union

## Idea and Problem

A collection of CIDR networks can contain duplicates, covered entries, and mergeable siblings; collapse a bounded collection into a canonical tuple that preserves exactly the represented address union.

The input contract matters: every item is a canonical, aligned network string, and
all items use the same IP version. The result describes only an address union. It
does not preserve input labels, ordering, policy, or provenance.

## When to Use

Use this at a bounded configuration or serialization boundary when downstream code
needs a minimal deterministic representation of an IPv4-only or IPv6-only address
set. It is suitable when duplicates and mergeable CIDR blocks are semantically
interchangeable with their collapsed form.

Do not use it where entries carry route priority, firewall actions, exclusions, or
other per-network meaning. Those semantics cannot be recovered after collapsing.

## Implementation

```python
from ipaddress import IPv4Network, IPv6Network, collapse_addresses, ip_network

IpNetwork = IPv4Network | IPv6Network

_MAX_NETWORKS = 256
_MAX_CIDR_TEXT_BYTES = 49
_MAX_TOTAL_TEXT_BYTES = 8_192


class CidrSetError(ValueError):
    """Raised when a CIDR collection violates the bounded input contract."""


def _parse_canonical_network(raw: object, position: int) -> IpNetwork:
    if type(raw) is not str:
        raise TypeError(f"cidrs[{position}] must be an exact str")
    if not raw.isascii() or not 1 <= len(raw) <= _MAX_CIDR_TEXT_BYTES:
        raise CidrSetError(f"cidrs[{position}] has an invalid text length")
    if "%" in raw:
        raise CidrSetError(f"cidrs[{position}] must not contain a scope identifier")

    try:
        network = ip_network(raw, strict=True)
    except ValueError as exc:
        raise CidrSetError(f"cidrs[{position}] is not an aligned network") from exc

    if raw != network.with_prefixlen:
        raise CidrSetError(f"cidrs[{position}] is not in canonical prefix form")
    return network


def collapse_cidr_set(cidrs: tuple[str, ...]) -> tuple[str, ...]:
    """Return a canonical CIDR tuple with the same represented address union."""
    if type(cidrs) is not tuple:
        raise TypeError("cidrs must be an exact tuple")
    if len(cidrs) > _MAX_NETWORKS:
        raise CidrSetError(f"cidrs must contain at most {_MAX_NETWORKS} networks")

    parsed: list[IpNetwork] = []
    total_text_bytes = 0
    version: int | None = None

    for position, raw in enumerate(cidrs):
        network = _parse_canonical_network(raw, position)
        total_text_bytes += len(raw)
        if total_text_bytes > _MAX_TOTAL_TEXT_BYTES:
            raise CidrSetError("CIDR text exceeds the total byte limit")
        if version is None:
            version = network.version
        elif network.version != version:
            raise CidrSetError("all CIDR networks must use the same IP version")
        parsed.append(network)

    return tuple(network.with_prefixlen for network in collapse_addresses(parsed))
```

## Example

```python
provided = (
    "198.51.100.0/25",
    "192.0.2.128/25",
    "192.0.2.0/26",
    "192.0.2.0/25",
    "192.0.2.128/25",
)
collapsed = collapse_cidr_set(provided)

invalid = 0
for candidate in (
    ("192.0.2.0/24", "2001:db8::/32"),
    ("192.0.2.1/24",),
    ("2001:0db8::/32",),
):
    try:
        collapse_cidr_set(candidate)
    except CidrSetError:
        invalid += 1

assert (
    collapsed == ("192.0.2.0/24", "198.51.100.0/25")
    and collapse_cidr_set(tuple(reversed(provided))) == collapsed
    and collapse_cidr_set(("2001:db8::/33", "2001:db8:8000::/33")) == ("2001:db8::/32",)
    and invalid == 3
)
```

## Trade-offs and Limitations

The input limits bound parsing and sorting work. Collapsing is dominated by ordering
the networks, so expect roughly `O(n log n)` time and `O(n)` additional memory for
`n` accepted entries.

The function intentionally discards duplicates, containment, input order, and the
identity of merged entries. It accepts one IP family per call and rejects host bits,
alternate spellings, netmask notation, and scoped forms. It performs no DNS lookup
and makes no statement about route selection, longest-prefix matching, firewall
precedence, or any metadata attached to the original ranges.

## Related Snippets

<!-- catalog:related:start -->
- [Classify a Pre-Resolved IP Against a Bounded CIDR-Zone Snapshot](classify-a-pre-resolved-ip-against-a-bounded-cidr-zone-snapshot.md)
- [Parse a Bounded Host and Port with Bracketed IPv6](parse-a-bounded-host-and-port-with-bracketed-ipv6.md)
- [Coalesce Bounded Half-Open Integer Intervals Under an Explicit Adjacency Policy](../algorithms-data-structures/coalesce-bounded-half-open-integer-intervals-under-an-explicit-adjacency-policy.md)
<!-- catalog:related:end -->
