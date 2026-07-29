---
title: "Build and Verify a Bounded SHA-256 Merkle Inclusion Proof"
snippet_type: recipe
use_cases:
  - security
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - authenticate-bounded-payloads-with-versioned-hmac-keys.md
  - verify-a-bounded-byte-stream-before-returning-its-payload.md
  - ../storage-databases/store-bytes-by-their-content-digest.md
---

# Build and Verify a Bounded SHA-256 Merkle Inclusion Proof

## Idea and Problem

Commit to an ordered bounded tuple of byte leaves and prove that one exact leaf occupies one exact position without sending the full tuple.

The tree uses byte-exact domain separation and four-byte unsigned big-endian
lengths. Odd-width levels duplicate their final digest, while a root envelope
also commits to the leaf count. A proof carries one explicitly oriented sibling
per level, including a verifiable self-sibling when its path owns an odd final
node.

## When to Use

Use this recipe for a small immutable batch when a verifier already possesses
an independently trusted root and needs one positional membership check. All
producers and verifiers must use this exact framing, ordering, odd-node rule,
and count envelope.

Use a reviewed authenticated protocol when roots must be distributed, signed,
rotated, or associated with an identity. Use a purpose-built persistent Merkle
tree when leaves change, or when append/update proofs, multiproofs, or
incremental storage matter.

## Implementation

```python
import hmac
from dataclasses import dataclass
from hashlib import sha256

_MAX_LEAVES = 4_096
_MAX_LEAF_BYTES = 65_536
_MAX_TOTAL_LEAF_BYTES = 4 * 1024 * 1024
_SHA256_BYTES = 32
_LEFT = "left"
_RIGHT = "right"


@dataclass(frozen=True, slots=True)
class MerkleProofStep:
    sibling: bytes
    side: str


@dataclass(frozen=True, slots=True)
class MerkleProof:
    leaf_index: int
    leaf_count: int
    steps: tuple[MerkleProofStep, ...]


def _uint32be(value: int) -> bytes:
    return value.to_bytes(4, "big")


def _leaf_digest(leaf: bytes) -> bytes:
    digest = sha256()
    digest.update(b"\x00")
    digest.update(_uint32be(len(leaf)))
    digest.update(leaf)
    return digest.digest()


def _node_digest(left: bytes, right: bytes) -> bytes:
    digest = sha256()
    digest.update(b"\x01")
    digest.update(left)
    digest.update(right)
    return digest.digest()


def _root_digest(leaf_count: int, tree_digest: bytes) -> bytes:
    digest = sha256()
    digest.update(b"\x02")
    digest.update(_uint32be(leaf_count))
    digest.update(tree_digest)
    return digest.digest()


def _validate_leaf_batch(leaves: object) -> tuple[bytes, ...]:
    if type(leaves) is not tuple:
        raise TypeError("leaves must be an exact tuple")
    if not 1 <= len(leaves) <= _MAX_LEAVES:
        raise ValueError("leaf count is outside the supported range")

    total_bytes = 0
    for leaf in leaves:
        if type(leaf) is not bytes:
            raise TypeError("leaves must contain exact bytes")
        if len(leaf) > _MAX_LEAF_BYTES:
            raise ValueError("a leaf exceeds the supported byte limit")
        total_bytes += len(leaf)
        if total_bytes > _MAX_TOTAL_LEAF_BYTES:
            raise ValueError("aggregate leaf bytes exceed the supported limit")
    return leaves


def build_merkle_proof(
    leaves: tuple[bytes, ...],
    target_index: int,
) -> tuple[bytes, MerkleProof]:
    leaves = _validate_leaf_batch(leaves)
    if type(target_index) is not int:
        raise TypeError("target_index must be an exact integer")
    if not 0 <= target_index < len(leaves):
        raise ValueError("target_index is outside the leaf tuple")

    level = [_leaf_digest(leaf) for leaf in leaves]
    current_index = target_index
    steps: list[MerkleProofStep] = []

    while len(level) > 1:
        width = len(level)
        if current_index == width - 1 and width % 2:
            steps.append(MerkleProofStep(level[current_index], _RIGHT))
        elif current_index % 2:
            steps.append(MerkleProofStep(level[current_index - 1], _LEFT))
        else:
            steps.append(MerkleProofStep(level[current_index + 1], _RIGHT))

        if width % 2:
            level.append(level[-1])
        level = [
            _node_digest(level[offset], level[offset + 1]) for offset in range(0, len(level), 2)
        ]
        current_index //= 2

    proof = MerkleProof(target_index, len(leaves), tuple(steps))
    return _root_digest(len(leaves), level[0]), proof


def verify_merkle_proof(
    leaf: bytes,
    proof: MerkleProof,
    trusted_root: bytes,
) -> bool:
    if type(leaf) is not bytes or len(leaf) > _MAX_LEAF_BYTES:
        return False
    if type(proof) is not MerkleProof:
        return False
    if type(trusted_root) is not bytes or len(trusted_root) != _SHA256_BYTES:
        return False
    if type(proof.leaf_count) is not int or not 1 <= proof.leaf_count <= _MAX_LEAVES:
        return False
    if type(proof.leaf_index) is not int or not 0 <= proof.leaf_index < proof.leaf_count:
        return False
    if type(proof.steps) is not tuple:
        return False
    if len(proof.steps) != (proof.leaf_count - 1).bit_length():
        return False
    for step in proof.steps:
        if type(step) is not MerkleProofStep:
            return False
        if type(step.sibling) is not bytes or len(step.sibling) != _SHA256_BYTES:
            return False
        if type(step.side) is not str or step.side not in {_LEFT, _RIGHT}:
            return False

    current = _leaf_digest(leaf)
    current_index = proof.leaf_index
    width = proof.leaf_count

    for step in proof.steps:
        if current_index == width - 1 and width % 2:
            if step.side != _RIGHT or not hmac.compare_digest(step.sibling, current):
                return False
            current = _node_digest(current, current)
        elif current_index % 2:
            if step.side != _LEFT:
                return False
            current = _node_digest(step.sibling, current)
        else:
            if step.side != _RIGHT:
                return False
            current = _node_digest(current, step.sibling)

        current_index //= 2
        width = (width + 1) // 2

    expected_root = _root_digest(proof.leaf_count, current)
    return hmac.compare_digest(expected_root, trusted_root)
```

## Example

```python
leaves = (b"red", b"green", b"blue")
root, proof = build_merkle_proof(leaves, 2)

tampered_steps = (
    MerkleProofStep(b"\x00" * _SHA256_BYTES, _RIGHT),
    *proof.steps[1:],
)
tampered_proof = MerkleProof(2, 3, tampered_steps)
wrong_count = MerkleProof(2, 4, proof.steps)

single_root, single_proof = build_merkle_proof((b"only",), 0)

assert (
    root.hex(),
    proof.steps[0],
    len(proof.steps),
    verify_merkle_proof(leaves[2], proof, root),
    verify_merkle_proof(b"changed", proof, root),
    verify_merkle_proof(leaves[2], tampered_proof, root),
    verify_merkle_proof(leaves[2], wrong_count, root),
    single_proof.steps,
    verify_merkle_proof(b"only", single_proof, single_root),
) == (
    "0f41381d0e94bbdc37ce6883a03b2914005a0d59c29a108005d38eacaf67d05d",
    MerkleProofStep(_leaf_digest(b"blue"), _RIGHT),
    2,
    True,
    False,
    False,
    False,
    (),
    True,
)
```

## Trade-offs and Limitations

Building the tree hashes `B` bytes of leaf content and performs linear
additional tree work, for `O(B + n)` time and `O(n)` auxiliary memory.
Verifying a selected leaf of size `b` takes `O(b + log n)` time and constant
auxiliary memory beyond the supplied proof. The proof has exactly
`(n - 1).bit_length()` 32-byte siblings plus orientations.

Membership is meaningful only relative to a root obtained through a trusted
channel. SHA-256 and constant-time root comparison do not sign that root,
identify its producer, or establish freshness. This exact convention is not
compatible with trees that omit duplicate siblings, use different domain
bytes, omit the count envelope, or serialize integers differently. The recipe
does not support mutable trees, append or update proofs, multiproofs,
persistence, or remote proof exchange.

## Related Snippets

<!-- catalog:related:start -->
- [Authenticate Bounded Payloads with Versioned HMAC Keys](authenticate-bounded-payloads-with-versioned-hmac-keys.md)
- [Verify a Bounded Byte Stream Before Returning Its Payload](verify-a-bounded-byte-stream-before-returning-its-payload.md)
- [Store Bytes by Their Content Digest](../storage-databases/store-bytes-by-their-content-digest.md)
<!-- catalog:related:end -->
