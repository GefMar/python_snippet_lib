---
title: "Select Top-K Nodes for One Key with Deterministic Rendezvous Hashing"
snippet_type: algorithm
use_cases:
  - data-transformation
  - resource-management
  - performance-optimization
tested_python:
  - "3.14"
dependencies: []
related:
  - map-keys-with-an-immutable-consistent-hash-ring.md
  - ../reliability-resilience/assign-stable-schedule-slots-with-a-digest.md
  - ../networking-protocols/choose-grouped-endpoints-with-explicit-random-fairness.md
---

# Select Top-K Nodes for One Key with Deterministic Rendezvous Hashing

## Idea and Problem

Rank a bounded immutable node set for one byte key by assigning every node a versioned SHA-256 score, then select the highest-scoring prefix.

Rendezvous hashing needs no prebuilt ring. Removing a node leaves the relative
order of all survivors unchanged, while adding a node only inserts that node
into the existing ranking. Fixed-width length frames keep the key and node
identifier boundaries unambiguous. A digest collision is resolved by the
ascending UTF-8 node identifier, so input order never affects the result.

## When to Use

Use this function for reproducible cache affinity, best-effort replica
selection, or deterministic worker preference when membership is a small
coherent snapshot. All participants must use exactly the same byte key, node
identifiers, digest version, framing, and membership snapshot.

Use a consistent hash ring when repeated lookups must avoid hashing every node.
Use a coordinated placement service when capacity, health, failover, or durable
ownership transitions are part of correctness.

## Implementation

```python
import hashlib
from itertools import permutations

_RENDEZVOUS_VERSION = b"rendezvous-sha256-v1"
_MAX_KEY_BYTES = 1_024
_MAX_NODES = 256
_MAX_NODE_BYTES = 255


def _rendezvous_score(key: bytes, node: bytes) -> bytes:
    digest = hashlib.sha256()
    digest.update(_RENDEZVOUS_VERSION)
    digest.update(len(key).to_bytes(2, "big"))
    digest.update(key)
    digest.update(len(node).to_bytes(1, "big"))
    digest.update(node)
    return digest.digest()


def select_rendezvous_nodes(
    key: bytes,
    nodes: tuple[str, ...],
    *,
    top_k: int,
) -> tuple[str, ...]:
    """Return the deterministic highest-scoring node prefix for one key."""
    if type(key) is not bytes:
        raise TypeError("key must be exact bytes")
    if len(key) > _MAX_KEY_BYTES:
        raise ValueError("key exceeds the supported byte limit")
    if type(nodes) is not tuple:
        raise TypeError("nodes must be an exact tuple")
    if not 1 <= len(nodes) <= _MAX_NODES:
        raise ValueError("node count is outside the supported range")
    if type(top_k) is not int:
        raise TypeError("top_k must be an exact non-boolean integer")
    if not 1 <= top_k <= len(nodes):
        raise ValueError("top_k must select between one and every node")

    encoded_nodes: list[tuple[bytes, str]] = []
    seen: set[str] = set()
    for position, node in enumerate(nodes):
        if type(node) is not str:
            raise TypeError(f"nodes[{position}] must be an exact string")
        if node in seen:
            raise ValueError(f"nodes[{position}] duplicates an earlier identifier")
        seen.add(node)
        try:
            encoded = node.encode("utf-8")
        except UnicodeEncodeError:
            raise ValueError(
                f"nodes[{position}] must be valid UTF-8 text"
            ) from None
        if not 1 <= len(encoded) <= _MAX_NODE_BYTES:
            raise ValueError(
                f"nodes[{position}] is outside the encoded byte limit"
            )
        encoded_nodes.append((encoded, node))

    scored = [
        (_rendezvous_score(key, encoded), encoded, node)
        for encoded, node in encoded_nodes
    ]
    scored.sort(key=lambda item: (-int.from_bytes(item[0]), item[1]))
    return tuple(node for _score, _encoded, node in scored[:top_k])
```

## Example

```python
def oracle_ranking(key: bytes, nodes: tuple[str, ...]) -> tuple[str, ...]:
    scored: list[tuple[int, bytes, str]] = []
    for node in nodes:
        encoded = node.encode("utf-8")
        framed = (
            b"rendezvous-sha256-v1"
            + len(key).to_bytes(2, "big")
            + key
            + len(encoded).to_bytes(1, "big")
            + encoded
        )
        score = int.from_bytes(hashlib.sha256(framed).digest())
        scored.append((score, encoded, node))
    scored.sort(key=lambda item: (-item[0], item[1]))
    return tuple(node for _score, _encoded, node in scored)


key = b"bounded-placement-key"
nodes = ("alpha", "beta", "gamma", "delta")
expected = oracle_ranking(key, nodes)

for declared_order in permutations(nodes):
    declaration = tuple(declared_order)
    for top_k in range(1, len(nodes) + 1):
        assert select_rendezvous_nodes(
            key,
            declaration,
            top_k=top_k,
        ) == expected[:top_k]

for removed in nodes:
    survivors = tuple(node for node in nodes if node != removed)
    survivor_ranking = select_rendezvous_nodes(
        key,
        survivors,
        top_k=len(survivors),
    )
    assert survivor_ranking == tuple(node for node in expected if node != removed)

base_nodes = nodes[:-1]
base_ranking = select_rendezvous_nodes(key, base_nodes, top_k=len(base_nodes))
added_ranking = select_rendezvous_nodes(key, nodes, top_k=len(nodes))

boundary_nodes = tuple(f"node-{index:03d}" for index in range(_MAX_NODES))
boundary_ranking = select_rendezvous_nodes(
    b"k" * _MAX_KEY_BYTES,
    boundary_nodes,
    top_k=_MAX_NODES,
)
maximum_utf8_node = "\u00e9" * 127 + "a"
assert len(maximum_utf8_node.encode("utf-8")) == _MAX_NODE_BYTES
assert select_rendezvous_nodes(b"", (maximum_utf8_node,), top_k=1) == (
    maximum_utf8_node,
)

invalid_calls = (
    lambda: select_rendezvous_nodes(key, nodes, top_k=0),
    lambda: select_rendezvous_nodes(key, ("alpha", "alpha"), top_k=1),
    lambda: select_rendezvous_nodes(key, ("x" * 256,), top_k=1),
)
for invalid_call in invalid_calls:
    try:
        invalid_call()
    except ValueError:
        pass
    else:
        raise AssertionError("invalid rendezvous input was accepted")

assert (
    select_rendezvous_nodes(key, nodes, top_k=2) == expected[:2]
    and tuple(node for node in added_ranking if node in base_nodes) == base_ranking
    and len(boundary_ranking) == _MAX_NODES
    and set(boundary_ranking) == set(boundary_nodes)
    and _rendezvous_score(b"a", b"bc") != _rendezvous_score(b"ab", b"c")
)
```

## Trade-offs and Limitations

For `N` nodes, key length `K`, and total encoded identifier length `I`, this
direct implementation hashes `O(NK + I)` bytes, sorts in `O(N log N)` time,
and stores `O(N)` scores. The explicit bounds keep that repeated key hashing
and the complete ranking small. A reusable prehashed key prefix could reduce
constant work when many membership snapshots share one key.

Changing the version bytes, field widths, SHA-256, UTF-8 encoding, or tie rule
changes placement and requires a migration. The public unkeyed digest does not
defend against adversarially chosen keys or nodes. This function has no
weights, health signals, load bound, mutable membership, distributed
configuration agreement, or guarantee of balanced placement for a finite key
set.

## Related Snippets

<!-- catalog:related:start -->
- [Map Keys with an Immutable Consistent Hash Ring](map-keys-with-an-immutable-consistent-hash-ring.md)
- [Assign Stable Schedule Slots with a Digest](../reliability-resilience/assign-stable-schedule-slots-with-a-digest.md)
- [Choose Grouped Endpoints with Explicit Random Fairness](../networking-protocols/choose-grouped-endpoints-with-explicit-random-fairness.md)
<!-- catalog:related:end -->
