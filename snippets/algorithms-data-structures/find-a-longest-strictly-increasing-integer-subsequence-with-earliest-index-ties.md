---
title: "Find a Longest Strictly Increasing Integer Subsequence with Earliest-Index Ties"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - find-a-maximum-sum-non-empty-contiguous-integer-subarray-with-explicit-ties.md
  - select-a-maximum-cardinality-set-of-non-overlapping-half-open-integer-intervals.md
  - rank-bounded-records-with-stable-ties-and-neighbor-windows.md
---

# Find a Longest Strictly Increasing Integer Subsequence with Earliest-Index Ties

## Idea and Problem

Find one longest strictly increasing subsequence while resolving every equal-length choice by the earliest complete tuple of original indexes.

Compute the longest possible suffix that starts at every input position. Then
scan forward for the earliest position that is greater than the previously
selected value and still has enough suffix length to complete an optimum. This
makes the index tie policy part of the algorithm rather than an incidental
property of loop or replacement order.

## When to Use

Use this algorithm for one bounded in-memory integer sequence when both the
subsequence and its original positions matter. It is useful for deterministic
analysis, fixtures, and transformations where several longest subsequences may
contain different values or duplicate input values.

Use a specialized `O(n log n)` implementation when the input is large and its
tie behavior has been specified and tested separately. Choose a contiguous
range algorithm when skipped positions are not allowed, and choose a
non-decreasing variant when equal adjacent values should be admissible.

## Implementation

```python
from dataclasses import dataclass

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_VALUE_COUNT = 2_048


@dataclass(frozen=True, slots=True)
class IncreasingSubsequence:
    indexes: tuple[int, ...]
    values: tuple[int, ...]


def longest_strictly_increasing_subsequence(
    values: tuple[int, ...],
) -> IncreasingSubsequence:
    """Return an earliest-index longest strictly increasing subsequence."""
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    if not 1 <= len(values) <= _MAX_VALUE_COUNT:
        raise ValueError("value count is outside the supported range")

    for value in values:
        if type(value) is not int:
            raise TypeError("values must contain exact integers")
        if not _MIN_INT64 <= value <= _MAX_INT64:
            raise ValueError("values must be in the signed 64-bit range")

    suffix_lengths = [1] * len(values)
    for index in range(len(values) - 1, -1, -1):
        current_value = values[index]
        best_length = 1
        for following_index in range(index + 1, len(values)):
            if values[following_index] > current_value:
                best_length = max(
                    best_length,
                    1 + suffix_lengths[following_index],
                )
        suffix_lengths[index] = best_length

    remaining = max(suffix_lengths)
    selected_indexes: list[int] = []
    search_start = 0

    while remaining:
        for index in range(search_start, len(values)):
            if selected_indexes and values[index] <= values[selected_indexes[-1]]:
                continue
            if suffix_lengths[index] < remaining:
                continue
            selected_indexes.append(index)
            search_start = index + 1
            remaining -= 1
            break
        else:
            raise AssertionError("suffix lengths cannot complete reconstruction")

    indexes = tuple(selected_indexes)
    return IncreasingSubsequence(
        indexes=indexes,
        values=tuple(values[index] for index in indexes),
    )
```

## Example

```python
result = longest_strictly_increasing_subsequence((3, 1, 2, 2, 4, 3, 5, 3, 5))
duplicate_heavy = longest_strictly_increasing_subsequence((2, 2, 3, 3, 4, 4))
all_equal = longest_strictly_increasing_subsequence((7, 7, 7))

try:
    longest_strictly_increasing_subsequence((1, True, 2))
except TypeError:
    bool_rejected = True
else:
    bool_rejected = False

assert (result, duplicate_heavy, all_equal, bool_rejected) == (
    IncreasingSubsequence(indexes=(1, 2, 4, 6), values=(1, 2, 4, 5)),
    IncreasingSubsequence(indexes=(0, 2, 4), values=(2, 3, 4)),
    IncreasingSubsequence(indexes=(0,), values=(7,)),
    True,
)
```

## Trade-offs and Limitations

Validation takes `O(n)` time. Computing suffix lengths takes `O(n^2)` time,
reconstruction scans the input once, and the working state plus returned
tuples use `O(n)` memory. The fixed input limit bounds the intentionally simple
quadratic algorithm.

The suffix length at a candidate proves that an optimum can still be
completed. Choosing the first feasible candidate at every step therefore
minimizes the first differing original index and produces the lexicographically
smallest complete index tuple. This policy is not a value-tuple tie: in the
example it chooses the earlier value `4` before a later value `3`.

The function accepts only a non-empty fixed tuple of signed 64-bit integers and
returns one strict, non-contiguous subsequence. It does not enumerate other
optima, admit equal consecutive values, process a stream, support updates, or
claim the asymptotic performance of an `O(n log n)` variant.

## Related Snippets

<!-- catalog:related:start -->
- [Find a Maximum-Sum Non-Empty Contiguous Integer Subarray with Explicit Ties](find-a-maximum-sum-non-empty-contiguous-integer-subarray-with-explicit-ties.md)
- [Select a Maximum-Cardinality Set of Non-Overlapping Half-Open Integer Intervals](select-a-maximum-cardinality-set-of-non-overlapping-half-open-integer-intervals.md)
- [Rank Bounded Records with Stable Ties and Neighbor Windows](rank-bounded-records-with-stable-ties-and-neighbor-windows.md)
<!-- catalog:related:end -->
