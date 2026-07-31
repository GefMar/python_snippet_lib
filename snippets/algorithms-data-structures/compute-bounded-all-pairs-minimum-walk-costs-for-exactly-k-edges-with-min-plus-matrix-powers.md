---
title: "Compute Bounded All-Pairs Minimum Walk Costs for Exactly K Edges with Min-Plus Matrix Powers"
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
  - compute-bounded-all-pairs-shortest-distances-with-floyd-warshall.md
  - compute-the-reflexive-transitive-closure-of-a-bounded-directed-graph-with-integer-bitsets.md
  - plan-a-minimum-cost-parenthesization-for-a-bounded-matrix-chain.md
---

# Compute Bounded All-Pairs Minimum Walk Costs for Exactly K Edges with Min-Plus Matrix Powers

## Idea and Problem

Find the minimum cost of a directed walk using exactly K edges for every ordered pair of vertices in a bounded dense graph.

Min-plus matrix multiplication replaces scalar multiplication with addition
and scalar addition with minimum. If matrix entries hold one-edge costs, its
`K`th min-plus power holds the best costs of exactly `K` consecutive edges.
Binary exponentiation obtains that power without iterating through all `K`
layers. The min-plus identity has zero on its diagonal and `None` elsewhere,
so the zeroth power correctly represents zero-edge walks.

`None` means that an edge or walk is absent. A numeric diagonal entry is an
explicit self-loop; the implementation never inserts free loops into the input
matrix merely because source and destination are equal.

## When to Use

Use this operation when the edge count is part of the constraint, the graph is
dense and small, and `K` may be too large for layer-by-layer dynamic
programming. Negative and zero edge costs are supported because every answer
uses a fixed finite number of edges.

The input must be an exact square tuple matrix with 1..32 vertices. Every entry
must be `None` or an exact signed 64-bit integer, and `K` must be an exact
integer in `0..10**18`. Vertex indices are the row and column positions.

Use a sparse recurrence when most entries are absent and `K` is modest. Use an
ordinary shortest-path algorithm when the number of edges is unrestricted or
bounded above rather than required exactly.

## Implementation

```python
_MIN_WALK_EDGE_COST = -(1 << 63)
_MAX_WALK_EDGE_COST = (1 << 63) - 1
_MAX_WALK_VERTICES = 32
_MAX_WALK_EDGE_COUNT = 10**18


def _validate_walk_matrix(
    weights: tuple[tuple[int | None, ...], ...],
) -> None:
    if type(weights) is not tuple:
        raise TypeError("weights must be an exact tuple")
    if not 1 <= len(weights) <= _MAX_WALK_VERTICES:
        raise ValueError("weights must contain 1..32 rows")

    size = len(weights)
    for row in weights:
        if type(row) is not tuple:
            raise TypeError("every weights row must be an exact tuple")
        if len(row) != size:
            raise ValueError("weights must be square")
        for value in row:
            if value is None:
                continue
            if type(value) is not int:
                raise TypeError("every weight must be None or an exact integer")
            if not _MIN_WALK_EDGE_COST <= value <= _MAX_WALK_EDGE_COST:
                raise ValueError("every weight must fit a signed 64-bit integer")


def _min_plus_product(
    left: tuple[tuple[int | None, ...], ...],
    right: tuple[tuple[int | None, ...], ...],
) -> tuple[tuple[int | None, ...], ...]:
    size = len(left)
    result: list[list[int | None]] = [[None] * size for _ in range(size)]

    for source in range(size):
        for middle, left_cost in enumerate(left[source]):
            if left_cost is None:
                continue
            for target, right_cost in enumerate(right[middle]):
                if right_cost is None:
                    continue
                candidate = left_cost + right_cost
                current = result[source][target]
                if current is None or candidate < current:
                    result[source][target] = candidate

    return tuple(tuple(row) for row in result)


def exact_walk_costs(
    weights: tuple[tuple[int | None, ...], ...],
    edge_count: int,
) -> tuple[tuple[int | None, ...], ...]:
    """Return all-pairs minimum costs among walks of exactly edge_count edges."""
    _validate_walk_matrix(weights)
    if type(edge_count) is not int:
        raise TypeError("edge_count must be an exact integer")
    if not 0 <= edge_count <= _MAX_WALK_EDGE_COUNT:
        raise ValueError("edge_count is outside 0..10**18")

    size = len(weights)
    result = tuple(
        tuple(0 if source == target else None for target in range(size)) for source in range(size)
    )
    power = weights
    remaining = edge_count

    while remaining:
        if remaining & 1:
            result = _min_plus_product(result, power)
        remaining >>= 1
        if remaining:
            power = _min_plus_product(power, power)

    return result
```

## Example

```python
from itertools import product


def layered_walk_costs(
    weights: tuple[tuple[int | None, ...], ...],
    edge_count: int,
) -> tuple[tuple[int | None, ...], ...]:
    """Small direct oracle that advances one edge layer at a time."""
    size = len(weights)
    answer: list[tuple[int | None, ...]] = []

    for source in range(size):
        current: list[int | None] = [None] * size
        current[source] = 0
        for _ in range(edge_count):
            following: list[int | None] = [None] * size
            for middle, prefix_cost in enumerate(current):
                if prefix_cost is None:
                    continue
                for target, edge_cost in enumerate(weights[middle]):
                    if edge_cost is None:
                        continue
                    candidate = prefix_cost + edge_cost
                    known = following[target]
                    if known is None or candidate < known:
                        following[target] = candidate
            current = following
        answer.append(tuple(current))

    return tuple(answer)


def compose_walk_costs(
    left: tuple[tuple[int | None, ...], ...],
    right: tuple[tuple[int | None, ...], ...],
) -> tuple[tuple[int | None, ...], ...]:
    """Independent direct composition used to check the split law."""
    size = len(left)
    return tuple(
        tuple(
            min(
                (
                    left[source][middle] + right[middle][target]
                    for middle in range(size)
                    if left[source][middle] is not None and right[middle][target] is not None
                ),
                default=None,
            )
            for target in range(size)
        )
        for source in range(size)
    )


checked_cases = 0
for flat_matrix in product((None, -1, 0, 2), repeat=4):
    small_matrix = (flat_matrix[:2], flat_matrix[2:])
    for small_edge_count in range(6):
        assert exact_walk_costs(small_matrix, small_edge_count) == layered_walk_costs(
            small_matrix,
            small_edge_count,
        )
        checked_cases += 1

negative_loop_graph = (
    (2, 5),
    (None, -3),
)
left_length = exact_walk_costs(negative_loop_graph, 2)
right_length = exact_walk_costs(negative_loop_graph, 3)
split_law_holds = compose_walk_costs(left_length, right_length) == exact_walk_costs(
    negative_loop_graph,
    5,
)


def rejects(weights: object, edge_count: object) -> bool:
    try:
        exact_walk_costs(weights, edge_count)  # type: ignore[arg-type]
    except (TypeError, ValueError):
        return True
    return False


assert (
    checked_cases,
    exact_walk_costs(negative_loop_graph, 4),
    exact_walk_costs(((None, 5), (None, None)), 0),
    exact_walk_costs(((None, 5), (None, None)), 2),
    exact_walk_costs(((0, None), (None, None)), 10**18),
    exact_walk_costs(((-7,),), 10**18),
    exact_walk_costs(((None,),), 10**18),
    split_law_holds,
    rejects([], 1),
    rejects((), 1),
    rejects(((0,), (0,)), 1),
    rejects(([0],), 1),
    rejects(((True,),), 1),
    rejects((((1 << 63),),), 1),
    rejects(((0,),), True),
    rejects(((0,),), -1),
    rejects(((0,),), 10**18 + 1),
) == (
    1_536,
    ((8, -4), (None, -12)),
    ((0, None), (None, 0)),
    ((None, None), (None, None)),
    ((0, None), (None, None)),
    ((-7 * 10**18,),),
    ((None,),),
    True,
    True,
    True,
    True,
    True,
    True,
    True,
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

After `O(V**2)` validation, binary exponentiation performs
`O(V**3 * log(K + 1))` integer work and retains `O(V**2)` matrix entries.
Python addition and comparison costs grow with the bit length of accumulated
costs; results can exceed the signed 64-bit input range because a walk may sum
many valid edges.

The dense cubic product is deliberately simple and predictable, but it wastes
work on sparse matrices. It returns only costs, not a walk reconstruction.
Tie-breaking between equal-cost walks is therefore irrelevant.

These are costs for exactly `K` edges, not simple paths, minimum costs over at
most `K` edges, or ordinary all-pairs shortest paths. In particular, negative
self-loops and cycles are valid and are used only as many times as the fixed
edge count permits. The function does not support incremental edge updates.

## Related Snippets

<!-- catalog:related:start -->
- [Compute Bounded All-Pairs Shortest Distances with Floyd-Warshall](compute-bounded-all-pairs-shortest-distances-with-floyd-warshall.md)
- [Compute the Reflexive Transitive Closure of a Bounded Directed Graph with Integer Bitsets](compute-the-reflexive-transitive-closure-of-a-bounded-directed-graph-with-integer-bitsets.md)
- [Plan a Minimum-Cost Parenthesization for a Bounded Matrix Chain](plan-a-minimum-cost-parenthesization-for-a-bounded-matrix-chain.md)
<!-- catalog:related:end -->
