---
title: "Subtract a Bounded Disjoint CIDR Exclusion Set from One Parent Network"
snippet_type: algorithm
use_cases:
  - networking
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - collapse-a-bounded-same-family-cidr-set-without-changing-its-address-union.md
  - classify-a-pre-resolved-ip-against-a-bounded-cidr-zone-snapshot.md
  - ../algorithms-data-structures/subtract-one-canonical-half-open-integer-interval-set-from-another.md
---

# Subtract a Bounded Disjoint CIDR Exclusion Set from One Parent Network

## Idea and Problem

Subtract a small disjoint CIDR exclusion set from one parent network and return a minimal canonical partition whose limits and order do not depend on caller input order.

This operation is set subtraction over addresses. It preserves no labels or
input ordering and assigns no routing or firewall meaning to either the parent
or exclusions.

## When to Use

Use this at a bounded networking configuration boundary when one aligned IPv4
or IPv6 allocation has a small set of reserved, delegated, or unavailable
subnets and downstream code needs the exact remaining address union as CIDRs.
It is suitable when exclusions are already disjoint and every address has the
same meaning.

Do not use it to combine rules with priorities, actions, longest-prefix
semantics, or per-network metadata. Normalize overlapping policy inputs before
this boundary only if merging them is semantically valid. Use an interval
structure rather than materialized CIDRs when the exclusion count or remainder
partition can be large.

## Implementation

```python
from ipaddress import IPv4Network, IPv6Network, collapse_addresses, ip_network
from itertools import pairwise

IpNetwork = IPv4Network | IPv6Network

_MAX_EXCLUSIONS = 8
_MAX_CIDR_BYTES = 49
_MAX_CIDR_TEXT_BYTES = 512
_MAX_REMAINDERS = 512


class CidrDifferenceError(ValueError):
    """Raised when CIDRs violate the bounded set-difference contract."""


def _parse_canonical_cidr(raw: object, label: str) -> IpNetwork:
    if type(raw) is not str:
        raise TypeError(f"{label} must be an exact str")
    if not raw.isascii() or not 1 <= len(raw) <= _MAX_CIDR_BYTES:
        raise CidrDifferenceError(f"{label} has an invalid ASCII length")
    if "%" in raw:
        raise CidrDifferenceError(f"{label} must not contain a scope identifier")

    try:
        network = ip_network(raw, strict=True)
    except ValueError as exc:
        raise CidrDifferenceError(f"{label} is not an aligned CIDR") from exc
    if raw != network.with_prefixlen:
        raise CidrDifferenceError(f"{label} is not in canonical prefix form")
    return network


def subtract_cidr_exclusions(
    parent: str,
    exclusions: tuple[str, ...],
) -> tuple[str, ...]:
    """Return a minimal canonical partition of parent minus exclusions."""
    parsed_parent = _parse_canonical_cidr(parent, "parent")
    if type(exclusions) is not tuple:
        raise TypeError("exclusions must be an exact tuple")
    if len(exclusions) > _MAX_EXCLUSIONS:
        raise CidrDifferenceError("too many exclusions")

    parsed_exclusions: list[IpNetwork] = []
    seen: set[str] = set()
    total_text_bytes = len(parent)
    for index, raw in enumerate(exclusions):
        network = _parse_canonical_cidr(raw, f"exclusions[{index}]")
        if raw in seen:
            raise CidrDifferenceError("exclusions must be unique")
        seen.add(raw)
        total_text_bytes += len(raw)
        if total_text_bytes > _MAX_CIDR_TEXT_BYTES:
            raise CidrDifferenceError("CIDR text exceeds the aggregate limit")
        if network.version != parsed_parent.version:
            raise CidrDifferenceError("all CIDRs must use the same IP version")
        if not network.subnet_of(parsed_parent):
            raise CidrDifferenceError("every exclusion must be inside the parent")
        parsed_exclusions.append(network)

    parsed_exclusions.sort(
        key=lambda network: (int(network.network_address), network.prefixlen)
    )
    for previous, current in pairwise(parsed_exclusions):
        if previous.overlaps(current):
            raise CidrDifferenceError("exclusions must be pairwise disjoint")

    remaining: list[IpNetwork] = [parsed_parent]
    for exclusion in parsed_exclusions:
        next_remaining: list[IpNetwork] = []
        matched = False
        for network in remaining:
            if exclusion.subnet_of(network):
                next_remaining.extend(network.address_exclude(exclusion))
                matched = True
            else:
                next_remaining.append(network)
        if not matched:
            raise AssertionError("validated exclusion lost its containing remainder")
        if len(next_remaining) > _MAX_REMAINDERS:
            raise CidrDifferenceError("CIDR remainder limit exceeded")
        remaining = next_remaining

    minimal = tuple(collapse_addresses(remaining))
    if len(minimal) > _MAX_REMAINDERS:
        raise CidrDifferenceError("CIDR remainder limit exceeded")
    ordered = sorted(
        minimal,
        key=lambda network: (int(network.network_address), network.prefixlen),
    )
    return tuple(network.with_prefixlen for network in ordered)
```

## Example

```python
parent = "192.0.2.0/28"
excluded = ("192.0.2.4/30", "192.0.2.12/31")
remaining = subtract_cidr_exclusions(parent, excluded)
reversed_input = subtract_cidr_exclusions(parent, tuple(reversed(excluded)))

parent_addresses = set(ip_network(parent))
excluded_addresses = {
    address
    for raw in excluded
    for address in ip_network(raw)
}
remaining_addresses = {
    address
    for raw in remaining
    for address in ip_network(raw)
}
minimal_again = tuple(
    network.with_prefixlen
    for network in collapse_addresses(ip_network(raw) for raw in remaining)
)

ipv6 = subtract_cidr_exclusions(
    "2001:db8::/124",
    ("2001:db8::4/126", "2001:db8::c/127"),
)
eight_disjoint = tuple(f"192.0.2.{index}/32" for index in range(0, 16, 2))
eight_result = subtract_cidr_exclusions("192.0.2.0/24", eight_disjoint)

invalid = 0
for invalid_parent, invalid_exclusions in (
    ("192.0.2.1/28", ()),
    (parent, ("192.0.2.0/29", "192.0.2.4/30")),
    (parent, ("192.0.2.16/32",)),
    (parent, ("2001:db8::/128",)),
    (parent, ("192.0.2.4/30", "192.0.2.4/30")),
    ("192.0.2.0/24", (*eight_disjoint, "192.0.2.17/32")),
):
    try:
        subtract_cidr_exclusions(invalid_parent, invalid_exclusions)
    except CidrDifferenceError:
        invalid += 1

assert (
    remaining == ("192.0.2.0/30", "192.0.2.8/30", "192.0.2.14/31")
    and reversed_input == remaining
    and remaining_addresses == parent_addresses - excluded_addresses
    and minimal_again == remaining
    and subtract_cidr_exclusions(parent, ()) == (parent,)
    and subtract_cidr_exclusions(parent, (parent,)) == ()
    and ipv6 == ("2001:db8::/126", "2001:db8::8/126", "2001:db8::e/127")
    and len(eight_result) <= 512
    and invalid == 6
)
```

## Trade-offs and Limitations

For `E` exclusions and at most `R` retained remainders, sorting and adjacent
overlap validation plus repeated subtraction and final canonicalization take
bounded `O(E log E + E * R + R log R)` wrapper work. IP integers have fixed
32-bit or 128-bit widths; the standard library's internal costs are separate.
The intermediate cap is checked after every exclusion, so memory growth is
bounded and deterministic because exclusions are sorted first.

The function deliberately rejects overlaps, duplicates, host bits, alternate
spellings, scoped addresses, mixed families, and exclusions outside the
parent. It returns an address set only: no DNS lookup, interface scope, route
selection, firewall action, authorization, or input metadata survives. A
remainder count near the cap is often a signal that a different representation
would be easier to operate.

## Related Snippets

<!-- catalog:related:start -->
- [Collapse a Bounded Same-Family CIDR Set Without Changing Its Address Union](collapse-a-bounded-same-family-cidr-set-without-changing-its-address-union.md)
- [Classify a Pre-Resolved IP Against a Bounded CIDR-Zone Snapshot](classify-a-pre-resolved-ip-against-a-bounded-cidr-zone-snapshot.md)
- [Subtract One Canonical Half-Open Integer Interval Set from Another](../algorithms-data-structures/subtract-one-canonical-half-open-integer-interval-set-from-another.md)
<!-- catalog:related:end -->
