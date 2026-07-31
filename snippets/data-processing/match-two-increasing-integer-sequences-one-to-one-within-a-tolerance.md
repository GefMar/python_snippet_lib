---
title: "Match Two Increasing Integer Sequences One-to-One Within a Tolerance"
snippet_type: algorithm
use_cases:
  - data-transformation
  - testing
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - match-increasing-timestamps-to-the-nearest-right-timestamp-within-a-tolerance.md
  - ../algorithms-data-structures/find-a-deterministic-maximum-cardinality-matching-in-a-bounded-bipartite-graph.md
  - ../algorithms-data-structures/align-bounded-integer-sequences-with-exact-dynamic-time-warping.md
---

# Match Two Increasing Integer Sequences One-to-One Within a Tolerance

## Idea and Problem

Pair increasing integer observations without reusing an item or crossing the order of the selected pairs.

An eligible pair has an absolute value difference no greater than one inclusive
tolerance. Among every order-preserving one-to-one matching, the objective
first maximizes pair count, then minimizes total absolute distance, and finally
chooses the lexicographically smallest tuple of index pairs.

A suffix dynamic-programming table considers three complete possibilities at
each pair of cursors: leave the current left item unmatched, leave the current
right item unmatched, or match the two current items when they are eligible.
Keeping the complete best witness in every cell makes all three objective
levels explicit and independently inspectable.

## When to Use

Use this algorithm for two small, complete, already increasing snapshots when
each observation may be consumed at most once and matches must preserve order.
It fits bounded timestamp reconciliation, ordered measurement alignment, and
reference calculations where cardinality takes priority over closeness.

Use an independent nearest-neighbor match when one right item may serve several
left items. Use general bipartite matching when crossing assignments are valid,
or dynamic time warping when either sequence position may be repeated along an
alignment path. Choose a specialized assignment or sequence-alignment library
for larger inputs or richer costs.

## Implementation

```python
from dataclasses import dataclass

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_DISTANCE = (1 << 64) - 1
_MAX_SEQUENCE_LENGTH = 64


@dataclass(frozen=True, slots=True)
class IncreasingToleranceMatching:
    pairs: tuple[tuple[int, int], ...]
    total_distance: int


def _validate_increasing_sequence(
    values: object,
    *,
    field: str,
) -> tuple[int, ...]:
    if type(values) is not tuple:
        raise TypeError(f"{field} must be an exact tuple")
    if len(values) > _MAX_SEQUENCE_LENGTH:
        raise ValueError(f"{field} length exceeds the supported limit")

    previous: int | None = None
    for index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"{field}[{index}] must be an exact integer")
        if not _MIN_INT64 <= value <= _MAX_INT64:
            raise ValueError(f"{field}[{index}] is outside the signed 64-bit range")
        if previous is not None and value <= previous:
            raise ValueError(f"{field} must be strictly increasing")
        previous = value
    return values


def _objective_key(
    matching: IncreasingToleranceMatching,
) -> tuple[int, int, tuple[tuple[int, int], ...]]:
    return (-len(matching.pairs), matching.total_distance, matching.pairs)


def match_increasing_sequences_within_tolerance(
    left_values: tuple[int, ...],
    right_values: tuple[int, ...],
    *,
    tolerance: int,
) -> IncreasingToleranceMatching:
    """Return the canonical best non-crossing one-to-one matching."""
    left = _validate_increasing_sequence(left_values, field="left_values")
    right = _validate_increasing_sequence(right_values, field="right_values")
    if type(tolerance) is not int:
        raise TypeError("tolerance must be an exact non-boolean integer")
    if not 0 <= tolerance <= _MAX_DISTANCE:
        raise ValueError("tolerance is outside the supported range")

    empty = IncreasingToleranceMatching((), 0)
    table = [[empty for _ in range(len(right) + 1)] for _ in range(len(left) + 1)]

    for left_index in range(len(left) - 1, -1, -1):
        for right_index in range(len(right) - 1, -1, -1):
            candidates = [
                table[left_index + 1][right_index],
                table[left_index][right_index + 1],
            ]
            distance = abs(left[left_index] - right[right_index])
            if distance <= tolerance:
                suffix = table[left_index + 1][right_index + 1]
                candidates.append(
                    IncreasingToleranceMatching(
                        pairs=((left_index, right_index), *suffix.pairs),
                        total_distance=distance + suffix.total_distance,
                    )
                )
            table[left_index][right_index] = min(
                candidates,
                key=_objective_key,
            )

    return table[0][0]
```

## Example

```python
def exhaustive_matching_oracle(
    left: tuple[int, ...],
    right: tuple[int, ...],
    tolerance: int,
) -> IncreasingToleranceMatching:
    from itertools import combinations

    candidates = [IncreasingToleranceMatching((), 0)]
    for pair_count in range(1, min(len(left), len(right)) + 1):
        for left_indexes in combinations(range(len(left)), pair_count):
            for right_indexes in combinations(range(len(right)), pair_count):
                pairs = tuple(zip(left_indexes, right_indexes, strict=True))
                distances = tuple(
                    abs(left[left_index] - right[right_index]) for left_index, right_index in pairs
                )
                if all(distance <= tolerance for distance in distances):
                    candidates.append(IncreasingToleranceMatching(pairs, sum(distances)))
    return min(
        candidates,
        key=lambda matching: (
            -len(matching.pairs),
            matching.total_distance,
            matching.pairs,
        ),
    )


def small_increasing_sequences() -> tuple[tuple[int, ...], ...]:
    from itertools import combinations

    values = (-2, -1, 0, 1, 2)
    return tuple(sequence for length in range(4) for sequence in combinations(values, length))


distance_choice = match_increasing_sequences_within_tolerance(
    (0, 10),
    (1, 8, 11),
    tolerance=3,
)
lexicographic_choice = match_increasing_sequences_within_tolerance(
    (0, 10),
    (-1, 1, 9, 11),
    tolerance=1,
)
extreme_distance = match_increasing_sequences_within_tolerance(
    (_MIN_INT64,),
    (_MAX_INT64,),
    tolerance=_MAX_DISTANCE,
)
maximum_width = match_increasing_sequences_within_tolerance(
    tuple(_MIN_INT64 + index for index in range(_MAX_SEQUENCE_LENGTH)),
    tuple(_MAX_INT64 - _MAX_SEQUENCE_LENGTH + 1 + index for index in range(_MAX_SEQUENCE_LENGTH)),
    tolerance=_MAX_DISTANCE,
)

exhaustive_count = 0
for left in small_increasing_sequences():
    for right in small_increasing_sequences():
        for allowed_distance in (0, 1, 4, _MAX_DISTANCE):
            assert match_increasing_sequences_within_tolerance(
                left,
                right,
                tolerance=allowed_distance,
            ) == exhaustive_matching_oracle(left, right, allowed_distance)
            exhaustive_count += 1


def is_rejected(
    left: object,
    right: object,
    tolerance: object,
    expected: type[Exception],
) -> bool:
    try:
        match_increasing_sequences_within_tolerance(
            left,
            right,
            tolerance=tolerance,
        )
    except expected:
        return True
    return False


invalid_calls = (
    ([0], (0,), 0, TypeError),
    ((0, True), (0,), 0, TypeError),
    ((0, 0), (0,), 0, ValueError),
    ((0,), (_MAX_INT64 + 1,), 0, ValueError),
    (tuple(range(65)), (), 0, ValueError),
    ((0,), (0,), True, TypeError),
    ((0,), (0,), _MAX_DISTANCE + 1, ValueError),
)
rejected = sum(
    is_rejected(left, right, tolerance, expected)
    for left, right, tolerance, expected in invalid_calls
)

assert (
    distance_choice,
    lexicographic_choice,
    extreme_distance,
    match_increasing_sequences_within_tolerance((), (1,), tolerance=0),
    len(maximum_width.pairs),
    maximum_width.total_distance,
    exhaustive_count,
    rejected,
) == (
    IncreasingToleranceMatching(((0, 0), (1, 2)), 2),
    IncreasingToleranceMatching(((0, 0), (1, 2)), 2),
    IncreasingToleranceMatching(((0, 0),), _MAX_DISTANCE),
    IncreasingToleranceMatching((), 0),
    _MAX_SEQUENCE_LENGTH,
    _MAX_SEQUENCE_LENGTH * (_MAX_DISTANCE - _MAX_SEQUENCE_LENGTH + 1),
    2_704,
    len(invalid_calls),
)
```

## Trade-offs and Limitations

For lengths `L` and `R`, the table contains at most `(L + 1) * (R + 1)`
cells, or 4,225 cells under the 64-item-per-side caps. If
`K = min(L, R)`, constructing and comparing the retained pair tuples takes
`O(L * R * K)` time in the worst case, and all stored immutable witnesses can
hold `O(L * R * K)` index-pair references. This deliberately simple form is
intended for bounded reference work, not large production alignment jobs.

Inputs must already be strictly increasing; the function does not sort or
deduplicate them. Every index is used at most once, and the returned index pairs
increase on both sides. An empty side or absence of eligible pairs has the
valid empty optimum with distance zero.

Subtraction can produce the full unsigned-64 distance between signed-64
endpoints, and summing several distances can exceed both signed- and
unsigned-64 ranges. Python integers preserve that exact total. The function
does not attach payloads, mutate or consume a stream, accept floating-point
values, allow crossing or many-to-one pairs, or support asymmetric, weighted,
or application-specific costs.

## Related Snippets

<!-- catalog:related:start -->
- [Match Increasing Timestamps to the Nearest Right Timestamp Within a Tolerance](match-increasing-timestamps-to-the-nearest-right-timestamp-within-a-tolerance.md)
- [Find a Deterministic Maximum-Cardinality Matching in a Bounded Bipartite Graph](../algorithms-data-structures/find-a-deterministic-maximum-cardinality-matching-in-a-bounded-bipartite-graph.md)
- [Align Bounded Integer Sequences with Exact Dynamic Time Warping](../algorithms-data-structures/align-bounded-integer-sequences-with-exact-dynamic-time-warping.md)
<!-- catalog:related:end -->
