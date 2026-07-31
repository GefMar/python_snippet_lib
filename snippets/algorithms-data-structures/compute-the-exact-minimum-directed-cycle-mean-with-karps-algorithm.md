---
title: "Compute the Exact Minimum Directed Cycle Mean with Karp's Algorithm"
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
  - find-the-canonical-shortest-cycle-in-a-bounded-directed-graph.md
  - compute-bounded-bellman-ford-distances-and-detect-reachable-negative-cycles.md
  - partition-a-bounded-directed-graph-into-deterministic-strongly-connected-components.md
---

# Compute the Exact Minimum Directed Cycle Mean with Karp's Algorithm

## Idea and Problem

Find the smallest average edge weight of any directed cycle without confusing that objective with minimum total cycle weight.

A long negative cycle can have a lower total but a higher per-edge mean than a
short cycle. Karp's dynamic program records the minimum weight of an exact-
length walk from one source to every vertex. Inside a strongly connected
component of size `m`, differences between the length-`m` row and earlier rows
recover its minimum cycle mean.

A general graph is partitioned into strongly connected components first.
Only a component with multiple vertices or a self-loop can contain a cycle;
the smallest exact component result is the answer for the whole graph.

## When to Use

Use this calculation for bounded directed state, scheduling, or cost graphs
when sustained average edge cost is the quantity of interest and exact
rational comparison matters. Negative weights and self-loops are valid, and
disconnected cyclic regions are compared independently.

Use a shortest-cycle algorithm when total weight or edge count is the actual
objective. Choose a specialized implementation when an attaining cycle must be
reconstructed, parallel edges must retain identity, or the graph is too large
for `O(VE)` dynamic programming.

## Implementation

```python
from fractions import Fraction
from random import Random

_MAX_MEAN_CYCLE_VERTICES = 128
_MAX_MEAN_CYCLE_EDGES = 4_096
_MIN_SIGNED_32 = -(2**31)
_MAX_SIGNED_32 = 2**31 - 1


def exact_minimum_directed_cycle_mean(
    vertex_count: int,
    edges: tuple[tuple[int, int, int], ...],
) -> Fraction | None:
    """Return the exact minimum directed-cycle mean, or None if acyclic."""
    if type(vertex_count) is not int:
        raise TypeError("vertex_count must be an exact integer")
    if not 1 <= vertex_count <= _MAX_MEAN_CYCLE_VERTICES:
        raise ValueError("vertex_count is outside 1..128")
    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if len(edges) > _MAX_MEAN_CYCLE_EDGES:
        raise ValueError("edge count exceeds 4096")

    endpoint_pairs: set[tuple[int, int]] = set()
    outgoing: list[list[tuple[int, int]]] = [[] for _ in range(vertex_count)]
    for edge_index, edge in enumerate(edges):
        if type(edge) is not tuple or len(edge) != 3:
            raise TypeError(f"edges[{edge_index}] must be an exact triple")
        source, target, weight = edge
        if type(source) is not int or type(target) is not int:
            raise TypeError("edge endpoints must be exact integers")
        if not 0 <= source < vertex_count or not 0 <= target < vertex_count:
            raise ValueError("edge endpoint is outside the graph")
        if type(weight) is not int:
            raise TypeError("edge weights must be exact integers")
        if not _MIN_SIGNED_32 <= weight <= _MAX_SIGNED_32:
            raise ValueError("edge weight is outside the signed 32-bit range")
        pair = source, target
        if pair in endpoint_pairs:
            raise ValueError("parallel endpoint pairs are not supported")
        endpoint_pairs.add(pair)
        outgoing[source].append((target, weight))

    discovery = [-1] * vertex_count
    low = [0] * vertex_count
    on_stack = [False] * vertex_count
    stack: list[int] = []
    components: list[tuple[int, ...]] = []
    next_discovery = 0

    def visit(vertex: int) -> None:
        nonlocal next_discovery
        discovery[vertex] = low[vertex] = next_discovery
        next_discovery += 1
        stack.append(vertex)
        on_stack[vertex] = True
        for target, _ in outgoing[vertex]:
            if discovery[target] == -1:
                visit(target)
                low[vertex] = min(low[vertex], low[target])
            elif on_stack[target]:
                low[vertex] = min(low[vertex], discovery[target])
        if low[vertex] != discovery[vertex]:
            return
        component: list[int] = []
        while True:
            member = stack.pop()
            on_stack[member] = False
            component.append(member)
            if member == vertex:
                break
        components.append(tuple(sorted(component)))

    for vertex in range(vertex_count):
        if discovery[vertex] == -1:
            visit(vertex)

    component_means: list[Fraction] = []
    for component in components:
        members = set(component)
        component_edges = tuple(
            (source, target, weight)
            for source in component
            for target, weight in outgoing[source]
            if target in members
        )
        if len(component) == 1 and not any(
            source == target for source, target, _ in component_edges
        ):
            continue

        local_index = {vertex: index for index, vertex in enumerate(component)}
        local_edges = tuple(
            (local_index[source], local_index[target], weight)
            for source, target, weight in component_edges
        )
        size = len(component)
        costs: list[list[int | None]] = [[None] * size for _ in range(size + 1)]
        costs[0][0] = 0
        for length in range(1, size + 1):
            previous = costs[length - 1]
            current = costs[length]
            for source, target, weight in local_edges:
                prefix = previous[source]
                if prefix is None:
                    continue
                candidate = prefix + weight
                if current[target] is None or candidate < current[target]:
                    current[target] = candidate

        vertex_bounds: list[Fraction] = []
        for vertex in range(size):
            final_cost = costs[size][vertex]
            if final_cost is None:
                continue
            bounds = tuple(
                Fraction(final_cost - earlier_cost, size - length)
                for length in range(size)
                if (earlier_cost := costs[length][vertex]) is not None
            )
            if bounds:
                vertex_bounds.append(max(bounds))
        if not vertex_bounds:
            raise AssertionError("a cyclic strongly connected component has a Karp bound")
        component_means.append(min(vertex_bounds))

    return min(component_means) if component_means else None
```

## Example

```python
def simple_cycle_oracle(
    vertex_count: int,
    edges: tuple[tuple[int, int, int], ...],
) -> Fraction | None:
    outgoing: list[list[tuple[int, int]]] = [[] for _ in range(vertex_count)]
    for source, target, weight in edges:
        outgoing[source].append((target, weight))
    means: list[Fraction] = []

    def extend(start: int, current: int, path: tuple[int, ...], total: int) -> None:
        for target, weight in outgoing[current]:
            if target == start:
                means.append(Fraction(total + weight, len(path)))
            elif target not in path:
                extend(start, target, (*path, target), total + weight)

    for start in range(vertex_count):
        extend(start, start, (start,), 0)
    return min(means) if means else None


sample_edges = (
    (0, 1, 3),
    (1, 2, -5),
    (2, 0, 1),
    (2, 2, 2),
    (3, 3, -2),
)
assert exact_minimum_directed_cycle_mean(4, sample_edges) == Fraction(-2)
assert exact_minimum_directed_cycle_mean(3, ((0, 1, -8), (1, 2, 1))) is None

rng = Random(0xCA_2F)
checked = 0
for _ in range(3_000):
    vertex_count = rng.randrange(1, 8)
    edges = tuple(
        (source, target, rng.randrange(-8, 9))
        for source in range(vertex_count)
        for target in range(vertex_count)
        if rng.randrange(4) == 0
    )
    expected = simple_cycle_oracle(vertex_count, edges)
    assert exact_minimum_directed_cycle_mean(vertex_count, edges) == expected
    assert (
        exact_minimum_directed_cycle_mean(
            vertex_count,
            tuple(reversed(edges)),
        )
        == expected
    )
    checked += 1

assert checked == 3_000
```

## Trade-offs and Limitations

Strongly connected components cost `O(V + E)` time. Karp's tables cost the sum
of `O(component_vertices * component_edges)` across cyclic components, at most
`O(VE)`, and retain `O(max_component_vertices² + V + E)` state. Integer path
costs and `Fraction` bounds preserve
exact comparisons; large Python integers can still increase constant costs.

The scalar mean is the complete result of this helper. It does not reconstruct
an attaining cycle, preserve parallel-edge identity, compute a maximum mean,
or accept floating weights. A minimum-mean cycle can differ from the shortest,
fewest-edge, or minimum-total-weight cycle, so those objectives are not
interchangeable.

## Related Snippets

<!-- catalog:related:start -->
- [Find the Canonical Shortest Cycle in a Bounded Directed Graph](find-the-canonical-shortest-cycle-in-a-bounded-directed-graph.md)
- [Compute Bounded Bellman-Ford Distances and Detect Reachable Negative Cycles](compute-bounded-bellman-ford-distances-and-detect-reachable-negative-cycles.md)
- [Partition a Bounded Directed Graph into Deterministic Strongly Connected Components](partition-a-bounded-directed-graph-into-deterministic-strongly-connected-components.md)
<!-- catalog:related:end -->
