---
title: "Recover One Missing Equal-Length Byte Shard with XOR Parity"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - ../algorithms-data-structures/build-a-canonical-reduced-xor-basis-for-bounded-unsigned-integers.md
  - ../configuration-serialization/append-a-fixed-width-crc-check-to-a-human-readable-identifier.md
  - ../storage-databases/validate-a-bounded-chunk-manifest-before-a-conditional-version-switch.md
---

# Recover One Missing Equal-Length Byte Shard with XOR Parity

## Idea and Problem

Add one bytewise parity shard that can reconstruct exactly one known missing position among equal-length data shards.

XOR is associative, commutative, and its own inverse. XORing every data byte at
the same offset produces parity; XORing that parity with all surviving data
shards cancels the survivors and leaves the missing bytes. No finite-field
tables or external dependency is required.

The positional tuple supplied to recovery contains exactly one `None`, so the
lost position is explicit. This matters because parity alone cannot identify
which shard is absent or distinguish loss from corruption.

## When to Use

Use one XOR parity shard for bounded local fixtures, simple striped storage
experiments, or recovery tests where shard boundaries and the missing index are
already trusted. Equal lengths and a single erasure make the behavior easy to
audit.

Use an established erasure-coding system for multiple losses, large data,
streaming repair, shard identity metadata, or interoperability. Authenticate or
checksum every shard separately when accidental or adversarial corruption must
be detected before recovery.

## Implementation

```python
from dataclasses import dataclass
from itertools import product
from random import Random

_MIN_XOR_DATA_SHARDS = 2
_MAX_XOR_DATA_SHARDS = 32
_MAX_XOR_SHARD_BYTES = 1_048_576
_MAX_XOR_AGGREGATE_BYTES = 16 * 1_048_576


@dataclass(frozen=True, slots=True)
class RecoveredShard:
    index: int
    data: bytes


def xor_parity_shard(shards: tuple[bytes, ...]) -> bytes:
    """Return one bytewise-XOR parity shard for bounded equal-length data."""
    if type(shards) is not tuple:
        raise TypeError("shards must be an exact tuple")
    if not _MIN_XOR_DATA_SHARDS <= len(shards) <= _MAX_XOR_DATA_SHARDS:
        raise ValueError("data shard count is outside 2..32")

    shard_size: int | None = None
    for index, shard in enumerate(shards):
        if type(shard) is not bytes:
            raise TypeError(f"shards[{index}] must be exact bytes")
        if shard_size is None:
            shard_size = len(shard)
            if not 1 <= shard_size <= _MAX_XOR_SHARD_BYTES:
                raise ValueError("shard size is outside 1..1048576 bytes")
        elif len(shard) != shard_size:
            raise ValueError("all data shards must have equal length")
    if shard_size is None or len(shards) * shard_size > _MAX_XOR_AGGREGATE_BYTES:
        raise ValueError("aggregate data exceeds 16 MiB")

    parity = bytearray(shard_size)
    for shard in shards:
        for offset, value in enumerate(shard):
            parity[offset] ^= value
    return bytes(parity)


def recover_known_missing_xor_shard(
    shards: tuple[bytes | None, ...],
    parity: bytes,
) -> RecoveredShard:
    """Recover the one position explicitly marked None."""
    if type(shards) is not tuple:
        raise TypeError("shards must be an exact tuple")
    if not _MIN_XOR_DATA_SHARDS <= len(shards) <= _MAX_XOR_DATA_SHARDS:
        raise ValueError("data shard count is outside 2..32")
    if type(parity) is not bytes:
        raise TypeError("parity must be exact bytes")
    shard_size = len(parity)
    if not 1 <= shard_size <= _MAX_XOR_SHARD_BYTES:
        raise ValueError("parity size is outside 1..1048576 bytes")
    if len(shards) * shard_size > _MAX_XOR_AGGREGATE_BYTES:
        raise ValueError("aggregate data exceeds 16 MiB")

    missing_indexes = tuple(index for index, shard in enumerate(shards) if shard is None)
    if len(missing_indexes) != 1:
        raise ValueError("shards must contain exactly one missing position")
    for index, shard in enumerate(shards):
        if shard is None:
            continue
        if type(shard) is not bytes:
            raise TypeError(f"shards[{index}] must be exact bytes or None")
        if len(shard) != shard_size:
            raise ValueError("present shards and parity must have equal length")

    recovered = bytearray(parity)
    for shard in shards:
        if shard is not None:
            for offset, value in enumerate(shard):
                recovered[offset] ^= value
    return RecoveredShard(missing_indexes[0], bytes(recovered))
```

## Example

```python
def column_parity_oracle(shards: tuple[bytes, ...]) -> bytes:
    return bytes(
        sum((sum((shard[offset] >> bit) & 1 for shard in shards) % 2) << bit for bit in range(8))
        for offset in range(len(shards[0]))
    )


data_shards = (b"alpha", b"bravo", b"cider")
parity = xor_parity_shard(data_shards)
assert recover_known_missing_xor_shard(
    (data_shards[0], None, data_shards[2]),
    parity,
) == RecoveredShard(1, b"bravo")

checked = 0
for shard_count in range(2, 5):
    for shard_size in range(1, 3):
        for flat_values in product(range(3), repeat=shard_count * shard_size):
            shards = tuple(
                bytes(flat_values[start : start + shard_size])
                for start in range(0, len(flat_values), shard_size)
            )
            parity = xor_parity_shard(shards)
            assert parity == column_parity_oracle(shards)
            assert xor_parity_shard((*shards, parity)) == bytes(shard_size)
            for missing_index in range(shard_count):
                incomplete = tuple(
                    None if index == missing_index else shard for index, shard in enumerate(shards)
                )
                assert recover_known_missing_xor_shard(
                    incomplete,
                    parity,
                ) == RecoveredShard(missing_index, shards[missing_index])
                checked += 1

rng = Random(0x50_5A)
for _ in range(1_000):
    shard_count = rng.randrange(2, 10)
    shards = tuple(rng.randbytes(rng.randrange(1, 65)) for _ in range(shard_count))
    shard_size = len(shards[0])
    shards = tuple(shard[:shard_size].ljust(shard_size, b"\x00") for shard in shards)
    missing_index = rng.randrange(shard_count)
    parity = xor_parity_shard(shards)
    incomplete = tuple(
        None if index == missing_index else shard for index, shard in enumerate(shards)
    )
    assert recover_known_missing_xor_shard(incomplete, parity).data == shards[missing_index]

assert checked == 29_016
```

## Trade-offs and Limitations

For `K` shards of `S` bytes, parity creation and recovery take `O(KS)` time
and one `O(S)` output buffer. The straightforward Python byte loops favor
readability and exact behavior over vectorized throughput. The 16 MiB aggregate
cap bounds worst-case work.

One parity shard recovers only one known erasure. It cannot locate an unknown
missing position, recover two losses, detect a flipped byte, identify a stale
shard, authenticate data, or substitute for Reed-Solomon coding. Shard order,
length, identity, storage, and integrity metadata remain the caller's separate
responsibility.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Canonical Reduced XOR Basis for Bounded Unsigned Integers](../algorithms-data-structures/build-a-canonical-reduced-xor-basis-for-bounded-unsigned-integers.md)
- [Append a Fixed-Width CRC Check to a Human-Readable Identifier](../configuration-serialization/append-a-fixed-width-crc-check-to-a-human-readable-identifier.md)
- [Validate a Bounded Chunk Manifest Before a Conditional Version Switch](../storage-databases/validate-a-bounded-chunk-manifest-before-a-conditional-version-switch.md)
<!-- catalog:related:end -->
