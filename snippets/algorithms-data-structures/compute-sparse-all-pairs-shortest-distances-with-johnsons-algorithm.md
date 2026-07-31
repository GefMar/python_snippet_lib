---
title: "Compute Sparse All-Pairs Shortest Distances with Johnson's Algorithm"
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
  - compute-bounded-bellman-ford-distances-and-detect-reachable-negative-cycles.md
  - find-one-deterministic-shortest-path-in-a-bounded-non-negatively-weighted-directed-graph.md
---

# Compute Sparse All-Pairs Shortest Distances with Johnson's Algorithm

## Idea and Problem

Compute every exact shortest-path distance in a bounded sparse directed graph that may contain negative edges, while rejecting a negative cycle in any component.

Johnson's algorithm first runs Bellman-Ford from an implicit synthetic source
with a zero-weight edge to every vertex. The resulting potentials transform
each edge to a non-negative weight without changing which paths are shortest.
Dijkstra's algorithm can then compute one matrix row efficiently from every
vertex, after which the potentials restore the original exact distances.

Initializing every potential to zero has the same effect as materializing the
synthetic source. It also ensures that Bellman-Ford detects a negative cycle
even when no original vertex outside that component can reach it.

## When to Use

Use this algorithm when all-pairs distances are needed from a static sparse
directed graph, negative edge weights are valid, and negative cycles must
invalidate the complete result. It is useful for bounded cost models and
repeated exact distance lookups where a dense cubic scan would waste work.

Use Floyd-Warshall for a small dense graph or when its simpler matrix update is
preferable. Use Bellman-Ford for only one or a few sources with negative edges,
and ordinary Dijkstra for non-negative graphs queried from few sources. Choose
a graph library when paths, cycle witnesses, parallel edges, dynamic updates,
or substantially larger graphs are required.

## Implementation

```python
from dataclasses import dataclass
from heapq import heappop, heappush

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_JOHNSON_VERTICES = 256
_MAX_JOHNSON_EDGES = 2_048


@dataclass(frozen=True, slots=True)
class NegativeCycle:
    pass


def johnson_all_pairs_distances(
    vertex_count: int,
    edges: tuple[tuple[int, int, int], ...],
) -> tuple[tuple[int | None, ...], ...] | NegativeCycle:
    """Return every exact distance, or a marker for any negative cycle."""
    if type(vertex_count) is not int:
        raise TypeError("vertex_count must be an exact integer")
    if not 1 <= vertex_count <= _MAX_JOHNSON_VERTICES:
        raise ValueError("vertex_count is outside the supported range")
    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if len(edges) > _MAX_JOHNSON_EDGES:
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

    validated_edges.sort()

    # These zeros are the distances over implicit zero-weight synthetic edges.
    potentials = [0] * vertex_count
    for pass_index in range(vertex_count):
        changed = False
        for source, target, weight in validated_edges:
            candidate = potentials[source] + weight
            if candidate < potentials[target]:
                potentials[target] = candidate
                changed = True
        if not changed:
            break
        if pass_index + 1 == vertex_count:
            return NegativeCycle()

    adjacency: list[list[tuple[int, int]]] = [[] for _ in range(vertex_count)]
    for source, target, weight in validated_edges:
        reweighted = weight + potentials[source] - potentials[target]
        adjacency[source].append((target, reweighted))

    rows: list[tuple[int | None, ...]] = []
    for start in range(vertex_count):
        reweighted_distances: list[int | None] = [None] * vertex_count
        reweighted_distances[start] = 0
        frontier = [(0, start)]

        while frontier:
            distance, source = heappop(frontier)
            if reweighted_distances[source] != distance:
                continue
            for target, weight in adjacency[source]:
                candidate = distance + weight
                known = reweighted_distances[target]
                if known is None or candidate < known:
                    reweighted_distances[target] = candidate
                    heappush(frontier, (candidate, target))

        rows.append(
            tuple(
                None if distance is None else distance - potentials[start] + potentials[target]
                for target, distance in enumerate(reweighted_distances)
            )
        )

    return tuple(rows)
```

## Example

```python
def floyd_warshall_oracle(
    vertex_count: int,
    edges: tuple[tuple[int, int, int], ...],
) -> tuple[tuple[int | None, ...], ...] | NegativeCycle:
    distances: list[list[int | None]] = [[None] * vertex_count for _ in range(vertex_count)]
    for vertex in range(vertex_count):
        distances[vertex][vertex] = 0
    for source, target, weight in edges:
        current = distances[source][target]
        if current is None or weight < current:
            distances[source][target] = weight

    for intermediate in range(vertex_count):
        for source in range(vertex_count):
            prefix = distances[source][intermediate]
            if prefix is None:
                continue
            for target in range(vertex_count):
                suffix = distances[intermediate][target]
                if suffix is None:
                    continue
                candidate = prefix + suffix
                current = distances[source][target]
                if current is None or candidate < current:
                    distances[source][target] = candidate

    if any(distances[vertex][vertex] < 0 for vertex in range(vertex_count)):
        return NegativeCycle()
    return tuple(tuple(row) for row in distances)


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
            expected = floyd_warshall_oracle(vertex_count, edges)
            assert johnson_all_pairs_distances(vertex_count, edges) == expected
            assert johnson_all_pairs_distances(vertex_count, tuple(reversed(edges))) == expected
            checked += 1
    return checked


def exercise_potential_reweighted_graphs() -> int:
    from random import Random

    generator = Random(20_260_731)
    checked = 0
    for _ in range(300):
        vertex_count = generator.randint(1, 8)
        potentials = [generator.randint(-20, 20) for _ in range(vertex_count)]
        pairs = [
            (source, target) for source in range(vertex_count) for target in range(vertex_count)
        ]
        generator.shuffle(pairs)
        edge_count = generator.randint(0, min(len(pairs), 3 * vertex_count))
        edges = tuple(
            (
                source,
                target,
                generator.randint(0, 9) + potentials[target] - potentials[source],
            )
            for source, target in pairs[:edge_count]
        )
        expected = floyd_warshall_oracle(vertex_count, edges)
        assert johnson_all_pairs_distances(vertex_count, edges) == expected
        assert johnson_all_pairs_distances(vertex_count, tuple(reversed(edges))) == expected
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
disconnected_negative_cycle = ((0, 1, 2), (3, 4, -2), (4, 3, 1))
extreme_edges = ((0, 1, _MAX_INT64), (1, 2, _MIN_INT64))
extreme_expected = (
    (0, _MAX_INT64, -1),
    (None, 0, _MIN_INT64),
    (None, None, 0),
)
maximum_edges = tuple(
    (source, (source + offset) % _MAX_JOHNSON_VERTICES, offset)
    for source in range(_MAX_JOHNSON_VERTICES)
    for offset in range(8)
)
maximum_expected = tuple(
    tuple((target - source) % _MAX_JOHNSON_VERTICES for target in range(_MAX_JOHNSON_VERTICES))
    for source in range(_MAX_JOHNSON_VERTICES)
)

assert (
    exercise_small_dags(),
    exercise_potential_reweighted_graphs(),
    johnson_all_pairs_distances(5, edges),
    johnson_all_pairs_distances(5, tuple(reversed(edges))),
    johnson_all_pairs_distances(5, disconnected_negative_cycle),
    johnson_all_pairs_distances(2, ((1, 1, -1),)),
    johnson_all_pairs_distances(1, ((0, 0, 7),)),
    johnson_all_pairs_distances(3, extreme_edges),
    johnson_all_pairs_distances(_MAX_JOHNSON_VERTICES, maximum_edges),
    raises(TypeError, lambda: johnson_all_pairs_distances(True, ())),
    raises(TypeError, lambda: johnson_all_pairs_distances(2, ((0, 1, True),))),
    raises(TypeError, lambda: johnson_all_pairs_distances(2, ([0, 1, 2],))),
    raises(ValueError, lambda: johnson_all_pairs_distances(2, ((0, 1, 1), (0, 1, 2)))),
    raises(ValueError, lambda: johnson_all_pairs_distances(257, ())),
    raises(ValueError, lambda: johnson_all_pairs_distances(256, (*maximum_edges, (0, 0, 0)))),
) == (
    4_165,
    300,
    expected,
    expected,
    NegativeCycle(),
    NegativeCycle(),
    ((0,),),
    extreme_expected,
    maximum_expected,
    True,
    True,
    True,
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

Validation and deterministic edge sorting cost `O(V + E log E)` time. The
synthetic-source Bellman-Ford phase costs `O(VE)`, and all Dijkstra runs cost
`O(V(E + V) log V)` in total. The returned matrix, normalized graph,
potentials, one-source state, and heap occupy `O(V^2 + E)` slots.

Edge weights must be signed 64-bit integers, but Python keeps potentials,
reweighted weights, and path totals exact when derived values exceed that
range. Big-integer addition and comparison are not constant-cost. The result
has a zero diagonal, uses `None` for unreachable pairs, and is independent of
edge declaration order. Self-loops are accepted, but directed endpoint pairs
must be unique.

Any negative cycle returns one frozen `NegativeCycle()` marker and discards all
partial distances; the marker carries no cycle witness. The function does not
return paths or predecessors, select among parallel edges, accept floats,
reuse work across calls, handle graph updates, or provide a persistent index.
For dense graphs, its heap overhead can make Floyd-Warshall a better choice.

## Related Snippets

<!-- catalog:related:start -->
- [Compute Bounded All-Pairs Shortest Distances with Floyd-Warshall](compute-bounded-all-pairs-shortest-distances-with-floyd-warshall.md)
- [Compute Bounded Bellman-Ford Distances and Detect Reachable Negative Cycles](compute-bounded-bellman-ford-distances-and-detect-reachable-negative-cycles.md)
- [Find One Deterministic Shortest Path in a Bounded Non-Negatively Weighted Directed Graph](find-one-deterministic-shortest-path-in-a-bounded-non-negatively-weighted-directed-graph.md)
<!-- catalog:related:end -->
