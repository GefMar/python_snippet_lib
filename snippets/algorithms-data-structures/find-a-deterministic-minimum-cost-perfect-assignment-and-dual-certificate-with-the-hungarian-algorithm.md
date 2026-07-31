---
title: "Find a Deterministic Minimum-Cost Assignment and Dual Certificate with the Hungarian Algorithm"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - find-a-lexicographically-first-minimum-cost-perfect-assignment-by-bitmask-dp.md
  - find-a-deterministic-maximum-cardinality-matching-in-a-bounded-bipartite-graph.md
  - compute-a-bounded-integer-maximum-flow-and-canonical-minimum-cut-with-edmonds-karp.md
---

# Find a Deterministic Minimum-Cost Assignment and Dual Certificate with the Hungarian Algorithm

## Idea and Problem

Assign every row of a bounded square integer cost matrix to a different column and return exact dual potentials that independently certify the minimum total cost.

The Hungarian algorithm augments one row at a time. Reduced costs relative to
row and column potentials identify the cheapest way to reach each unused
column. Updating both potentials by the smallest current slack exposes at
least one new tight edge, and predecessor columns reconstruct the augmented
matching.

At completion, every reduced cost is non-negative and every selected edge is
tight. Those dual conditions plus equal primal and dual totals prove that the
returned assignment is globally optimal without enumerating permutations.

## When to Use

Use this algorithm for a complete square assignment problem when exact integer
costs, polynomial scaling, deterministic output, and a checkable optimality
certificate matter. It is suitable for matrices too large for subset dynamic
programming but small enough to retain densely in memory.

Use the bitmask dynamic program when order is at most 18 and the
lexicographically first optimum is required. Use minimum-cost flow or an
optimization library for forbidden pairs, rectangular sides, capacities,
partial assignments, floating-point objectives, or additional constraints.

## Implementation

```python
from dataclasses import dataclass

_MIN_HUNGARIAN_INT64 = -(1 << 63)
_MAX_HUNGARIAN_INT64 = (1 << 63) - 1
_MAX_HUNGARIAN_ORDER = 256


@dataclass(frozen=True, slots=True)
class HungarianAssignment:
    total_cost: int
    columns_by_row: tuple[int, ...]
    row_potentials: tuple[int, ...]
    column_potentials: tuple[int, ...]


def hungarian_assignment(
    matrix: tuple[tuple[int, ...], ...],
) -> HungarianAssignment:
    """Return one deterministic optimum and an exact dual certificate."""
    if type(matrix) is not tuple:
        raise TypeError("matrix must be an exact tuple")
    order = len(matrix)
    if order > _MAX_HUNGARIAN_ORDER:
        raise ValueError("matrix order exceeds 256")
    for row_index, row in enumerate(matrix):
        if type(row) is not tuple:
            raise TypeError(f"matrix[{row_index}] must be an exact tuple")
        if len(row) != order:
            raise ValueError("matrix must be square")
        for column_index, cost in enumerate(row):
            if type(cost) is not int:
                raise TypeError(f"matrix[{row_index}][{column_index}] must be an exact integer")
            if not _MIN_HUNGARIAN_INT64 <= cost <= _MAX_HUNGARIAN_INT64:
                raise ValueError(f"matrix[{row_index}][{column_index}] is outside signed int64")

    if order == 0:
        return HungarianAssignment(0, (), (), ())

    row_potentials = [0] * (order + 1)
    column_potentials = [0] * (order + 1)
    row_by_column = [0] * (order + 1)
    predecessor_column = [0] * (order + 1)

    for added_row in range(1, order + 1):
        row_by_column[0] = added_row
        minimum_slack: list[int | None] = [None] * (order + 1)
        used_columns = [False] * (order + 1)
        current_column = 0

        while True:
            used_columns[current_column] = True
            current_row = row_by_column[current_column]
            delta: int | None = None
            next_column = 0

            for column in range(1, order + 1):
                if used_columns[column]:
                    continue
                reduced_cost = (
                    matrix[current_row - 1][column - 1]
                    - row_potentials[current_row]
                    - column_potentials[column]
                )
                slack = minimum_slack[column]
                if slack is None or reduced_cost < slack:
                    minimum_slack[column] = reduced_cost
                    predecessor_column[column] = current_column
                    slack = reduced_cost
                if delta is None or slack < delta:
                    delta = slack
                    next_column = column

            if delta is None:
                raise RuntimeError("assignment augmentation found no unused column")

            for column in range(order + 1):
                if used_columns[column]:
                    row_potentials[row_by_column[column]] += delta
                    column_potentials[column] -= delta
                elif column:
                    slack = minimum_slack[column]
                    if slack is None:
                        raise RuntimeError("unused column has no finite assignment slack")
                    minimum_slack[column] = slack - delta

            current_column = next_column
            if row_by_column[current_column] == 0:
                break

        while current_column:
            previous_column = predecessor_column[current_column]
            row_by_column[current_column] = row_by_column[previous_column]
            current_column = previous_column

    columns_by_row = [0] * order
    for column in range(1, order + 1):
        columns_by_row[row_by_column[column] - 1] = column - 1

    raw_rows = row_potentials[1:]
    raw_columns = column_potentials[1:]
    shift = raw_rows[0]
    normalized_rows = tuple(value - shift for value in raw_rows)
    normalized_columns = tuple(value + shift for value in raw_columns)
    assignment = tuple(columns_by_row)
    total_cost = sum(matrix[row][column] for row, column in enumerate(assignment))
    return HungarianAssignment(
        total_cost=total_cost,
        columns_by_row=assignment,
        row_potentials=normalized_rows,
        column_potentials=normalized_columns,
    )
```

## Example

```python
from itertools import permutations, product
from random import Random


def verify_assignment_certificate(
    matrix: tuple[tuple[int, ...], ...],
    result: HungarianAssignment,
) -> None:
    order = len(matrix)
    assert tuple(sorted(result.columns_by_row)) == tuple(range(order))
    assert len(result.row_potentials) == order
    assert len(result.column_potentials) == order
    if order:
        assert result.row_potentials[0] == 0

    primal = sum(matrix[row][column] for row, column in enumerate(result.columns_by_row))
    dual = sum(result.row_potentials) + sum(result.column_potentials)
    assert result.total_cost == primal == dual

    for row in range(order):
        for column in range(order):
            assert (
                result.row_potentials[row] + result.column_potentials[column] <= matrix[row][column]
            )
        assigned_column = result.columns_by_row[row]
        assert (
            result.row_potentials[row] + result.column_potentials[assigned_column]
            == matrix[row][assigned_column]
        )


def brute_assignment_cost(matrix: tuple[tuple[int, ...], ...]) -> int:
    if not matrix:
        return 0
    return min(
        sum(matrix[row][column] for row, column in enumerate(columns))
        for columns in permutations(range(len(matrix)))
    )


checked = 0
for flat_values in product((-2, 0, 3), repeat=4):
    matrix = (flat_values[:2], flat_values[2:])
    result = hungarian_assignment(matrix)
    verify_assignment_certificate(matrix, result)
    assert result.total_cost == brute_assignment_cost(matrix)
    checked += 1

rng = Random(0)
for order in range(1, 8):
    for _ in range(12):
        matrix = tuple(tuple(rng.randint(-100, 100) for _ in range(order)) for _ in range(order))
        result = hungarian_assignment(matrix)
        verify_assignment_certificate(matrix, result)
        assert result.total_cost == brute_assignment_cost(matrix)
        checked += 1

tied = ((0, 0, 0),) * 3
tied_first = hungarian_assignment(tied)
tied_second = hungarian_assignment(tied)
verify_assignment_certificate(tied, tied_first)

boundary = (
    (_MAX_HUNGARIAN_INT64, _MIN_HUNGARIAN_INT64),
    (_MIN_HUNGARIAN_INT64, _MAX_HUNGARIAN_INT64),
)
boundary_result = hungarian_assignment(boundary)
verify_assignment_certificate(boundary, boundary_result)

maximum_matrix = tuple(
    tuple(0 if row == column else abs(row - column) + 1 for column in range(256))
    for row in range(256)
)
maximum_result = hungarian_assignment(maximum_matrix)
verify_assignment_certificate(maximum_matrix, maximum_result)


class TupleSubclass(tuple):
    pass


rejected = 0
invalid_matrices = (
    TupleSubclass(()),
    (TupleSubclass((0,)),),
    ((0, 1),),
    ((True,),),
    ((_MAX_HUNGARIAN_INT64 + 1,),),
    tuple((0,) * 257 for _ in range(257)),
)
for invalid_matrix in invalid_matrices:
    try:
        hungarian_assignment(invalid_matrix)
    except (TypeError, ValueError):
        rejected += 1

assert (
    checked == 165
    and hungarian_assignment(()) == HungarianAssignment(0, (), (), ())
    and tied_first == tied_second
    and boundary_result.total_cost == 2 * _MIN_HUNGARIAN_INT64
    and maximum_result.total_cost == 0
    and maximum_result.columns_by_row == tuple(range(256))
    and rejected == len(invalid_matrices)
)
```

## Trade-offs and Limitations

For matrix order `N`, the algorithm performs `O(N**3)` exact-integer work and
uses `O(N)` mutable matching, slack, predecessor, and potential state beyond
the retained `O(N**2)` input. The returned assignment and potentials use
`O(N)` space. Derived totals and potentials can exceed signed 64-bit range and
therefore remain exact Python integers.

The certificate is independently checkable in `O(N**2)` time: all dual
inequalities must hold, assigned edges must be tight, and primal and dual
totals must agree. Potentials have an arbitrary additive gauge, so this profile
normalizes the first row potential to zero. The matching is deterministic for
one matrix but is not promised to be the lexicographically first optimum.

The matrix must be complete, square, dense, and integer-valued. The function
does not represent forbidden pairs or infinity, handle rectangular or partial
assignments, optimize several objectives, accept floating-point costs, update
an existing matching, or replace a specialized optimization library.

## Related Snippets

<!-- catalog:related:start -->
- [Find a Lexicographically First Minimum-Cost Perfect Assignment by Bitmask DP](find-a-lexicographically-first-minimum-cost-perfect-assignment-by-bitmask-dp.md)
- [Find a Deterministic Maximum-Cardinality Matching in a Bounded Bipartite Graph](find-a-deterministic-maximum-cardinality-matching-in-a-bounded-bipartite-graph.md)
- [Compute a Bounded Integer Maximum Flow and Canonical Minimum Cut with Edmonds-Karp](compute-a-bounded-integer-maximum-flow-and-canonical-minimum-cut-with-edmonds-karp.md)
<!-- catalog:related:end -->
