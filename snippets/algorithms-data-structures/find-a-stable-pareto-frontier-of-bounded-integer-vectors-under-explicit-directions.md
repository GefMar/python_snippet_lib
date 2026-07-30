---
title: "Find a Stable Pareto Frontier of Bounded Integer Vectors under Explicit Directions"
snippet_type: algorithm
use_cases:
  - data-transformation
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - select-the-stable-k-smallest-bounded-records-with-a-max-heap.md
  - rank-bounded-records-with-stable-ties-and-neighbor-windows.md
  - compute-a-canonical-convex-hull-for-bounded-integer-points.md
---

# Find a Stable Pareto Frontier of Bounded Integer Vectors under Explicit Directions

## Idea and Problem

Find every nondominated record in a bounded multi-objective data set without collapsing the objectives into an arbitrary scalar score.

One vector dominates another when it is no worse in every objective and
strictly better in at least one, using a declared minimize or maximize direction
per coordinate. The result preserves input order and gives every excluded
record an earliest input-order frontier member that proves its domination.

## When to Use

Use this scan for small decision tables, test oracles, and deterministic
pre-filtering when objectives are exact integers and trade-offs should remain
visible. It is useful when there is no defensible weighting that turns several
objectives into one total ranking.

Use a specialized multi-objective algorithm for large or frequently changing
sets. Define domain-specific handling before calling this function if values can
be missing, approximate, constrained, or uncertain.

## Implementation

```python
from dataclasses import dataclass
from enum import StrEnum

_MIN_INT64 = -(2**63)
_MAX_INT64 = 2**63 - 1
_MAX_RECORDS = 1_024
_MAX_DIMENSIONS = 8
_MAX_KEY_BYTES = 128


class ObjectiveDirection(StrEnum):
    MINIMIZE = "minimize"
    MAXIMIZE = "maximize"


@dataclass(frozen=True, slots=True)
class ParetoRecord:
    key: str
    values: tuple[int, ...]


@dataclass(frozen=True, slots=True)
class DominanceWitness:
    record_key: str
    dominator_key: str


@dataclass(frozen=True, slots=True)
class ParetoFrontier:
    frontier_keys: tuple[str, ...]
    dominated: tuple[DominanceWitness, ...]


def _dominates(
    left: tuple[int, ...],
    right: tuple[int, ...],
    directions: tuple[ObjectiveDirection, ...],
) -> bool:
    strictly_better = False
    for left_value, right_value, direction in zip(
        left,
        right,
        directions,
        strict=True,
    ):
        if direction is ObjectiveDirection.MINIMIZE:
            if left_value > right_value:
                return False
            strictly_better |= left_value < right_value
        else:
            if left_value < right_value:
                return False
            strictly_better |= left_value > right_value
    return strictly_better


def find_pareto_frontier(
    records: tuple[ParetoRecord, ...],
    directions: tuple[ObjectiveDirection, ...],
) -> ParetoFrontier:
    if type(directions) is not tuple:
        raise TypeError("directions must be an exact tuple")
    if not 2 <= len(directions) <= _MAX_DIMENSIONS:
        raise ValueError("objective count is outside 2..8")
    for index, direction in enumerate(directions):
        if type(direction) is not ObjectiveDirection:
            raise TypeError(f"directions[{index}] must be an exact ObjectiveDirection")

    if type(records) is not tuple:
        raise TypeError("records must be an exact tuple")
    if len(records) > _MAX_RECORDS:
        raise ValueError("record count exceeds 1024")

    checked: list[ParetoRecord] = []
    for record_index, record in enumerate(records):
        if type(record) is not ParetoRecord:
            raise TypeError(f"records[{record_index}] must be an exact ParetoRecord")
        if type(record.key) is not str:
            raise TypeError(f"records[{record_index}].key must be an exact string")
        key_bytes = len(record.key.encode("utf-8"))
        if not 1 <= key_bytes <= _MAX_KEY_BYTES:
            raise ValueError(f"records[{record_index}].key must contain 1..128 UTF-8 bytes")
        if type(record.values) is not tuple:
            raise TypeError(f"records[{record_index}].values must be an exact tuple")
        if len(record.values) != len(directions):
            raise ValueError("every objective vector must match directions")
        for value_index, value in enumerate(record.values):
            if type(value) is not int:
                raise TypeError(
                    f"records[{record_index}].values[{value_index}] must be an exact integer"
                )
            if not _MIN_INT64 <= value <= _MAX_INT64:
                raise ValueError(
                    f"records[{record_index}].values[{value_index}] is outside signed int64"
                )
        checked.append(record)

    if len({record.key for record in checked}) != len(checked):
        raise ValueError("record keys must be unique")

    frontier_indexes = tuple(
        candidate_index
        for candidate_index, candidate in enumerate(checked)
        if not any(
            other_index != candidate_index
            and _dominates(other.values, candidate.values, directions)
            for other_index, other in enumerate(checked)
        )
    )

    frontier_set = set(frontier_indexes)
    witnesses: list[DominanceWitness] = []
    for record_index, record in enumerate(checked):
        if record_index in frontier_set:
            continue
        dominator_index = next(
            index
            for index in frontier_indexes
            if _dominates(checked[index].values, record.values, directions)
        )
        witnesses.append(
            DominanceWitness(record.key, checked[dominator_index].key)
        )

    return ParetoFrontier(
        tuple(checked[index].key for index in frontier_indexes),
        tuple(witnesses),
    )
```

## Example

```python
directions = (
    ObjectiveDirection.MINIMIZE,
    ObjectiveDirection.MAXIMIZE,
)
records = (
    ParetoRecord("dominated-first", (5, 1)),
    ParetoRecord("intermediate", (4, 2)),
    ParetoRecord("fast", (1, 3)),
    ParetoRecord("capable", (3, 5)),
    ParetoRecord("capable-tie", (3, 5)),
)
result = find_pareto_frontier(records, directions)


def literal_dominates(
    left: tuple[int, ...],
    right: tuple[int, ...],
    directions: tuple[ObjectiveDirection, ...],
) -> bool:
    comparisons = tuple(
        (left_value <= right_value, left_value < right_value)
        if direction is ObjectiveDirection.MINIMIZE
        else (left_value >= right_value, left_value > right_value)
        for left_value, right_value, direction in zip(
            left,
            right,
            directions,
            strict=True,
        )
    )
    return all(no_worse for no_worse, _ in comparisons) and any(
        better for _, better in comparisons
    )


def check_tiny_grids() -> int:
    from itertools import product

    checked_cases = 0
    for direction_pair in product(tuple(ObjectiveDirection), repeat=2):
        grid = tuple(
            ParetoRecord(f"p-{left}-{right}", (left, right))
            for left, right in product(range(3), repeat=2)
        )
        observed = find_pareto_frontier(grid, direction_pair)
        expected_indexes = tuple(
            index
            for index, candidate in enumerate(grid)
            if not any(
                other_index != index
                and literal_dominates(other.values, candidate.values, direction_pair)
                for other_index, other in enumerate(grid)
            )
        )
        expected_keys = tuple(grid[index].key for index in expected_indexes)
        expected_witnesses = tuple(
            DominanceWitness(
                candidate.key,
                next(
                    grid[index].key
                    for index in expected_indexes
                    if literal_dominates(
                        grid[index].values,
                        candidate.values,
                        direction_pair,
                    )
                ),
            )
            for candidate_index, candidate in enumerate(grid)
            if candidate_index not in set(expected_indexes)
        )
        assert observed == ParetoFrontier(expected_keys, expected_witnesses)
        checked_cases += 1
    return checked_cases


checked_cases = check_tiny_grids()

equal_vectors = find_pareto_frontier(
    (
        ParetoRecord("same-a", (2, 2)),
        ParetoRecord("same-b", (2, 2)),
    ),
    (ObjectiveDirection.MINIMIZE, ObjectiveDirection.MINIMIZE),
)
endpoints = find_pareto_frontier(
    (
        ParetoRecord("low-high", (_MIN_INT64, _MAX_INT64)),
        ParetoRecord("high-low", (_MAX_INT64, _MIN_INT64)),
    ),
    directions,
)

assert (
    result.frontier_keys == ("fast", "capable", "capable-tie")
    and result.dominated
    == (
        DominanceWitness("dominated-first", "fast"),
        DominanceWitness("intermediate", "fast"),
    )
    and checked_cases == 4
    and equal_vectors.frontier_keys == ("same-a", "same-b")
    and endpoints.frontier_keys == ("low-high",)
)
```

## Trade-offs and Limitations

The transparent scan takes `O(N**2 * D)` time for `N` records and `D`
objectives, then up to the same order of work to select frontier witnesses. It
uses `O(N)` auxiliary space. The explicit bounds favor auditability over a more
complex index that would help much larger inputs.

Distinct records with identical vectors do not dominate one another and remain
together unless another vector dominates both. A witness is always an actual
frontier member, not merely an earlier dominated record. The result is a partial
order summary: it does not rank frontier members, assign weights, approximate
floats, apply epsilon dominance, or claim that the frontier is a convex hull.

## Related Snippets

<!-- catalog:related:start -->
- [Select the Stable K Smallest Bounded Records with a Max-Heap](select-the-stable-k-smallest-bounded-records-with-a-max-heap.md)
- [Rank Bounded Records with Stable Ties and Neighbor Windows](rank-bounded-records-with-stable-ties-and-neighbor-windows.md)
- [Compute a Canonical Convex Hull for Bounded Integer Points](compute-a-canonical-convex-hull-for-bounded-integer-points.md)
<!-- catalog:related:end -->
