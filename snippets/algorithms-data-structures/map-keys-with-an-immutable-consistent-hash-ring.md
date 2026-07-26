---
title: "Map Keys with an Immutable Consistent Hash Ring"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
tested_python:
  - "3.14"
dependencies: []
related:
  - ../reliability-resilience/assign-stable-schedule-slots-with-a-digest.md
  - ../storage-databases/build-and-apply-a-deterministic-mapping-delta.md
  - build-a-capacity-sized-bloom-filter.md
---

# Map Keys with an Immutable Consistent Hash Ring

## Idea and Problem

Map keys to changing node sets with a deterministic hash ring so membership changes move only the adjacent portions of the hash space.

Multiple virtual points per node improve expected balance. Building the ring as
an immutable value makes the membership, replica count, serialization, and
digest version explicit; lookup then uses precomputed sorted point arrays in
`O(log(RN))` time for `R` replicas and `N` nodes.

## When to Use

Use this algorithm for best-effort cache placement, worker affinity, or sharding
where the node set changes and reducing remapping is more important than exact
balance. Every participant must use the same identifier set, replica
count, and hash contract. Use rendezvous hashing when rebuilding a ring is less
convenient, or a coordinated storage system when ownership changes require
durable migration and consensus.

## Implementation

```python
import hashlib
from bisect import bisect_left
from collections.abc import Iterable
from dataclasses import dataclass, field


_HASH_VERSION = b"consistent-hash-ring-v1"


def _digest_point(domain: bytes, text: str, *, replica: int | None = None) -> int:
    encoded = text.encode("utf-8")
    digest = hashlib.blake2b(digest_size=16)
    digest.update(_HASH_VERSION)
    digest.update(b"\x00")
    digest.update(domain)
    digest.update(len(encoded).to_bytes(8, "big"))
    digest.update(encoded)
    if replica is not None:
        digest.update(replica.to_bytes(8, "big"))
    return int.from_bytes(digest.digest(), "big")


@dataclass(frozen=True, slots=True, init=False, eq=False)
class ConsistentHashRing:
    nodes: tuple[str, ...]
    replicas: int
    max_points: int
    _points: tuple[int, ...] = field(repr=False)
    _owners: tuple[str, ...] = field(repr=False)

    def __init__(
        self,
        nodes: Iterable[str],
        *,
        replicas: int = 128,
        max_points: int = 1_000_000,
    ) -> None:
        if isinstance(replicas, bool) or not isinstance(replicas, int):
            raise TypeError("replicas must be an integer")
        if not 1 <= replicas < 2**64:
            raise ValueError("replicas must be between 1 and 2**64 - 1")
        if isinstance(max_points, bool) or not isinstance(max_points, int):
            raise TypeError("max_points must be an integer")
        if max_points <= 0:
            raise ValueError("max_points must be positive")

        if isinstance(nodes, str):
            raise TypeError("nodes must be an iterable of identifiers, not text")
        supplied = tuple(nodes)
        if not supplied:
            raise ValueError("at least one node is required")
        if any(not isinstance(node, str) for node in supplied):
            raise TypeError("node identifiers must be text")
        if any(not node for node in supplied):
            raise ValueError("node identifiers must not be empty")
        if len(set(supplied)) != len(supplied):
            raise ValueError("node identifiers must be unique")
        if len(supplied) * replicas > max_points:
            raise ValueError("ring would exceed max_points")

        sorted_nodes = tuple(sorted(supplied))
        entries = sorted(
            (
                _digest_point(b"node", node, replica=replica),
                node,
                replica,
            )
            for node in sorted_nodes
            for replica in range(replicas)
        )
        object.__setattr__(self, "nodes", sorted_nodes)
        object.__setattr__(self, "replicas", replicas)
        object.__setattr__(self, "max_points", max_points)
        object.__setattr__(
            self,
            "_points",
            tuple(point for point, _node, _replica in entries),
        )
        object.__setattr__(
            self,
            "_owners",
            tuple(node for _point, node, _replica in entries),
        )

    def owner(self, key: str) -> str:
        if not isinstance(key, str):
            raise TypeError("key must be text")
        point = _digest_point(b"key", key)
        index = bisect_left(self._points, point)
        if index == len(self._points):
            index = 0
        return self._owners[index]
```

## Example

```python
from dataclasses import FrozenInstanceError


keys = [f"item-{number}" for number in range(2_000)]
before = ConsistentHashRing(["alpha", "beta", "gamma"], replicas=64)
reordered = ConsistentHashRing(["gamma", "alpha", "beta"], replicas=64)
added = ConsistentHashRing(["alpha", "beta", "gamma", "delta"], replicas=64)
removed = ConsistentHashRing(["alpha", "beta"], replicas=64)

before_owners = {key: before.owner(key) for key in keys}
added_owners = {key: added.owner(key) for key in keys}
removed_owners = {key: removed.owner(key) for key in keys}
changed_after_add = [key for key in keys if before_owners[key] != added_owners[key]]

try:
    before.nodes = ("other",)
except FrozenInstanceError:
    immutable = True
else:
    immutable = False

try:
    ConsistentHashRing(["alpha", "beta"], replicas=3, max_points=5)
except ValueError:
    point_limit_enforced = True
else:
    point_limit_enforced = False

assert (
    [reordered.owner(key) for key in keys] == [before_owners[key] for key in keys]
    and changed_after_add
    and all(added_owners[key] == "delta" for key in changed_after_add)
    and all(
        removed_owners[key] == owner
        for key, owner in before_owners.items()
        if owner != "gamma"
    )
    and immutable
    and point_limit_enforced
)
```

## Trade-offs and Limitations

Virtual nodes improve balance statistically but do not guarantee bounded load;
more replicas also increase construction time and memory until `max_points`
rejects the requested ring. Digest collisions are
ordered deterministically by node identifier and replica, but still collapse
points onto the same location. Changing the version prefix, text encoding,
digest, replica count, or node identifier changes placement and therefore needs
an explicit migration. This example has no weights, node health, replication,
adversarial-key defense, bounded-load rule, or distributed agreement on ring
configuration. Sampled movement tests demonstrate the invariant for their keys
but are not a proof of operational balance.

## Related Snippets

<!-- catalog:related:start -->
- [Assign Stable Schedule Slots with a Digest](../reliability-resilience/assign-stable-schedule-slots-with-a-digest.md)
- [Build and Apply a Deterministic Mapping Delta](../storage-databases/build-and-apply-a-deterministic-mapping-delta.md)
- [Build a Capacity-Sized Bloom Filter](build-a-capacity-sized-bloom-filter.md)
<!-- catalog:related:end -->
