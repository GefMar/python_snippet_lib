---
title: "Report Partition Offsets Behind a Fixed Checkpoint"
snippet_type: algorithm
use_cases:
  - observability
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - classify-required-health-stamps-by-freshness.md
  - ../storage-databases/build-and-apply-a-deterministic-mapping-delta.md
---

# Report Partition Offsets Behind a Fixed Checkpoint

## Idea and Problem

Compare current partition offsets with one fixed target checkpoint and report every positive gap in deterministic order.

A single aggregate offset can hide a stalled partition. Indexing both snapshots
by `(shard, partition)` makes topology equality explicit before comparison and
keeps the result useful for logs, metrics, or a higher-level health policy.
Returning structured gaps also distinguishes a validated zero-gap result from
a missing or incompatible baseline.

## When to Use

Use this algorithm when a caller has two finite, atomically captured offset
snapshots for the same immutable partition topology. The target may be a
previously recorded source checkpoint or any other fixed progress boundary.
Use broker-specific monitoring when partitions can be added, removed,
truncated, renumbered, or moved between offset epochs.

## Implementation

```python
from collections.abc import Iterable
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class PartitionOffset:
    shard: str
    partition: int
    offset: int


@dataclass(frozen=True, slots=True)
class OffsetGap:
    shard: str
    partition: int
    target_offset: int
    current_offset: int
    behind_by: int


def _index_offsets(
    offsets: Iterable[PartitionOffset],
    *,
    name: str,
) -> dict[tuple[str, int], int]:
    indexed: dict[tuple[str, int], int] = {}
    for item in offsets:
        if not isinstance(item, PartitionOffset):
            raise TypeError(f"{name} must contain PartitionOffset values")
        if not isinstance(item.shard, str):
            raise TypeError("shard must be text")
        if not item.shard:
            raise ValueError("shard must not be empty")
        if isinstance(item.partition, bool) or not isinstance(item.partition, int):
            raise TypeError("partition must be an integer")
        if item.partition < 0:
            raise ValueError("partition must not be negative")
        if isinstance(item.offset, bool) or not isinstance(item.offset, int):
            raise TypeError("offset must be an integer")
        if item.offset < 0:
            raise ValueError("offset must not be negative")

        key = (item.shard, item.partition)
        if key in indexed:
            raise ValueError(f"duplicate partition key in {name}: {key!r}")
        indexed[key] = item.offset

    if not indexed:
        raise ValueError(f"{name} must not be empty")
    return indexed


def report_offset_gaps(
    current: Iterable[PartitionOffset],
    target: Iterable[PartitionOffset],
) -> tuple[OffsetGap, ...]:
    current_by_key = _index_offsets(current, name="current")
    target_by_key = _index_offsets(target, name="target")
    if current_by_key.keys() != target_by_key.keys():
        raise ValueError("current and target partition keys must be identical")

    gaps = []
    for shard, partition in sorted(target_by_key):
        target_offset = target_by_key[(shard, partition)]
        current_offset = current_by_key[(shard, partition)]
        if current_offset < target_offset:
            gaps.append(
                OffsetGap(
                    shard=shard,
                    partition=partition,
                    target_offset=target_offset,
                    current_offset=current_offset,
                    behind_by=target_offset - current_offset,
                )
            )
    return tuple(gaps)
```

## Example

```python
target = [
    PartitionOffset("west", 1, 240),
    PartitionOffset("east", 0, 120),
    PartitionOffset("west", 0, 200),
]
current = [
    PartitionOffset("west", 0, 205),
    PartitionOffset("west", 1, 210),
    PartitionOffset("east", 0, 120),
]

gaps = report_offset_gaps(current, target)

try:
    report_offset_gaps(
        current,
        [*target, PartitionOffset("east", 1, 0)],
    )
except ValueError:
    topology_mismatch_rejected = True
else:
    topology_mismatch_rejected = False

assert (gaps, topology_mismatch_rejected) == (
    (OffsetGap("west", 1, 240, 210, 30),),
    True,
)
```

## Trade-offs and Limitations

Both iterables are materialized into dictionaries, using `O(n)` memory, and
the keys are sorted in `O(n log n)` time. Exact topology equality is
deliberately strict; a caller that supports partition migration must reconcile
that state before comparison. A zero-gap result says only that every current
offset reached this fixed checkpoint. It does not measure current source lag,
freshness, throughput, data correctness, or consumer health, and it cannot
detect resets when identical integer offsets belong to different epochs.
Capture coherent snapshots and retain their timestamp and epoch metadata in
the surrounding monitoring system.

## Related Snippets

<!-- catalog:related:start -->
- [Classify Required Health Stamps by Freshness](classify-required-health-stamps-by-freshness.md)
- [Build and Apply a Deterministic Mapping Delta](../storage-databases/build-and-apply-a-deterministic-mapping-delta.md)
<!-- catalog:related:end -->
