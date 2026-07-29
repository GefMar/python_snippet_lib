---
title: "Find the Leftmost Shortest Non-Empty Contiguous Integer Span Reaching a Sum Threshold"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - find-the-first-non-empty-contiguous-integer-span-with-an-exact-sum.md
  - find-a-maximum-sum-non-empty-contiguous-integer-subarray-with-explicit-ties.md
  - compute-full-window-trailing-maxima-with-a-monotonic-index-deque.md
---

# Find the Leftmost Shortest Non-Empty Contiguous Integer Span Reaching a Sum Threshold

## Idea and Problem

Find a shortest non-empty contiguous integer span whose exact sum reaches a threshold, choosing the leftmost span when lengths tie.

For each right-side prefix sum, an earlier prefix qualifies when their
difference reaches the threshold. Candidate prefix sums are kept in increasing
order. Qualifying indexes leave from the front, while a new prefix removes any
larger or equal prefix behind it because the newer index will produce a span
that is no longer and has at least as large a future sum.

## When to Use

Use this algorithm for one bounded in-memory integer sequence when values may
be negative and the shortest threshold-reaching interval is required. It is
useful when an ordinary two-pointer window is invalid because adding a value
can lower the running sum.

Use a simpler sliding window when every value is non-negative. Use a range-sum
index when the sequence changes or many unrelated interval queries must be
answered, and use an exact-sum lookup when exceeding the target is not allowed.

## Implementation

```python
from collections import deque

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_VALUE_COUNT = 100_000


def find_leftmost_shortest_threshold_span(
    values: tuple[int, ...],
    threshold: int,
) -> tuple[int, int, int] | None:
    """Return start, stop, and sum for the leftmost shortest qualifying span."""
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    if type(threshold) is not int:
        raise TypeError("threshold must be an exact non-boolean integer")
    if len(values) > _MAX_VALUE_COUNT:
        raise ValueError("value count exceeds the supported limit")

    for index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"values[{index}] must be an exact non-boolean integer")
    for index, value in enumerate(values):
        if not _MIN_INT64 <= value <= _MAX_INT64:
            raise ValueError(f"values[{index}] is outside the signed 64-bit range")
    if not _MIN_INT64 <= threshold <= _MAX_INT64:
        raise ValueError("threshold is outside the signed 64-bit range")

    prefix_sums = [0]
    candidate_indexes: deque[int] = deque([0])
    running_sum = 0
    best_key: tuple[int, int] | None = None
    best_span: tuple[int, int, int] | None = None

    for stop, value in enumerate(values, start=1):
        running_sum += value
        prefix_sums.append(running_sum)

        while candidate_indexes and running_sum - prefix_sums[candidate_indexes[0]] >= threshold:
            start = candidate_indexes.popleft()
            key = (stop - start, start)
            if best_key is None or key < best_key:
                best_key = key
                best_span = (
                    start,
                    stop,
                    running_sum - prefix_sums[start],
                )

        while candidate_indexes and running_sum <= prefix_sums[candidate_indexes[-1]]:
            candidate_indexes.pop()
        candidate_indexes.append(stop)

    return best_span
```

## Example

```python
def enumerate_threshold_spans(
    values: tuple[int, ...],
    threshold: int,
) -> tuple[int, int, int] | None:
    candidates: list[tuple[int, int, int, int]] = []
    for start in range(len(values)):
        total = 0
        for stop in range(start + 1, len(values) + 1):
            total += values[stop - 1]
            if total >= threshold:
                candidates.append((stop - start, start, stop, total))
    if not candidates:
        return None
    _, start, stop, total = min(candidates, key=lambda item: item[:2])
    return start, stop, total


cases = (
    ((2, -1, 2, 1, -4, 3), 4),
    ((2, 2, -10, 2, 2), 4),
    ((-5, -2, -8), -3),
    ((-1, 0, -2), 0),
    ((1, -1), 5),
    ((), 0),
)
results = tuple(
    find_leftmost_shortest_threshold_span(values, threshold) for values, threshold in cases
)
for values, threshold in cases:
    assert find_leftmost_shortest_threshold_span(
        values,
        threshold,
    ) == enumerate_threshold_spans(values, threshold)

try:
    find_leftmost_shortest_threshold_span((1, True), 1)
except TypeError:
    boolean_rejected = True
else:
    boolean_rejected = False

assert (results, boolean_rejected) == (
    (
        (0, 4, 4),
        (0, 2, 4),
        (1, 2, -2),
        (1, 2, 0),
        None,
        None,
    ),
    True,
)
```

## Trade-offs and Limitations

Validation and the deque scan take `O(n)` time. Every prefix index enters and
leaves the deque at most once. The prefix list, deque, and returned result use
`O(n)` auxiliary memory. Exact addition, subtraction, and comparison are not
unit-cost operations for arbitrary-precision integers, but the declared input
limits keep every prefix sum and returned total to roughly 80 bits.

The result is a non-empty half-open span. It first minimizes length and then
start index; once those are fixed, the stop and total are fixed as well. The
function returns one witness and does not support circular spans, exact-sum-only
matching, maximum-sum selection, all-witness enumeration, mutable updates, or
bounded-memory streaming.

## Related Snippets

<!-- catalog:related:start -->
- [Find the First Non-Empty Contiguous Integer Span with an Exact Sum](find-the-first-non-empty-contiguous-integer-span-with-an-exact-sum.md)
- [Find a Maximum-Sum Non-Empty Contiguous Integer Subarray with Explicit Ties](find-a-maximum-sum-non-empty-contiguous-integer-subarray-with-explicit-ties.md)
- [Compute Full-Window Trailing Maxima with a Monotonic Index Deque](compute-full-window-trailing-maxima-with-a-monotonic-index-deque.md)
<!-- catalog:related:end -->
