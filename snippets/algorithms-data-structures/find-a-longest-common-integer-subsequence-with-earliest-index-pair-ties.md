---
title: "Find a Longest Common Integer Subsequence with Earliest Index-Pair Ties"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - find-a-longest-strictly-increasing-integer-subsequence-with-earliest-index-ties.md
  - rank-hierarchy-paths-with-bounded-weighted-edit-distance.md
  - merge-bounded-sorted-integer-runs-with-observable-source-order-ties.md
---

# Find a Longest Common Integer Subsequence with Earliest Index-Pair Ties

## Idea and Problem

Find one longest subsequence shared by two integer tuples while making every positional tie explicit and reproducible.

A suffix dynamic-programming table records the best remaining length from each
pair of positions. Reconstruction scans feasible `(left_index, right_index)`
pairs in lexicographic order and takes the first pair that can still complete
an optimum. The result therefore preserves both matched values and their exact
positions in the two inputs.

## When to Use

Use this algorithm for two bounded, in-memory integer sequences when skipped
items are allowed and the exact alignment matters. It is useful for comparing
ordered identifiers, validating deterministic transformations, and building
small alignment fixtures where repeated values can produce several equally
long answers.

Use substring search when matches must be contiguous. Choose an edit-distance
or sequence-alignment algorithm when insertions, deletions, substitutions, or
gaps need scores. For substantially larger inputs, prefer a specialized
implementation with a deliberately specified reconstruction policy.

## Implementation

```python
from dataclasses import dataclass

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_VALUE_COUNT = 512


@dataclass(frozen=True, slots=True)
class CommonIntegerSubsequence:
    values: tuple[int, ...]
    index_pairs: tuple[tuple[int, int], ...]


def _validated_integer_tuple(values: object, *, field: str) -> tuple[int, ...]:
    if type(values) is not tuple:
        raise TypeError(f"{field} must be an exact tuple")
    if len(values) > _MAX_VALUE_COUNT:
        raise ValueError(f"{field} exceeds the supported value count")
    for index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"{field}[{index}] must be an exact integer")
        if not _MIN_INT64 <= value <= _MAX_INT64:
            raise ValueError(f"{field}[{index}] is outside the signed 64-bit range")
    return values


def longest_common_integer_subsequence(
    left: tuple[int, ...],
    right: tuple[int, ...],
) -> CommonIntegerSubsequence:
    """Return an index-pair-canonical longest common subsequence."""
    checked_left = _validated_integer_tuple(left, field="left")
    checked_right = _validated_integer_tuple(right, field="right")

    left_count = len(checked_left)
    right_count = len(checked_right)
    lengths = [[0] * (right_count + 1) for _ in range(left_count + 1)]

    for left_index in range(left_count - 1, -1, -1):
        current_row = lengths[left_index]
        following_row = lengths[left_index + 1]
        for right_index in range(right_count - 1, -1, -1):
            if checked_left[left_index] == checked_right[right_index]:
                current_row[right_index] = 1 + following_row[right_index + 1]
            else:
                current_row[right_index] = max(
                    following_row[right_index],
                    current_row[right_index + 1],
                )

    remaining = lengths[0][0]
    left_start = 0
    right_start = 0
    selected_pairs: list[tuple[int, int]] = []

    while remaining:
        selected: tuple[int, int] | None = None
        for left_index in range(left_start, left_count):
            for right_index in range(right_start, right_count):
                if (
                    checked_left[left_index] == checked_right[right_index]
                    and 1 + lengths[left_index + 1][right_index + 1] == remaining
                ):
                    selected = (left_index, right_index)
                    break
            if selected is not None:
                break

        if selected is None:
            raise AssertionError("the suffix table must permit reconstruction")
        selected_pairs.append(selected)
        left_start, right_start = selected[0] + 1, selected[1] + 1
        remaining -= 1

    index_pairs = tuple(selected_pairs)
    return CommonIntegerSubsequence(
        values=tuple(checked_left[left_index] for left_index, _ in index_pairs),
        index_pairs=index_pairs,
    )
```

## Example

```python
result = longest_common_integer_subsequence(
    (2, 1, 2, 1),
    (1, 2, 1, 2),
)
duplicate_tie = longest_common_integer_subsequence((7, 7), (7,))
empty = longest_common_integer_subsequence((), (1, 2, 3))

try:
    longest_common_integer_subsequence((1, True), (1,))
except TypeError:
    bool_rejected = True
else:
    bool_rejected = False

assert (result, duplicate_tie, empty, bool_rejected) == (
    CommonIntegerSubsequence(
        values=(2, 1, 2),
        index_pairs=((0, 1), (1, 2), (2, 3)),
    ),
    CommonIntegerSubsequence(values=(7,), index_pairs=((0, 0),)),
    CommonIntegerSubsequence(values=(), index_pairs=()),
    True,
)
```

## Trade-offs and Limitations

Validation takes `O(m + n)` time. The suffix table takes `O(mn)` time and
memory. Reconstruction also stays within `O(mn)` time: after choosing a pair,
both search starts move beyond it, so no inspected position pair is revisited.
The fixed 512-value limits bound the intentionally direct table.

The feasible-pair check proves that the remaining optimum can follow a
candidate. Taking the first feasible pair at every step therefore returns the
lexicographically smallest complete tuple of `(left_index, right_index)` pairs,
not the lexicographically smallest value tuple. Values and indexes are returned
as immutable tuples.

Inputs may be empty and contain duplicate values, but every value must be an
exact signed 64-bit integer. The function returns a subsequence rather than a
contiguous substring and does not produce an edit script, score gaps, accept
generic mutable values, stream inputs, use reduced-memory Hirschberg
reconstruction, or enumerate every optimum.

## Related Snippets

<!-- catalog:related:start -->
- [Find a Longest Strictly Increasing Integer Subsequence with Earliest-Index Ties](find-a-longest-strictly-increasing-integer-subsequence-with-earliest-index-ties.md)
- [Rank Hierarchy Paths with Bounded Weighted Edit Distance](rank-hierarchy-paths-with-bounded-weighted-edit-distance.md)
- [Merge Bounded Sorted Integer Runs with Observable Source-Order Ties](merge-bounded-sorted-integer-runs-with-observable-source-order-ties.md)
<!-- catalog:related:end -->
