---
title: "Find the First Non-Empty Contiguous Integer Span with an Exact Sum"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - find-a-maximum-sum-non-empty-contiguous-integer-subarray-with-explicit-ties.md
  - maintain-bounded-point-adds-and-half-open-range-sums-with-a-fenwick-tree.md
  - build-a-bounded-integer-summed-area-table-for-half-open-rectangle-sums.md
---

# Find the First Non-Empty Contiguous Integer Span with an Exact Sum

## Idea and Problem

Find the first non-empty contiguous integer span whose exact sum equals a target, with one explicit order for resolving multiple matches.

If the prefix sum at `stop` is `current`, a qualifying `start` has prefix sum
`current - target`. Scanning stops from left to right fixes the smallest stop.
Remembering only the earliest index for each prefix then fixes the smallest
start at that stop, even when negative values repeat prefix sums.

## When to Use

Use this algorithm for one bounded in-memory integer sequence when negative
values prevent a sliding window from moving monotonically. It is useful when
the first answer under stop-then-start ordering is enough and exact Python
integer arithmetic is required.

Use a two-pointer window for a non-negative sequence when bounded memory is
more important, or a range-sum data structure when values change or many
independent range queries must be answered.

## Implementation

```python
from dataclasses import dataclass

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_VALUE_COUNT = 65_536


@dataclass(frozen=True, slots=True)
class ExactSumSpan:
    start: int
    stop: int


def find_first_exact_sum_span(
    values: tuple[int, ...],
    target: int,
) -> ExactSumSpan | None:
    """Return the first half-open exact-sum span under stop-then-start ties."""
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    if not 1 <= len(values) <= _MAX_VALUE_COUNT:
        raise ValueError("value count is outside the supported range")
    for index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"values[{index}] must be an exact integer")
        if not _MIN_INT64 <= value <= _MAX_INT64:
            raise ValueError(f"values[{index}] is outside the signed 64-bit range")
    if type(target) is not int:
        raise TypeError("target must be an exact integer")
    if not _MIN_INT64 <= target <= _MAX_INT64:
        raise ValueError("target is outside the signed 64-bit range")

    prefix_sum = 0
    earliest_index_by_prefix = {0: 0}

    for stop, value in enumerate(values, start=1):
        prefix_sum += value
        start = earliest_index_by_prefix.get(prefix_sum - target)
        if start is not None:
            return ExactSumSpan(start=start, stop=stop)
        earliest_index_by_prefix.setdefault(prefix_sum, stop)

    return None
```

## Example

```python
values = (4, -4, 3, 1, -1, 2)

exact_three = find_first_exact_sum_span(values, 3)
exact_zero = find_first_exact_sum_span(values, 0)
missing = find_first_exact_sum_span(values, 9)

assert (exact_three, exact_zero, missing) == (
    ExactSumSpan(0, 3),
    ExactSumSpan(0, 2),
    None,
)
```

## Trade-offs and Limitations

Validation and the prefix scan take expected `O(n)` dictionary work. The map
can retain `O(n)` distinct exact prefix sums, and the returned span uses constant
space. Python integers keep aggregate sums exact beyond the signed-64-bit input
range, but hashing, subtraction, and addition grow with integer bit length.

Looking up a prefix before storing the current one guarantees a non-empty span.
Stops are examined in increasing order, and retaining the earliest index for a
repeated prefix implements the smallest-start secondary tie. The function
returns only one half-open span and does not support approximate sums, circular
ranges, mutation, all-match enumeration, or bounded-memory streaming.

## Related Snippets

<!-- catalog:related:start -->
- [Find a Maximum-Sum Non-Empty Contiguous Integer Subarray with Explicit Ties](find-a-maximum-sum-non-empty-contiguous-integer-subarray-with-explicit-ties.md)
- [Maintain Bounded Point Adds and Half-Open Range Sums with a Fenwick Tree](maintain-bounded-point-adds-and-half-open-range-sums-with-a-fenwick-tree.md)
- [Build a Bounded Integer Summed-Area Table for Half-Open Rectangle Sums](build-a-bounded-integer-summed-area-table-for-half-open-rectangle-sums.md)
<!-- catalog:related:end -->
