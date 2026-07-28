---
title: "Classify a Pre-Resolved IP Against a Bounded CIDR-Zone Snapshot"
snippet_type: algorithm
use_cases:
  - caching
  - networking
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - parse-a-bounded-host-and-port-with-bracketed-ipv6.md
  - choose-grouped-endpoints-with-explicit-random-fairness.md
  - ../reliability-resilience/cache-values-with-a-monotonic-ttl-and-early-jitter.md
---

# Classify a Pre-Resolved IP Against a Bounded CIDR-Zone Snapshot

## Idea and Problem

Classify one already-resolved IP literal by longest-prefix match against an immutable, topology-identified CIDR-to-zone snapshot.

An exact, unexpired cache observation can supply the decision; every miss falls
back to the same deterministic snapshot scan. The result distinguishes a single
winning zone, conflicting zones at the winning prefix length, no coverage, and
a cache hit without resolving a hostname or choosing an endpoint.

## When to Use

Use this algorithm after a trusted component has resolved an IPv4 or IPv6
address and loaded one coherent routing, policy, or locality snapshot. Give each
semantically distinct snapshot a new topology ID, and pass the current finite
monotonic time from the same clock domain as the cache deadline.

The cache observation is trusted input, not evidence that its payload is
authentic. Accept it only from a source that protects the address, topology ID,
decision, and deadline together. The function reads no clock, invokes no
callback, performs no DNS or network operation, mutates no cache, and makes no
endpoint or authorization decision.

## Implementation

```python
import math
import re
from dataclasses import dataclass
from enum import StrEnum
from ipaddress import (
    IPv4Address,
    IPv4Network,
    IPv6Address,
    IPv6Network,
    ip_address,
    ip_network,
)

_MAX_ADDRESS_TEXT = 45
_MAX_CIDR_TEXT = 49
_MAX_ASSIGNMENTS = 256
_ZONE_TOKEN = re.compile(r"[a-z][a-z0-9-]{0,31}", re.ASCII)
_TOPOLOGY_TOKEN = re.compile(r"[A-Za-z0-9][A-Za-z0-9._-]{0,63}", re.ASCII)


class AddressFamily(StrEnum):
    IPV4 = "IPv4"
    IPV6 = "IPv6"


class ZoneStatus(StrEnum):
    MATCHED = "MATCHED"
    AMBIGUOUS = "AMBIGUOUS"
    UNCOVERED = "UNCOVERED"
    CACHED = "CACHED"


def _zone_token(value: object) -> str:
    if type(value) is not str:
        raise TypeError("zone_id must be an exact string")
    if _ZONE_TOKEN.fullmatch(value) is None:
        raise ValueError("zone_id must be a bounded lowercase ASCII token")
    return value


def _topology_token(value: object) -> str:
    if type(value) is not str:
        raise TypeError("topology_id must be an exact string")
    if _TOPOLOGY_TOKEN.fullmatch(value) is None:
        raise ValueError("topology_id must be a bounded conservative ASCII token")
    return value


def _literal(value: object, *, field_name: str) -> IPv4Address | IPv6Address:
    if type(value) is not str:
        raise TypeError(f"{field_name} must be an exact string")
    if not 1 <= len(value) <= _MAX_ADDRESS_TEXT or "%" in value:
        raise ValueError(f"{field_name} must be an unscoped IP literal")
    try:
        return ip_address(value)
    except ValueError:
        raise ValueError(f"{field_name} must be an exact IPv4 or IPv6 literal") from None


def _network(value: str) -> IPv4Network | IPv6Network:
    return ip_network(value, strict=True)


def _finite_time(value: object, *, field_name: str) -> int | float:
    if type(value) not in (int, float):
        raise TypeError(f"{field_name} must be an exact integer or float")
    if type(value) is float and not math.isfinite(value):
        raise ValueError(f"{field_name} must be finite")
    return value


def _base_status(zone_ids: tuple[str, ...]) -> ZoneStatus:
    if not zone_ids:
        return ZoneStatus.UNCOVERED
    if len(zone_ids) == 1:
        return ZoneStatus.MATCHED
    return ZoneStatus.AMBIGUOUS


@dataclass(frozen=True, slots=True)
class CidrZone:
    cidr: str
    zone_id: str

    def __post_init__(self) -> None:
        if type(self.cidr) is not str:
            raise TypeError("cidr must be an exact string")
        if not 1 <= len(self.cidr) <= _MAX_CIDR_TEXT or "%" in self.cidr:
            raise ValueError("cidr must be a bounded unscoped network")
        try:
            network = _network(self.cidr)
        except ValueError:
            raise ValueError("cidr must be an aligned IPv4 or IPv6 network") from None
        if self.cidr != network.with_prefixlen:
            raise ValueError("cidr must use canonical network notation")
        _zone_token(self.zone_id)


@dataclass(frozen=True, slots=True)
class ZoneSnapshot:
    topology_id: str
    family: AddressFamily
    assignments: tuple[CidrZone, ...]

    def __post_init__(self) -> None:
        _topology_token(self.topology_id)
        if type(self.family) is not AddressFamily:
            raise TypeError("family must be an exact AddressFamily value")
        if type(self.assignments) is not tuple:
            raise TypeError("assignments must be an exact tuple")
        if len(self.assignments) > _MAX_ASSIGNMENTS:
            raise ValueError("assignment count exceeds the supported limit")

        expected_version = 4 if self.family is AddressFamily.IPV4 else 6
        seen: set[tuple[str, str]] = set()
        for assignment in self.assignments:
            if type(assignment) is not CidrZone:
                raise TypeError("assignments must contain exact CidrZone values")
            if _network(assignment.cidr).version != expected_version:
                raise ValueError("every CIDR must match the snapshot family")
            identity = (assignment.cidr, assignment.zone_id)
            if identity in seen:
                raise ValueError("duplicate CIDR-to-zone assignments are forbidden")
            seen.add(identity)


@dataclass(frozen=True, slots=True)
class ZoneDecision:
    address: str
    topology_id: str
    status: ZoneStatus
    zone_ids: tuple[str, ...]
    cached_status: ZoneStatus | None = None

    def __post_init__(self) -> None:
        parsed = _literal(self.address, field_name="address")
        if self.address != parsed.compressed:
            raise ValueError("address must be normalized")
        _topology_token(self.topology_id)
        if type(self.status) is not ZoneStatus:
            raise TypeError("status must be an exact ZoneStatus value")
        if type(self.zone_ids) is not tuple:
            raise TypeError("zone_ids must be an exact tuple")
        for zone_id in self.zone_ids:
            _zone_token(zone_id)
        if self.zone_ids != tuple(sorted(set(self.zone_ids))):
            raise ValueError("zone_ids must be unique and sorted")

        effective = _base_status(self.zone_ids)
        if self.status is ZoneStatus.CACHED:
            if self.cached_status is not effective:
                raise ValueError("cached_status must describe the cached zone_ids")
        elif self.status is not effective or self.cached_status is not None:
            raise ValueError("status must describe zone_ids without cached_status")


@dataclass(frozen=True, slots=True)
class CacheObservation:
    decision: ZoneDecision
    expires_at: int | float

    def __post_init__(self) -> None:
        if type(self.decision) is not ZoneDecision:
            raise TypeError("decision must be an exact ZoneDecision value")
        if self.decision.status is ZoneStatus.CACHED:
            raise ValueError("a cache observation must contain a base decision")
        object.__setattr__(
            self,
            "expires_at",
            _finite_time(self.expires_at, field_name="expires_at"),
        )


def classify_ip_zone(
    address: str,
    snapshot: ZoneSnapshot,
    *,
    monotonic_now: int | float,
    cache: CacheObservation | None = None,
) -> ZoneDecision:
    if type(snapshot) is not ZoneSnapshot:
        raise TypeError("snapshot must be an exact ZoneSnapshot value")
    parsed = _literal(address, field_name="address")
    expected_version = 4 if snapshot.family is AddressFamily.IPV4 else 6
    if parsed.version != expected_version:
        raise ValueError("address family does not match the snapshot")
    now = _finite_time(monotonic_now, field_name="monotonic_now")
    normalized = parsed.compressed

    if cache is not None:
        if type(cache) is not CacheObservation:
            raise TypeError("cache must be an exact CacheObservation value")
        cached = cache.decision
        if (
            cached.address == normalized
            and cached.topology_id == snapshot.topology_id
            and now < cache.expires_at
        ):
            return ZoneDecision(
                normalized,
                snapshot.topology_id,
                ZoneStatus.CACHED,
                cached.zone_ids,
                cached.status,
            )

    longest_prefix = -1
    winning_zones: set[str] = set()
    for assignment in snapshot.assignments:
        network = _network(assignment.cidr)
        if parsed not in network:
            continue
        prefix_length = network.prefixlen
        if prefix_length > longest_prefix:
            longest_prefix = prefix_length
            winning_zones = {assignment.zone_id}
        elif prefix_length == longest_prefix:
            winning_zones.add(assignment.zone_id)

    zone_ids = tuple(sorted(winning_zones))
    return ZoneDecision(
        normalized,
        snapshot.topology_id,
        _base_status(zone_ids),
        zone_ids,
    )
```

## Example

```python
snapshot = ZoneSnapshot(
    "map.12",
    AddressFamily.IPV4,
    (
        CidrZone("192.0.2.0/24", "outer"),
        CidrZone("192.0.2.128/25", "violet"),
        CidrZone("192.0.2.128/25", "amber"),
    ),
)

broad = classify_ip_zone("192.0.2.10", snapshot, monotonic_now=40)
tie = classify_ip_zone("192.0.2.200", snapshot, monotonic_now=40)
miss = classify_ip_zone("203.0.113.9", snapshot, monotonic_now=40)
observation = CacheObservation(broad, expires_at=50)
hit = classify_ip_zone(
    "192.0.2.10",
    snapshot,
    monotonic_now=49.5,
    cache=observation,
)
at_expiry = classify_ip_zone(
    "192.0.2.10",
    snapshot,
    monotonic_now=50,
    cache=observation,
)
try:
    CidrZone("fe80::%en0/64", "violet")
except ValueError:
    scoped_cidr_rejected = True
else:
    scoped_cidr_rejected = False
large_integer_hit = classify_ip_zone(
    "192.0.2.10",
    snapshot,
    monotonic_now=2**60,
    cache=CacheObservation(broad, expires_at=2**60 + 1),
)

ipv6_snapshot = ZoneSnapshot(
    "map.v6",
    AddressFamily.IPV6,
    (CidrZone("2001:db8::/32", "violet"),),
)
ipv6 = classify_ip_zone("2001:0db8::7", ipv6_snapshot, monotonic_now=3)

assert (
    (broad.status, broad.zone_ids),
    (tie.status, tie.zone_ids),
    (miss.status, miss.zone_ids),
    (hit.status, hit.cached_status, hit.zone_ids),
    at_expiry.status,
    scoped_cidr_rejected,
    large_integer_hit.status,
    (ipv6.address, ipv6.status),
) == (
    (ZoneStatus.MATCHED, ("outer",)),
    (ZoneStatus.AMBIGUOUS, ("amber", "violet")),
    (ZoneStatus.UNCOVERED, ()),
    (ZoneStatus.CACHED, ZoneStatus.MATCHED, ("outer",)),
    ZoneStatus.MATCHED,
    True,
    ZoneStatus.CACHED,
    ("2001:db8::7", ZoneStatus.MATCHED),
)
```

## Trade-offs and Limitations

Classification reparses at most 256 canonical CIDR strings into short-lived
standard-library network objects, so its time cost is linear in the snapshot
size. No parsed network object is stored in or exposed by an assignment, so
mutating some other parse cannot change the frozen string snapshot.
More-specific CIDRs intentionally override broader ones. At one prefix length,
two canonical CIDRs cannot overlap unless they name the same network; identical
network-and-zone duplicates are rejected, while the same network assigned to
different zones is retained and reported as `AMBIGUOUS` instead of being
resolved by input order.

A cache hit trusts the stored base decision and does not compare it with the
CIDR records. Exact normalized address and case-sensitive topology ID equality
are therefore the complete identity contract, and a topology ID must never be
reused after its mappings change. A deadline is live only while
`monotonic_now < expires_at`; equality is expired. Both values must come from
the same process-local monotonic epoch. Integer readings remain exact rather
than being rounded through `float`; mixed integer/float comparisons use the
represented float value. Persisted deadlines generally cannot survive a
restart.

The fixed ASCII identifiers exclude richer naming schemes, and scoped IPv6
literals are deliberately unsupported. The snapshot is read-only and may be
empty; rebuilding or indexing it, resolving names, refreshing or writing a
cache, selecting endpoints, checking reachability, and enforcing access policy
remain separate responsibilities.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Bounded Host and Port with Bracketed IPv6](parse-a-bounded-host-and-port-with-bracketed-ipv6.md)
- [Choose Grouped Endpoints with Explicit Random Fairness](choose-grouped-endpoints-with-explicit-random-fairness.md)
- [Cache Values with a Monotonic TTL and Early Jitter](../reliability-resilience/cache-values-with-a-monotonic-ttl-and-early-jitter.md)
<!-- catalog:related:end -->
