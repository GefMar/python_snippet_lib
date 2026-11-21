---
title: "Stream Count Deltas Between Two Sorted Inventories"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - co-group-bounded-sorted-integer-keys-with-full-outer-coverage.md
  - merge-join-bounded-sorted-records-with-duplicate-key-cartesian-matches.md
---

# Stream Count Deltas Between Two Sorted Inventories

## Idea and Problem

Compare two sorted sequences of item codes and emit only the keys whose occurrence counts changed.

Group runs of equal keys into counts. Merge the two count streams in key
order, treating an absent key as zero. One key can represent many occurrences
without storing its records or building a dictionary of every distinct key.

## When to Use

Use this when two inventory exports are already sorted by the same Python
string ordering and each occurrence represents one unit. Read the results
lazily when you want changed counts for a large input. For small unsorted
inputs, two `Counter` objects are simpler.

## Implementation

```python
from collections.abc import Iterable, Iterator
from dataclasses import dataclass
from heapq import merge
from itertools import groupby
from operator import itemgetter


@dataclass(frozen=True, slots=True)
class CountChange:
    key: str
    before: int
    after: int


def _run_counts(keys: Iterable[str]) -> Iterator[tuple[str, int]]:
    previous = None
    for key, run in groupby(keys):
        if previous is not None and key < previous:
            raise ValueError("inventory keys must be sorted")
        yield key, sum(1 for _ in run)
        previous = key


def changed_counts(
    before: Iterable[str],
    after: Iterable[str],
) -> Iterator[CountChange]:
    old_counts = ((key, count, 0) for key, count in _run_counts(before))
    new_counts = ((key, 0, count) for key, count in _run_counts(after))
    by_key = itemgetter(0)
    for key, contributions in groupby(merge(old_counts, new_counts, key=by_key), by_key):
        old_total = new_total = 0
        for _, old_count, new_count in contributions:
            old_total += old_count
            new_total += new_count
        if old_total != new_total:
            yield CountChange(key, old_total, new_total)
```

## Example

```python
before = iter(("cable", "cable", "case", "dock"))
after = iter(("cable", "case", "dock", "dock", "stand"))
assert tuple(changed_counts(before, after)) == (
    CountChange("cable", 2, 1),
    CountChange("dock", 1, 2),
    CountChange("stand", 0, 1),
)
```

## Trade-offs and Limitations

The merge consumes both inputs once and keeps a fixed number of keys and
counters in addition to the inputs and whatever the caller retains. Long
equal-key runs must be consumed before their counts become available. The
caller owns the inputs and should arrange for file handles to close if it
stops reading early.

Order errors are detected during iteration. Earlier changes may already have
been yielded when a later error surfaces. Validate or stage the entire result
first if changes must be applied atomically. The recipe uses exact strings as
keys, so normalize keys upstream when case or whitespace should be ignored.

## Related Snippets

<!-- catalog:related:start -->
- [Co-Group Bounded Sorted Integer Keys with Full-Outer Coverage](co-group-bounded-sorted-integer-keys-with-full-outer-coverage.md)
- [Merge-Join Bounded Sorted Records with Duplicate-Key Cartesian Matches](merge-join-bounded-sorted-records-with-duplicate-key-cartesian-matches.md)
<!-- catalog:related:end -->
