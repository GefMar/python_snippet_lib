---
title: "Align Bounded Integer Sequences with Exact Dynamic Time Warping"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - build-a-canonical-minimum-unit-cost-edit-script-between-bounded-unicode-texts.md
  - rank-hierarchy-paths-with-bounded-weighted-edit-distance.md
  - ../data-processing/match-increasing-timestamps-to-the-nearest-right-timestamp-within-a-tolerance.md
---

# Align Bounded Integer Sequences with Exact Dynamic Time Warping

## Idea and Problem

Align two non-empty integer sequences while allowing either sequence to repeat a position along the path.

Every visited coordinate `(i, j)` contributes `abs(left[i] - right[j])` to the
cost. A path begins at `(0, 0)`, ends at both final indexes, and moves only
diagonally, by advancing the first sequence, or by advancing the second
sequence. A suffix dynamic-programming table records the minimum cost from
each coordinate through the final coordinate.

Several paths can have the same minimum cost. Reconstruction makes the result
repeatable by choosing diagonal first, then advance-first, then
advance-second whenever multiple optimal suffix continuations are available.

## When to Use

Use this algorithm when two complete, bounded integer sequences may progress
at different rates and callers need both the exact accumulated cost and every
coordinate in one deterministic alignment. The returned path is useful for
auditing which values were paired or for building small reference fixtures.

The inputs and the quadratic table must fit in memory. Choose a domain-specific
distance or alignment model when absolute scalar difference and the documented
move set do not represent the relationship between the sequences.

## Implementation

```python
from dataclasses import dataclass

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_SEQUENCE_LENGTH = 512
_MAX_DTW_CELLS = 262_144


@dataclass(frozen=True, slots=True)
class DynamicTimeWarpingAlignment:
    cost: int
    path: tuple[tuple[int, int], ...]


def _validate_integer_sequence(
    value: object,
    *,
    field: str,
) -> tuple[int, ...]:
    if type(value) is not tuple:
        raise TypeError(f"{field} must be an exact tuple")
    if not 1 <= len(value) <= _MAX_SEQUENCE_LENGTH:
        raise ValueError(f"{field} length is outside the supported range")
    for index, item in enumerate(value):
        if type(item) is not int:
            raise TypeError(f"{field}[{index}] must be an exact integer")
        if not _MIN_INT64 <= item <= _MAX_INT64:
            raise ValueError(f"{field}[{index}] is outside the signed 64-bit range")
    return value


def align_integer_sequences_with_dynamic_time_warping(
    left: tuple[int, ...],
    right: tuple[int, ...],
) -> DynamicTimeWarpingAlignment:
    """Return the minimum-cost alignment under the fixed transition priority."""
    checked_left = _validate_integer_sequence(left, field="left")
    checked_right = _validate_integer_sequence(right, field="right")

    left_count = len(checked_left)
    right_count = len(checked_right)
    if left_count * right_count > _MAX_DTW_CELLS:
        raise ValueError("dynamic-time-warping table exceeds the supported size")

    suffix_cost = [[0] * right_count for _ in range(left_count)]
    for left_index in range(left_count - 1, -1, -1):
        for right_index in range(right_count - 1, -1, -1):
            local_cost = abs(checked_left[left_index] - checked_right[right_index])
            if left_index == left_count - 1 and right_index == right_count - 1:
                suffix_cost[left_index][right_index] = local_cost
                continue

            successor_costs: list[int] = []
            if left_index + 1 < left_count and right_index + 1 < right_count:
                successor_costs.append(suffix_cost[left_index + 1][right_index + 1])
            if left_index + 1 < left_count:
                successor_costs.append(suffix_cost[left_index + 1][right_index])
            if right_index + 1 < right_count:
                successor_costs.append(suffix_cost[left_index][right_index + 1])
            suffix_cost[left_index][right_index] = local_cost + min(successor_costs)

    path: list[tuple[int, int]] = [(0, 0)]
    left_index = 0
    right_index = 0
    while left_index != left_count - 1 or right_index != right_count - 1:
        local_cost = abs(checked_left[left_index] - checked_right[right_index])
        remaining_cost = suffix_cost[left_index][right_index] - local_cost

        if (
            left_index + 1 < left_count
            and right_index + 1 < right_count
            and suffix_cost[left_index + 1][right_index + 1] == remaining_cost
        ):
            left_index += 1
            right_index += 1
        elif (
            left_index + 1 < left_count
            and suffix_cost[left_index + 1][right_index] == remaining_cost
        ):
            left_index += 1
        elif (
            right_index + 1 < right_count
            and suffix_cost[left_index][right_index + 1] == remaining_cost
        ):
            right_index += 1
        else:
            raise AssertionError("alignment reconstruction found no optimal successor")
        path.append((left_index, right_index))

    return DynamicTimeWarpingAlignment(suffix_cost[0][0], tuple(path))


```

## Example

```python
def recompute_alignment_cost(
    left: tuple[int, ...],
    right: tuple[int, ...],
    alignment: DynamicTimeWarpingAlignment,
) -> int:
    assert alignment.path[0] == (0, 0)
    assert alignment.path[-1] == (len(left) - 1, len(right) - 1)
    assert all(
        (next_left - left_index, next_right - right_index) in ((1, 1), (1, 0), (0, 1))
        for (left_index, right_index), (next_left, next_right) in zip(
            alignment.path,
            alignment.path[1:],
            strict=False,
        )
    )
    return sum(abs(left[i] - right[j]) for i, j in alignment.path)


expanded = align_integer_sequences_with_dynamic_time_warping(
    (0, 10),
    (0, 0, 10),
)
all_zero_tie = align_integer_sequences_with_dynamic_time_warping(
    (5, 5),
    (5, 5),
)
signed_extremes = align_integer_sequences_with_dynamic_time_warping(
    (-(1 << 63),),
    ((1 << 63) - 1,),
)

assert (
    expanded,
    recompute_alignment_cost((0, 10), (0, 0, 10), expanded),
    all_zero_tie,
    signed_extremes,
) == (
    DynamicTimeWarpingAlignment(0, ((0, 0), (0, 1), (1, 2))),
    0,
    DynamicTimeWarpingAlignment(0, ((0, 0), (1, 1))),
    DynamicTimeWarpingAlignment((1 << 64) - 1, ((0, 0),)),
)
```

## Trade-offs and Limitations

For sequence lengths `n` and `m`, construction takes `O(nm)` time and stores
`O(nm)` exact Python integers. Reconstruction takes at most `n + m - 1` path
steps and the immutable result stores every one of them. Python integer
arithmetic keeps costs exact even when a signed-64-bit subtraction or the
accumulated path cost exceeds that range.

The transition priority selects one minimum-cost path but does not establish
that the optimum is unique. Changing that priority can change the returned
path without changing its cost. Complete validation occurs before the table or
result is built.

This implementation has no warping window, normalization, multidimensional
points, per-position weights, approximate arithmetic, or early abandonment.
It does not stream either input, update an earlier alignment, enumerate every
optimum, or infer semantic similarity beyond exact absolute integer distance.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Canonical Minimum Unit-Cost Edit Script Between Bounded Unicode Texts](build-a-canonical-minimum-unit-cost-edit-script-between-bounded-unicode-texts.md)
- [Rank Hierarchy Paths with Bounded Weighted Edit Distance](rank-hierarchy-paths-with-bounded-weighted-edit-distance.md)
- [Match Increasing Timestamps to the Nearest Right Timestamp Within a Tolerance](../data-processing/match-increasing-timestamps-to-the-nearest-right-timestamp-within-a-tolerance.md)
<!-- catalog:related:end -->
