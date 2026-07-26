---
title: "Select One Record per Key with an Explicit Ranking Rule"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - aggregate-consecutive-values-into-weighted-runs.md
  - normalize-optional-csv-columns-in-a-single-pass.md
  - ../storage-databases/build-and-apply-a-deterministic-mapping-delta.md
---

# Select One Record per Key with an Explicit Ranking Rule

## Idea and Problem

Choose one record per logical key with a stable documented ranking rule and an audit of every group decision.

The caller supplies an identity function and a non-empty tuple of integer rank
components. The greatest lexicographic rank wins, equal ranks keep the earlier
input, and survivor groups remain in order of their first appearance. Identity
and rank are evaluated exactly once for each record.

## When to Use

Use this algorithm for a finite in-memory record stream when duplicate identity
is exact and the business preference can be expressed as ordered integer
components, such as `(revision, completeness)`. Define every component's
direction before calling: negate a component when a smaller value should win.
Use entity resolution or domain review instead when identity is fuzzy or a
ranking rule would discard materially conflicting facts.

## Implementation

```python
from collections.abc import Callable, Hashable, Iterable
from dataclasses import dataclass
from typing import Generic, TypeVar


T = TypeVar("T")


@dataclass(frozen=True, slots=True)
class SurvivorshipAudit:
    identity: Hashable
    group_size: int
    selected_index: int
    selected_rank: tuple[int, ...]


@dataclass(frozen=True, slots=True)
class SurvivorshipResult(Generic[T]):
    records: tuple[T, ...]
    audit: tuple[SurvivorshipAudit, ...]


@dataclass(slots=True)
class _Group(Generic[T]):
    identity: Hashable
    record: T
    selected_index: int
    selected_rank: tuple[int, ...]
    size: int = 1


def _validated_rank(value: object) -> tuple[int, ...]:
    if (
        not isinstance(value, tuple)
        or not value
        or any(type(component) is not int for component in value)
    ):
        raise TypeError("rank must be a non-empty tuple of integers")
    return value


def select_ranked_survivors(
    records: Iterable[T],
    *,
    identity: Callable[[T], Hashable],
    rank: Callable[[T], tuple[int, ...]],
) -> SurvivorshipResult[T]:
    if not callable(identity) or not callable(rank):
        raise TypeError("identity and rank must be callable")

    groups_by_identity: dict[Hashable, _Group[T]] = {}
    ordered_groups: list[_Group[T]] = []

    for input_index, record in enumerate(records):
        record_identity = identity(record)
        try:
            hash(record_identity)
        except TypeError as error:
            raise TypeError("identity must return a hashable value") from error
        record_rank = _validated_rank(rank(record))

        group = groups_by_identity.get(record_identity)
        if group is None:
            group = _Group(
                identity=record_identity,
                record=record,
                selected_index=input_index,
                selected_rank=record_rank,
            )
            groups_by_identity[record_identity] = group
            ordered_groups.append(group)
            continue

        group.size += 1
        if record_rank > group.selected_rank:
            group.record = record
            group.selected_index = input_index
            group.selected_rank = record_rank

    return SurvivorshipResult(
        records=tuple(group.record for group in ordered_groups),
        audit=tuple(
            SurvivorshipAudit(
                identity=group.identity,
                group_size=group.size,
                selected_index=group.selected_index,
                selected_rank=group.selected_rank,
            )
            for group in ordered_groups
        ),
    )
```

## Example

```python

@dataclass(frozen=True, slots=True)
class ExampleRecord:
    key: str
    revision: int
    complete: int
    value: str


records = [
    ExampleRecord("alpha", 1, 3, "old"),
    ExampleRecord("beta", 4, 1, "kept"),
    ExampleRecord("alpha", 2, 1, "winner"),
    ExampleRecord("alpha", 2, 1, "equal-later"),
    ExampleRecord("beta", 3, 9, "lower"),
]
identity_calls: list[str] = []
rank_calls: list[str] = []


def record_identity(record: ExampleRecord) -> str:
    identity_calls.append(record.value)
    return record.key


def record_rank(record: ExampleRecord) -> tuple[int, int]:
    rank_calls.append(record.value)
    return record.revision, record.complete


result = select_ranked_survivors(
    records,
    identity=record_identity,
    rank=record_rank,
)

assert (
    tuple(record.value for record in result.records),
    result.audit,
    identity_calls,
    rank_calls,
    records[2].value,
) == (
    ("winner", "kept"),
    (
        SurvivorshipAudit("alpha", 3, 2, (2, 1)),
        SurvivorshipAudit("beta", 2, 1, (4, 1)),
    ),
    ["old", "kept", "winner", "equal-later", "lower"],
    ["old", "kept", "winner", "equal-later", "lower"],
    "winner",
)
```

## Trade-offs and Limitations

The function retains one record and one audit object per distinct identity, so
it is not a bounded-memory solution for arbitrary streams. Records are returned
by reference and are never copied or mutated. Integer tuples provide a total,
portable ordering but cannot directly express missing or incomparable ranks;
normalize those cases before selection. Equal ranks deliberately favor the
first record, which makes input order part of the contract. Hashability does
not guarantee that a mutable custom key keeps a stable hash or equality result.
Callback errors propagate after any earlier callbacks have run, and the
algorithm performs no fuzzy matching, conflict preservation, persistence, or
transactional publication.

## Related Snippets

<!-- catalog:related:start -->
- [Aggregate Consecutive Values into Weighted Runs](aggregate-consecutive-values-into-weighted-runs.md)
- [Normalize Optional CSV Columns in a Single Pass](normalize-optional-csv-columns-in-a-single-pass.md)
- [Build and Apply a Deterministic Mapping Delta](../storage-databases/build-and-apply-a-deterministic-mapping-delta.md)
<!-- catalog:related:end -->
