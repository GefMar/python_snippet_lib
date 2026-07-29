---
title: "Select the Stable K Smallest Bounded Records with a Max-Heap"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - rank-bounded-records-with-stable-ties-and-neighbor-windows.md
  - find-exact-heavy-hitters-with-verified-misra-gries-candidates.md
  - ../data-processing/sample-a-weighted-stream-with-a-fixed-size-reservoir.md
---

# Select the Stable K Smallest Bounded Records with a Max-Heap

## Idea and Problem

Retain the stable k smallest records from a bounded input while storing only the current winners in a max-heap.

A max-heap keeps the worst retained key at its root. Each new record can then
be discarded or replace that root without sorting the full input. Pairing the
score with the original index makes equal scores stable: an earlier record is
always better than a later one, even when identifiers are duplicated.

## When to Use

Use this algorithm when the complete bounded input is available for validation
but only a small exact prefix by score should be retained. It is useful for
candidate shortlists, lowest-cost records, and deterministic diagnostic
samples where `k` is much smaller than the number of records.

The score must already be an exact signed 64-bit integer. Use a full stable
sort when every record is needed in order, or a selection implementation with
a caller-defined key when coercion, floating-point ordering, or multiple sort
fields are part of the contract.

## Implementation

```python
import heapq
from dataclasses import dataclass

_MAX_RECORDS = 65_536
_MAX_K = 1_024
_MAX_IDENTIFIER_BYTES = 64
_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1


@dataclass(frozen=True, slots=True)
class ScoredRecord:
    identifier: str
    score: int


@dataclass(frozen=True, slots=True)
class SelectedRecord:
    input_index: int
    record: ScoredRecord


def _validate_record(record: ScoredRecord) -> None:
    if type(record) is not ScoredRecord:
        raise TypeError("records must contain exact ScoredRecord values")
    if type(record.identifier) is not str:
        raise TypeError("record identifiers must be exact strings")
    if not record.identifier:
        raise ValueError("record identifiers must not be empty")
    if len(record.identifier) > _MAX_IDENTIFIER_BYTES:
        raise ValueError("record identifier exceeds the supported byte count")
    try:
        encoded_identifier = record.identifier.encode("utf-8")
    except UnicodeEncodeError as error:
        raise ValueError("record identifiers must not contain surrogates") from error
    if len(encoded_identifier) > _MAX_IDENTIFIER_BYTES:
        raise ValueError("record identifier exceeds the supported byte count")
    if type(record.score) is not int:
        raise TypeError("record scores must be exact integers")
    if not _MIN_INT64 <= record.score <= _MAX_INT64:
        raise ValueError("record score is outside the signed 64-bit range")


def select_stable_k_smallest(
    records: tuple[ScoredRecord, ...],
    k: int,
) -> tuple[SelectedRecord, ...]:
    if type(records) is not tuple:
        raise TypeError("records must be an exact tuple")
    if len(records) > _MAX_RECORDS:
        raise ValueError("records exceed the supported count")
    if type(k) is not int:
        raise TypeError("k must be an exact integer")
    if not 0 <= k <= _MAX_K:
        raise ValueError("k is outside the supported range")

    heap: list[tuple[int, int, ScoredRecord]] = []
    for input_index, record in enumerate(records):
        _validate_record(record)
        if k == 0:
            continue

        entry = (record.score, input_index, record)
        if len(heap) < k:
            heap.append(entry)
            if len(heap) == k:
                heapq.heapify_max(heap)
            continue

        if entry[:2] < heap[0][:2]:
            heapq.heapreplace_max(heap, entry)

    heap.sort(key=lambda entry: entry[:2])
    return tuple(
        SelectedRecord(input_index=input_index, record=record)
        for _, input_index, record in heap
    )
```

## Example

```python
records = (
    ScoredRecord("beta", 5),
    ScoredRecord("same", 2),
    ScoredRecord("same", 2),
    ScoredRecord("urgent", 1),
    ScoredRecord("later", 2),
)

selected = select_stable_k_smallest(records, 3)

assert tuple(
    (item.input_index, item.record.identifier, item.record.score)
    for item in selected
) == (
    (3, "urgent", 1),
    (1, "same", 2),
    (2, "same", 2),
)
assert select_stable_k_smallest(records, 0) == ()
assert len(select_stable_k_smallest(records, 100)) == len(records)

from itertools import product

for count in range(5):
    for scores in product((-1, 0, 1), repeat=count):
        candidates = tuple(
            ScoredRecord(f"item-{index % 2}", score)
            for index, score in enumerate(scores)
        )
        for retained_count in range(count + 2):
            actual = select_stable_k_smallest(candidates, retained_count)
            expected = sorted(
                enumerate(candidates),
                key=lambda pair: (pair[1].score, pair[0]),
            )[:retained_count]
            assert [
                (item.input_index, item.record) for item in actual
            ] == expected

try:
    select_stable_k_smallest((ScoredRecord("valid", 1), object()), 0)
except TypeError:
    pass
else:
    raise AssertionError("k=0 must still validate every record")

assert select_stable_k_smallest((), 0) == ()
```

## Trade-offs and Limitations

Let `n` be the input count and `m = min(k, n)`. Full validation, heap
maintenance, and final ordering use
`O(n + n log(m + 1) + m log(m + 1))` time and `O(m)` additional state. The
`+1` keeps the expression meaningful when `k` is zero; every input record is
still validated in that case.

This approach is exact but not online in its public contract because the input
is a tuple and the returned winners are sorted after selection. The record
objects remain shared immutable values rather than copied payloads. The helper
does not enforce unique identifiers, change scores, remove retained entries,
sample randomly, or select the largest `k`.

Python's max-heap operations used here require Python 3.14 or newer. For a
single small list, `sorted(records, key=...)[:k]` may be simpler. The heap is
most useful when `m` is materially smaller than `n` and bounded additional
state is important.

## Related Snippets

<!-- catalog:related:start -->
- [Rank Bounded Records with Stable Ties and Neighbor Windows](rank-bounded-records-with-stable-ties-and-neighbor-windows.md)
- [Find Exact Heavy Hitters with Verified Misra-Gries Candidates](find-exact-heavy-hitters-with-verified-misra-gries-candidates.md)
- [Sample a Weighted Stream with a Fixed-Size Reservoir](../data-processing/sample-a-weighted-stream-with-a-fixed-size-reservoir.md)
<!-- catalog:related:end -->
