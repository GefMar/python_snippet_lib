---
title: "Compute Bounded All-Pairs Shortest Distances with Floyd-Warshall"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-bounded-bellman-ford-distances-and-detect-reachable-negative-cycles.md
  - find-one-deterministic-shortest-path-in-a-bounded-non-negatively-weighted-directed-graph.md
  - partition-a-bounded-directed-graph-into-deterministic-strongly-connected-components.md
---

# Compute Bounded All-Pairs Shortest Distances with Floyd-Warshall

## Idea and Problem

Compute every exact directed shortest-path distance in one bounded dense graph while rejecting a negative cycle in any component.

Floyd-Warshall grows the set of vertices permitted inside a path. For each new
intermediate vertex, it compares the current source-to-target distance with the
route through that vertex. An explicit `None` distinguishes an unreachable
pair from every exact integer distance, including zero and negative values.

Every vertex is also a source, so a negative cycle anywhere in the graph makes
at least one final diagonal negative. Returning one immutable marker instead
of a partial matrix prevents a disconnected cycle from being overlooked.

## When to Use

Use this algorithm for a small, fully materialized directed graph when many or
all source-target distances are needed, signed integer edges are allowed, and
the graph is dense enough that a matrix is appropriate. It is useful for exact
bounded fixtures, global consistency checks, and repeated distance lookups
after one deterministic computation.

Use repeated Dijkstra searches for a larger sparse graph with non-negative
weights. Bellman-Ford is usually a better fit for a small number of sources
when negative weights must remain. Choose a graph library when paths, cycle
witnesses, dynamic updates, parallel edges, or substantially larger graphs are
required.

## Implementation

```python
from dataclasses import dataclass

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_FLOYD_WARSHALL_VERTICES = 128
_MAX_FLOYD_WARSHALL_EDGES = 16_384


@dataclass(frozen=True, slots=True)
class NegativeCycle:
    pass


def all_pairs_shortest_distances(
    vertex_count: int,
    edges: tuple[tuple[int, int, int], ...],
) -> tuple[tuple[int | None, ...], ...] | NegativeCycle:
    """Return all exact distances, or a marker for any negative cycle."""
    if type(vertex_count) is not int:
        raise TypeError("vertex_count must be an exact integer")
    if not 1 <= vertex_count <= _MAX_FLOYD_WARSHALL_VERTICES:
        raise ValueError("vertex_count is outside the supported range")
    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if len(edges) > _MAX_FLOYD_WARSHALL_EDGES:
        raise ValueError("edge count exceeds the supported limit")

    validated_edges: list[tuple[int, int, int]] = []
    seen_pairs: set[tuple[int, int]] = set()
    for edge_index, edge in enumerate(edges):
        if type(edge) is not tuple:
            raise TypeError(f"edges[{edge_index}] must be an exact tuple")
        if len(edge) != 3:
            raise ValueError(f"edges[{edge_index}] must contain exactly three values")

        source, target, weight = edge
        if type(source) is not int or type(target) is not int:
            raise TypeError(f"edges[{edge_index}] endpoints must be exact integers")
        if not 0 <= source < vertex_count or not 0 <= target < vertex_count:
            raise ValueError(f"edges[{edge_index}] endpoint is outside the vertex range")
        if type(weight) is not int:
            raise TypeError(f"edges[{edge_index}].weight must be an exact integer")
        if not _MIN_INT64 <= weight <= _MAX_INT64:
            raise ValueError(f"edges[{edge_index}].weight is outside the signed 64-bit range")

        pair = (source, target)
        if pair in seen_pairs:
            raise ValueError(f"edges[{edge_index}] duplicates a directed endpoint pair")
        seen_pairs.add(pair)
        validated_edges.append(edge)

    distances: list[list[int | None]] = [[None] * vertex_count for _ in range(vertex_count)]
    for vertex in range(vertex_count):
        distances[vertex][vertex] = 0
    for source, target, weight in validated_edges:
        current = distances[source][target]
        if current is None or weight < current:
            distances[source][target] = weight

    for intermediate in range(vertex_count):
        intermediate_row = distances[intermediate]
        for source in range(vertex_count):
            source_to_intermediate = distances[source][intermediate]
            if source_to_intermediate is None:
                continue

            source_row = distances[source]
            for target, intermediate_to_target in enumerate(intermediate_row):
                if intermediate_to_target is None:
                    continue
                candidate = source_to_intermediate + intermediate_to_target
                current = source_row[target]
                if current is None or candidate < current:
                    source_row[target] = candidate

    if any(distances[vertex][vertex] < 0 for vertex in range(vertex_count)):
        return NegativeCycle()
    return tuple(tuple(row) for row in distances)
```

## Example

```python
def bellman_ford_oracle(
    vertex_count: int,
    edges: tuple[tuple[int, int, int], ...],
) -> tuple[tuple[int | None, ...], ...] | NegativeCycle:
    global_distances = [0] * vertex_count
    for pass_index in range(vertex_count):
        changed = False
        for source, target, weight in edges:
            candidate = global_distances[source] + weight
            if candidate < global_distances[target]:
                global_distances[target] = candidate
                changed = True
        if not changed:
            break
        if pass_index + 1 == vertex_count:
            return NegativeCycle()

    rows: list[tuple[int | None, ...]] = []
    for start in range(vertex_count):
        distances: list[int | None] = [None] * vertex_count
        distances[start] = 0
        for _ in range(vertex_count - 1):
            changed = False
            for source, target, weight in edges:
                source_distance = distances[source]
                if source_distance is None:
                    continue
                candidate = source_distance + weight
                target_distance = distances[target]
                if target_distance is None or candidate < target_distance:
                    distances[target] = candidate
                    changed = True
            if not changed:
                break
        rows.append(tuple(distances))
    return tuple(rows)


def exercise_small_dags() -> int:
    from itertools import product

    checked = 0
    choices: tuple[int | None, ...] = (None, -2, 0, 3)
    for vertex_count in range(1, 5):
        possible_edges = tuple(
            (source, target)
            for source in range(vertex_count)
            for target in range(source + 1, vertex_count)
        )
        for weights in product(choices, repeat=len(possible_edges)):
            edges = tuple(
                (source, target, weight)
                for (source, target), weight in zip(possible_edges, weights, strict=True)
                if weight is not None
            )
            expected = bellman_ford_oracle(vertex_count, edges)
            assert all_pairs_shortest_distances(vertex_count, edges) == expected
            assert all_pairs_shortest_distances(vertex_count, tuple(reversed(edges))) == expected
            checked += 1
    return checked


def exercise_random_graphs() -> int:
    from random import Random

    generator = Random(20_260_729)
    checked = 0
    for _ in range(400):
        vertex_count = generator.randint(1, 7)
        pairs = [
            (source, target) for source in range(vertex_count) for target in range(vertex_count)
        ]
        generator.shuffle(pairs)
        edge_count = generator.randint(0, len(pairs))
        edges = tuple(
            (source, target, generator.randint(-5, 8)) for source, target in pairs[:edge_count]
        )
        expected = bellman_ford_oracle(vertex_count, edges)
        assert all_pairs_shortest_distances(vertex_count, edges) == expected
        assert all_pairs_shortest_distances(vertex_count, tuple(reversed(edges))) == expected
        checked += 1
    return checked


def raises(error_type: type[Exception], operation) -> bool:
    try:
        operation()
    except error_type:
        return True
    return False


edges = ((0, 1, 4), (0, 2, 5), (1, 2, -2), (2, 3, 3))
expected = (
    (0, 4, 2, 5, None),
    (None, 0, -2, 1, None),
    (None, None, 0, 3, None),
    (None, None, None, 0, None),
    (None, None, None, None, 0),
)

disconnected_negative_cycle = ((0, 1, 2), (2, 3, -2), (3, 2, 1))
extreme_edges = ((0, 1, _MAX_INT64), (1, 2, _MIN_INT64))
extreme_expected = (
    (0, _MAX_INT64, -1),
    (None, 0, _MIN_INT64),
    (None, None, 0),
)
dense_edges = tuple(
    (source, target, abs(source - target))
    for source in range(_MAX_FLOYD_WARSHALL_VERTICES)
    for target in range(_MAX_FLOYD_WARSHALL_VERTICES)
)
dense_expected = tuple(
    tuple(abs(source - target) for target in range(_MAX_FLOYD_WARSHALL_VERTICES))
    for source in range(_MAX_FLOYD_WARSHALL_VERTICES)
)

assert (
    exercise_small_dags(),
    exercise_random_graphs(),
    all_pairs_shortest_distances(5, edges),
    all_pairs_shortest_distances(5, tuple(reversed(edges))),
    all_pairs_shortest_distances(4, disconnected_negative_cycle),
    all_pairs_shortest_distances(2, ((1, 1, -1),)),
    all_pairs_shortest_distances(1, ((0, 0, 7),)),
    all_pairs_shortest_distances(3, extreme_edges),
    all_pairs_shortest_distances(_MAX_FLOYD_WARSHALL_VERTICES, dense_edges),
    raises(
        TypeError,
        lambda: all_pairs_shortest_distances(2, ((0, 0, -1), (0, 1, True))),
    ),
    raises(ValueError, lambda: all_pairs_shortest_distances(2, ((0, 1, 1), (0, 1, 2)))),
    raises(ValueError, lambda: all_pairs_shortest_distances(129, ())),
) == (
    4_165,
    400,
    expected,
    expected,
    NegativeCycle(),
    NegativeCycle(),
    ((0,),),
    extreme_expected,
    dense_expected,
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

Validation takes `O(V + E)` expected-time set operations and matrix
initialization takes `O(V^2)` time. Floyd-Warshall performs `O(V^3)` bigint
additions and comparisons. The normalized edges, mutable matrix, and immutable
result occupy `O(V^2 + E)` slots; the edge cap and uniqueness contract make
`E <= V^2`.

Integer addition and comparison are not constant-cost when derived distances
grow wider than machine words. Python preserves exact sums beyond the signed
64-bit input range, but the cost of those operations grows with operand bit
length. The cubic scan remains unsuitable for large sparse graphs even when
most matrix cells stay `None`.

The empty path fixes every diagonal at zero. A positive or zero self-arc cannot
increase that distance, while a negative self-arc or any other negative cycle
returns `NegativeCycle()` and discards all partial distances. `None` means no
directed path exists. The result is independent of edge declaration order. The
function does not return paths, next hops, a cycle witness, parallel-edge
selection, floating weights, dynamic updates, or a persistent graph index.

## Related Snippets

<!-- catalog:related:start -->
- [Compute Bounded Bellman-Ford Distances and Detect Reachable Negative Cycles](compute-bounded-bellman-ford-distances-and-detect-reachable-negative-cycles.md)
- [Find One Deterministic Shortest Path in a Bounded Non-Negatively Weighted Directed Graph](find-one-deterministic-shortest-path-in-a-bounded-non-negatively-weighted-directed-graph.md)
- [Partition a Bounded Directed Graph into Deterministic Strongly Connected Components](partition-a-bounded-directed-graph-into-deterministic-strongly-connected-components.md)
<!-- catalog:related:end -->
