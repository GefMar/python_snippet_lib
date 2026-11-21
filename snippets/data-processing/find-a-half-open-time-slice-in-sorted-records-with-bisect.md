---
title: "Find a Half-Open Time Slice in Sorted Records with bisect"
snippet_type: recipe
use_cases:
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../algorithms-data-structures/find-the-matching-range-for-a-literal-prefix-in-sorted-strings.md
  - join-two-strictly-increasing-streams-by-exact-timestamp.md
---

# Find a Half-Open Time Slice in Sorted Records with bisect

## Idea and Problem

Find the original record slice for a half-open time interval without building a separate time array.

Apply a key function to the sorted records and use the left insertion point at both ends. All records at the start are included, and all records at the end are left out. The threshold arguments are already times, not records.

## When to Use

Use this over a stable random-access sequence ordered by its integer time field. The slice describes a time interval including the start and excluding the end, with duplicates and record order preserved.

## Implementation

```python
from bisect import bisect_left
from collections.abc import Sequence
from operator import itemgetter


def time_slice(records: Sequence[tuple[int, str]], start: int, end: int) -> slice:
    if end < start:
        raise ValueError("the end must not precede the start")

    time = itemgetter(0)
    first = bisect_left(records, start, key=time)
    stop = bisect_left(records, end, lo=first, key=time)
    return slice(first, stop)
```

## Example

```python
rows = [(1, "a"), (5, "b"), (5, "c"), (8, "d"), (8, "e"), (20, "f")]
selection = time_slice(rows, 5, 8)
assert (selection, rows[selection], rows[time_slice(rows, 5, 5)]) == (
    slice(1, 3),
    [(5, "b"), (5, "c")],
    [],
)
```

## Trade-offs and Limitations

This assumes the data is already sorted and unchanged. Checking sorted order inside each call would require scanning the sequence. Returning the bounds does not copy records; slicing a list later allocates the selected items. For an expensive key function with many repeated queries, consider a cached key array.

## Related Snippets

<!-- catalog:related:start -->
- [Find the Matching Range for a Literal Prefix in Sorted Strings](../algorithms-data-structures/find-the-matching-range-for-a-literal-prefix-in-sorted-strings.md)
- [Join Two Strictly Increasing Streams by Exact Timestamp](join-two-strictly-increasing-streams-by-exact-timestamp.md)
<!-- catalog:related:end -->
