---
title: "Find a Maximum-Sum Non-Empty Contiguous Integer Subarray with Explicit Ties"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - maintain-bounded-point-adds-and-half-open-range-sums-with-a-fenwick-tree.md
  - compute-full-window-trailing-maxima-with-a-monotonic-index-deque.md
  - select-a-maximum-cardinality-set-of-non-overlapping-half-open-integer-intervals.md
---

# Find a Maximum-Sum Non-Empty Contiguous Integer Subarray with Explicit Ties

## Idea and Problem

Find one non-empty contiguous range with the greatest sum while making every equal-sum choice deterministic.

For each stop position, keep the best range that ends there. Restart at the
current value only when it is strictly better than extending the previous
range; equality retains the earlier start. Compare global candidates by total,
then by the smallest start and the smallest stop.

## When to Use

Use this algorithm for one bounded in-memory integer sequence when the caller
needs both the maximum sum and the exact half-open range that produced it. The
explicit tie policy is useful when results enter snapshots, tests, or another
deterministic transformation.

Use a range-query data structure when the sequence changes or many independent
ranges must be queried. Choose a different contract if an empty result with sum
zero is allowed, the sequence is circular, or all optimal ranges are required.

## Implementation

```python
from dataclasses import dataclass

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_VALUE_COUNT = 10_000


@dataclass(frozen=True, slots=True)
class MaximumSubarray:
    start: int
    stop: int
    total: int


def maximum_sum_subarray(values: tuple[int, ...]) -> MaximumSubarray:
    """Return the maximum-sum non-empty range under earliest-position ties."""
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    if not 1 <= len(values) <= _MAX_VALUE_COUNT:
        raise ValueError("value count is outside the supported range")

    for value in values:
        if type(value) is not int:
            raise TypeError("values must contain exact integers")
        if not _MIN_INT64 <= value <= _MAX_INT64:
            raise ValueError("values must be in the signed 64-bit range")

    current_start = 0
    current_total = values[0]
    best = MaximumSubarray(start=0, stop=1, total=values[0])

    for index in range(1, len(values)):
        value = values[index]
        extended_total = current_total + value
        if value > extended_total:
            current_start = index
            current_total = value
        else:
            current_total = extended_total

        candidate_stop = index + 1
        if current_total > best.total or (
            current_total == best.total
            and (current_start, candidate_stop) < (best.start, best.stop)
        ):
            best = MaximumSubarray(
                start=current_start,
                stop=candidate_stop,
                total=current_total,
            )

    return best
```

## Example

```python
result = maximum_sum_subarray((-2, 3, -1, 2, -4, 3, 0))
equal_sum_tie = maximum_sum_subarray((1, -1, 1))
all_negative = maximum_sum_subarray((-5, -2, -2))

assert (result, equal_sum_tie, all_negative) == (
    MaximumSubarray(start=1, stop=4, total=4),
    MaximumSubarray(start=0, stop=1, total=1),
    MaximumSubarray(start=1, stop=2, total=-2),
)
```

## Trade-offs and Limitations

Complete validation and the scan each take `O(n)` time. The algorithm uses
`O(1)` auxiliary state apart from the returned frozen object. Input points must
fit signed 64-bit integers, while Python calculates running and returned sums
exactly even when an aggregate leaves that range.

Restarting only on a strictly greater single-value candidate proves the local
tie rule: an equal extension keeps the earlier start. The global comparison
then prefers the earliest start and earliest stop among every maximum-sum
candidate. The function returns one non-empty half-open range; it does not
support an empty answer, circular ranges, streaming input, mutable updates,
parallel execution, or enumeration of every optimum.

## Related Snippets

<!-- catalog:related:start -->
- [Maintain Bounded Point Adds and Half-Open Range Sums with a Fenwick Tree](maintain-bounded-point-adds-and-half-open-range-sums-with-a-fenwick-tree.md)
- [Compute Full-Window Trailing Maxima with a Monotonic Index Deque](compute-full-window-trailing-maxima-with-a-monotonic-index-deque.md)
- [Select a Maximum-Cardinality Set of Non-Overlapping Half-Open Integer Intervals](select-a-maximum-cardinality-set-of-non-overlapping-half-open-integer-intervals.md)
<!-- catalog:related:end -->
