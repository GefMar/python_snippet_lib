---
title: "Find a Canonical Minimum-Cost Hamiltonian Cycle with Held-Karp DP"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - testing
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - find-a-lexicographically-first-minimum-cost-perfect-assignment-by-bitmask-dp.md
  - find-the-canonical-shortest-cycle-in-a-bounded-directed-graph.md
  - find-one-deterministic-shortest-path-in-a-bounded-non-negatively-weighted-directed-graph.md
---

# Find a Canonical Minimum-Cost Hamiltonian Cycle with Held-Karp DP

## Idea and Problem

Find the least-cost directed cycle that visits every indexed vertex exactly once and resolve equal totals with one canonical tour.

Anchor the open tour at vertex zero to remove rotational duplicates. A suffix
dynamic program records the minimum cost of visiting one remaining vertex set
from the current vertex and then closing back to zero. Missing directed edges
make individual transitions impossible, while signed integer costs remain safe
because every complete tour uses exactly the same number of edges.

The dynamic program retains costs rather than whole paths. During
reconstruction, scan remaining vertices in ascending order and choose the first
edge that preserves the proven optimum. Repeating that choice produces the
lexicographically smallest open tour among all minimum-cost cycles.

## When to Use

Use this algorithm for one small, fully materialized directed cost matrix when
every vertex must be visited, unavailable transitions matter, and an exact
reproducible optimum is more important than polynomial scaling. It is useful
for bounded route or sequencing references, exhaustive-test oracles, and
checking a heuristic on instances with at most fifteen vertices.

Use a specialized traveling-salesperson solver for larger instances,
approximation, branch-and-bound certificates, or repeated queries. Use a
minimum-cost assignment when successor choices need not form one cycle, or a
shortest-path algorithm when visiting every vertex is not required.

## Implementation

```python
from dataclasses import dataclass
from functools import cache

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_HAMILTONIAN_VERTEX_COUNT = 15


@dataclass(frozen=True, slots=True)
class CanonicalHamiltonianCycle:
    total_cost: int
    vertices: tuple[int, ...]


def canonical_minimum_cost_hamiltonian_cycle(
    cost_matrix: tuple[tuple[int | None, ...], ...],
) -> CanonicalHamiltonianCycle | None:
    """Return the minimum-cost tour anchored at zero, with lexical ties."""
    if type(cost_matrix) is not tuple:
        raise TypeError("cost_matrix must be an exact tuple")
    vertex_count = len(cost_matrix)
    if not 2 <= vertex_count <= _MAX_HAMILTONIAN_VERTEX_COUNT:
        raise ValueError("matrix order is outside the supported range")

    for source, row in enumerate(cost_matrix):
        if type(row) is not tuple:
            raise TypeError(f"cost_matrix[{source}] must be an exact tuple")
        if len(row) != vertex_count:
            raise ValueError("cost_matrix must be square")
        for target, edge_cost in enumerate(row):
            if source == target:
                if edge_cost is not None:
                    raise ValueError("diagonal entries must be None")
                continue
            if edge_cost is None:
                continue
            if type(edge_cost) is not int:
                raise TypeError(f"cost_matrix[{source}][{target}] must be None or an exact integer")
            if not _MIN_INT64 <= edge_cost <= _MAX_INT64:
                raise ValueError(
                    f"cost_matrix[{source}][{target}] is outside the signed 64-bit range"
                )

    @cache
    def minimum_suffix_cost(current: int, remaining: int) -> int | None:
        if remaining == 0:
            return cost_matrix[current][0]

        best: int | None = None
        candidates = remaining
        while candidates:
            next_bit = candidates & -candidates
            next_vertex = next_bit.bit_length() - 1
            candidates ^= next_bit

            edge_cost = cost_matrix[current][next_vertex]
            if edge_cost is None:
                continue
            suffix_cost = minimum_suffix_cost(next_vertex, remaining ^ next_bit)
            if suffix_cost is None:
                continue
            candidate = edge_cost + suffix_cost
            if best is None or candidate < best:
                best = candidate
        return best

    remaining = ((1 << vertex_count) - 1) ^ 1
    optimum = minimum_suffix_cost(0, remaining)
    if optimum is None:
        return None

    vertices = [0]
    current = 0
    while remaining:
        target_cost = minimum_suffix_cost(current, remaining)
        if target_cost is None:
            raise AssertionError("an optimal suffix must remain available")

        for next_vertex in range(1, vertex_count):
            next_bit = 1 << next_vertex
            if remaining & next_bit == 0:
                continue
            edge_cost = cost_matrix[current][next_vertex]
            if edge_cost is None:
                continue
            suffix_cost = minimum_suffix_cost(next_vertex, remaining ^ next_bit)
            if suffix_cost is None or edge_cost + suffix_cost != target_cost:
                continue
            vertices.append(next_vertex)
            current = next_vertex
            remaining ^= next_bit
            break
        else:
            raise AssertionError("the optimal suffix must have a next vertex")

    return CanonicalHamiltonianCycle(optimum, tuple(vertices))
```

## Example

```python
def brute_force_hamiltonian_cycle(
    cost_matrix: tuple[tuple[int | None, ...], ...],
) -> CanonicalHamiltonianCycle | None:
    from itertools import permutations

    best: CanonicalHamiltonianCycle | None = None
    for suffix in permutations(range(1, len(cost_matrix))):
        vertices = (0, *suffix)
        edge_costs: list[int] = []
        for source, target in zip(vertices, (*vertices[1:], 0), strict=True):
            edge_cost = cost_matrix[source][target]
            if edge_cost is None:
                break
            edge_costs.append(edge_cost)
        else:
            candidate = CanonicalHamiltonianCycle(sum(edge_costs), vertices)
            if best is None or (candidate.total_cost, candidate.vertices) < (
                best.total_cost,
                best.vertices,
            ):
                best = candidate
    return best


def every_three_vertex_matrix():
    from itertools import product

    positions = ((0, 1), (0, 2), (1, 0), (1, 2), (2, 0), (2, 1))
    for edge_costs in product((None, -1, 0, 2), repeat=len(positions)):
        rows: list[list[int | None]] = [[None] * 3 for _ in range(3)]
        for (source, target), edge_cost in zip(positions, edge_costs, strict=True):
            rows[source][target] = edge_cost
        yield tuple(tuple(row) for row in rows)


sparse = (
    (None, 5, None, None),
    (None, None, -2, None),
    (None, None, None, 1),
    (4, None, None, None),
)
negative = (
    (None, 4, 1, 9),
    (9, None, -5, 2),
    (3, 6, None, -4),
    (-2, 8, 7, None),
)
equal_costs = tuple(
    tuple(None if source == target else 1 for target in range(4)) for source in range(4)
)
no_tour = (
    (None, 1, None),
    (None, None, 1),
    (None, None, None),
)

checked_matrices = 0
for matrix in every_three_vertex_matrix():
    assert canonical_minimum_cost_hamiltonian_cycle(matrix) == brute_force_hamiltonian_cycle(matrix)
    checked_matrices += 1

assert (
    canonical_minimum_cost_hamiltonian_cycle(sparse),
    canonical_minimum_cost_hamiltonian_cycle(negative),
    canonical_minimum_cost_hamiltonian_cycle(equal_costs),
    canonical_minimum_cost_hamiltonian_cycle(no_tour),
    checked_matrices,
) == (
    CanonicalHamiltonianCycle(8, (0, 1, 2, 3)),
    CanonicalHamiltonianCycle(-7, (0, 1, 2, 3)),
    CanonicalHamiltonianCycle(4, (0, 1, 2, 3)),
    None,
    4_096,
)
```

## Trade-offs and Limitations

For matrix order `n`, validation takes `O(n^2)` time. The Held-Karp dynamic
program examines `O(n^2 * 2^n)` transitions in the worst case and stores
`O(n * 2^n)` exact costs or impossible-state markers. Reconstruction scans at
most `n` candidate vertices for each tour position and retains only the final
`O(n)` witness; complete path tuples are not stored in dynamic-programming
states.

The open result always starts at vertex zero, contains every vertex exactly
once, and omits the repeated closing zero. Its total includes the final edge
back to zero. Rotation is therefore fixed, while directed reversal remains a
different tour. Scanning feasible next vertices in ascending order makes the
result lexicographically smallest after total cost.

Each available edge cost is a signed-64 integer, but a complete Python-integer
total may exceed that range. Negative costs need no special cycle handling
because every state removes one unvisited vertex and every tour contains
exactly `n` edges. The function does not choose another start, revisit
vertices, attach capacities or time windows, approximate large instances,
return lower bounds, or update a stored problem incrementally.

## Related Snippets

<!-- catalog:related:start -->
- [Find a Lexicographically First Minimum-Cost Perfect Assignment by Bitmask DP](find-a-lexicographically-first-minimum-cost-perfect-assignment-by-bitmask-dp.md)
- [Find the Canonical Shortest Cycle in a Bounded Directed Graph](find-the-canonical-shortest-cycle-in-a-bounded-directed-graph.md)
- [Find One Deterministic Shortest Path in a Bounded Non-Negatively Weighted Directed Graph](find-one-deterministic-shortest-path-in-a-bounded-non-negatively-weighted-directed-graph.md)
<!-- catalog:related:end -->
