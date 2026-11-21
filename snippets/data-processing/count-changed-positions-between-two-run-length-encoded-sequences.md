---
title: "Count Changed Positions Between Two Run-Length Encoded Sequences"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - aggregate-consecutive-values-into-weighted-runs.md
  - join-non-overlapping-half-open-validity-intervals-by-exact-overlap.md
---

# Count Changed Positions Between Two Run-Length Encoded Sequences

## Idea and Problem

Count where two aligned sequences differ directly from their run lengths, even when they change values at different positions.

Keep the value and remaining length of one run from each input. The smaller
remaining length gives a span with a constant pair of values. Add that span's
length to the result if the values differ. Advance whichever runs end at
the boundary and continue with the rest.

## When to Use

Use this for aligned label snapshots or status histories that already encode
repeated consecutive values as `(label, length)` pairs. Labels are strings
and lengths are positive integers. The two decoded sequences must have equal
length. Adjacent input runs may repeat the same label.

## Implementation

```python
from collections.abc import Iterable, Iterator


def _checked_runs(runs: Iterable[tuple[str, int]]) -> Iterator[tuple[str, int]]:
    for label, length in runs:
        if type(label) is not str or type(length) is not int:
            raise TypeError("each run needs a string label and an integer length")
        if length <= 0:
            raise ValueError("run lengths must be positive")
        yield label, length


def count_changed_positions(
    before: Iterable[tuple[str, int]], after: Iterable[tuple[str, int]]
) -> int:
    left, right = _checked_runs(before), _checked_runs(after)
    current_left, current_right = next(left, None), next(right, None)
    changed = 0

    while current_left is not None and current_right is not None:
        left_label, left_length = current_left
        right_label, right_length = current_right
        width = min(left_length, right_length)
        if left_label != right_label:
            changed += width

        current_left = (
            next(left, None) if left_length == width else (left_label, left_length - width)
        )
        current_right = (
            next(right, None) if right_length == width else (right_label, right_length - width)
        )

    if current_left is not None or current_right is not None:
        raise ValueError("decoded sequences must have equal length")
    return changed
```

## Example

```python
before = (("open", 4), ("held", 3), ("closed", 2))
after = (("open", 3), ("held", 2), ("open", 2), ("closed", 2))

assert (
    count_changed_positions(iter(before), iter(after)),
    count_changed_positions((("open", 10**12),), (("held", 10**12),)),
    count_changed_positions((), ()),
) == (3, 10**12, 0)
```

## Trade-offs and Limitations

At least one run finishes on every iteration, so the loop count grows with
the number of encoded runs. Only the current pair and running total are
retained. The function consumes both inputs to return a complete count. Use
reusable sequences when the caller also needs to inspect the runs afterwards.

This is a position-by-position difference count. It cannot realign the inputs
after insertions or deletions. Use an edit-distance algorithm when an offset
shift should cost less than mismatching each later position.

## Related Snippets

<!-- catalog:related:start -->
- [Aggregate Consecutive Values into Weighted Runs](aggregate-consecutive-values-into-weighted-runs.md)
- [Join Non-Overlapping Half-Open Validity Intervals by Exact Overlap](join-non-overlapping-half-open-validity-intervals-by-exact-overlap.md)
<!-- catalog:related:end -->
