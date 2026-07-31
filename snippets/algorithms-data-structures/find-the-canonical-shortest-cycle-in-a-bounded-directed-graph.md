---
title: "Find the Canonical Shortest Cycle in a Bounded Directed Graph"
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
  - partition-a-bounded-directed-graph-into-deterministic-strongly-connected-components.md
  - compute-bounded-all-pairs-shortest-distances-with-floyd-warshall.md
  - decompose-one-bounded-functional-graph-orbit-into-prefix-and-cycle.md
---

# Find the Canonical Shortest Cycle in a Bounded Directed Graph

## Idea and Problem

Find a directed cycle with the fewest edges and make the result independent of adjacency-list order.

Vertices are the integer positions in an adjacency tuple. A cycle is returned
as an open tuple of distinct vertices: the closing edge from its last vertex
back to its first vertex is present but the first vertex is not repeated. A
self-loop is therefore the one-vertex cycle `(v,)`.

Run breadth-first search from every possible cycle start. Sorted outgoing
neighbors make first discovery choose the lexicographically smallest shortest
path from that start. Every incoming edge back to the start closes one such
path into a candidate cycle. Rotate each candidate to its lexicographically
smallest directed representation, then compare candidates first by edge count
and then by that tuple. Rotation never reverses the directed cycle.

## When to Use

Use this algorithm for one small, fully materialized directed graph when a
cycle witness must be globally shortest and ties require a stable canonical
answer. It fits bounded dependency analysis, state-machine inspection, and
reference fixtures whose result must not depend on neighbor ordering.

Use strongly connected components when every cyclic region matters rather
than one witness. Use an all-pairs shortest-path algorithm when distances
between every vertex pair are also required, and use a specialized graph
library for substantially larger or frequently changing graphs.

## Implementation

```python
"""Find one canonical shortest cycle in a bounded directed graph."""

from collections import deque

_MAX_VERTICES = 128
_MAX_EDGES = 4_096


def _validated_adjacency(
    adjacency: object,
) -> tuple[tuple[int, ...], ...]:
    if type(adjacency) is not tuple:
        raise TypeError("adjacency must be an exact tuple")
    if not 1 <= len(adjacency) <= _MAX_VERTICES:
        raise ValueError("vertex count is outside the supported range")

    vertex_count = len(adjacency)
    edge_count = 0
    ordered_rows: list[tuple[int, ...]] = []

    for source, raw_targets in enumerate(adjacency):
        if type(raw_targets) is not tuple:
            raise TypeError(f"adjacency[{source}] must be an exact tuple")

        seen_targets: set[int] = set()
        for target_index, target in enumerate(raw_targets):
            if type(target) is not int:
                raise TypeError(f"adjacency[{source}][{target_index}] must be an exact integer")
            if not 0 <= target < vertex_count:
                raise ValueError(f"adjacency[{source}][{target_index}] is outside the graph")
            if target in seen_targets:
                raise ValueError(f"adjacency[{source}] repeats target {target}")
            seen_targets.add(target)

        edge_count += len(seen_targets)
        if edge_count > _MAX_EDGES:
            raise ValueError("edge count exceeds the supported limit")
        ordered_rows.append(tuple(sorted(seen_targets)))

    return tuple(ordered_rows)


def _canonical_directed_rotation(cycle: tuple[int, ...]) -> tuple[int, ...]:
    smallest_position = cycle.index(min(cycle))
    return cycle[smallest_position:] + cycle[:smallest_position]


def _breadth_first_predecessors(
    graph: tuple[tuple[int, ...], ...],
    start: int,
) -> tuple[tuple[int, ...], tuple[int, ...]]:
    predecessors = [-1] * len(graph)
    distances = [-1] * len(graph)
    distances[start] = 0
    frontier = deque([start])

    while frontier:
        source = frontier.popleft()
        for target in graph[source]:
            if distances[target] != -1:
                continue
            distances[target] = distances[source] + 1
            predecessors[target] = source
            frontier.append(target)

    return tuple(predecessors), tuple(distances)


def canonical_shortest_directed_cycle(
    adjacency: tuple[tuple[int, ...], ...],
) -> tuple[int, ...] | None:
    """Return the shortest cycle under the fixed rotation tie-break."""
    graph = _validated_adjacency(adjacency)
    vertex_count = len(graph)

    incoming: list[list[int]] = [[] for _ in graph]
    for source, targets in enumerate(graph):
        for target in targets:
            incoming[target].append(source)

    best: tuple[int, ...] | None = None

    for start in range(vertex_count):
        predecessors, distances = _breadth_first_predecessors(graph, start)

        for end in incoming[start]:
            if end == start:
                candidate = (start,)
            elif distances[end] == -1:
                continue
            else:
                reversed_path = [end]
                while reversed_path[-1] != start:
                    reversed_path.append(predecessors[reversed_path[-1]])
                candidate = tuple(reversed(reversed_path))

            canonical = _canonical_directed_rotation(candidate)
            if best is None or (len(canonical), canonical) < (len(best), best):
                best = canonical

    return best
```

## Example

```python
def exhaustive_tiny_cycle_oracle(
    adjacency: tuple[tuple[int, ...], ...],
) -> tuple[int, ...] | None:
    """Enumerate every distinct-vertex directed cycle in one tiny graph."""
    from itertools import permutations

    vertex_count = len(adjacency)
    edges = {(source, target) for source, targets in enumerate(adjacency) for target in targets}

    for cycle_length in range(1, vertex_count + 1):
        candidates: set[tuple[int, ...]] = set()
        for cycle in permutations(range(vertex_count), cycle_length):
            if all(
                (cycle[index], cycle[(index + 1) % cycle_length]) in edges
                for index in range(cycle_length)
            ):
                candidates.add(
                    min(cycle[offset:] + cycle[:offset] for offset in range(cycle_length))
                )
        if candidates:
            return min(candidates)
    return None


equal_shortest_cycles = (
    (2, 1),
    (2, 0),
    (1, 0),
)
reordered_neighbors = tuple(tuple(reversed(targets)) for targets in equal_shortest_cycles)
directed_orientation = ((2,), (0,), (1,))
multiple_loops = ((1,), (2, 1), (2,))
acyclic = ((1,), (2,), ())

assert canonical_shortest_directed_cycle(equal_shortest_cycles) == (0, 1)
assert canonical_shortest_directed_cycle(reordered_neighbors) == (0, 1)
assert canonical_shortest_directed_cycle(directed_orientation) == (0, 2, 1)
assert canonical_shortest_directed_cycle(multiple_loops) == (1,)
assert canonical_shortest_directed_cycle(acyclic) is None

# Compare against exhaustive enumeration for every directed graph through
# three vertices, including graphs with self-loops.
_EXPECTED_EXHAUSTIVE_GRAPH_COUNT = 530
checked_graphs = 0
for tiny_vertex_count in range(1, 4):
    possible_edges = tuple(
        (source, target)
        for source in range(tiny_vertex_count)
        for target in range(tiny_vertex_count)
    )
    for edge_mask in range(1 << len(possible_edges)):
        mutable_rows: list[list[int]] = [[] for _ in range(tiny_vertex_count)]
        for edge_index, (source, target) in enumerate(possible_edges):
            if edge_mask & (1 << edge_index):
                mutable_rows[source].append(target)
        tiny_adjacency = tuple(tuple(reversed(targets)) for targets in mutable_rows)
        assert canonical_shortest_directed_cycle(tiny_adjacency) == exhaustive_tiny_cycle_oracle(
            tiny_adjacency
        )
        checked_graphs += 1

assert checked_graphs == _EXPECTED_EXHAUSTIVE_GRAPH_COUNT
```

## Trade-offs and Limitations

For `V` vertices and `E` edges, validation and neighbor sorting take
`O(V + E log E)` worst-case time. Running one breadth-first search per start
and reconstructing candidate paths takes `O(V * (V + E))` time. The normalized
adjacency, incoming edges, search state, and queues use `O(V + E)` memory;
candidate and result tuples use at most `O(V)` additional memory.

The graph contains between 1 and 128 vertices and at most 4,096 unique ordered
edges. Each adjacency row must contain distinct in-range integer targets.
Self-loops are accepted, parallel ordered pairs are rejected, and an acyclic
graph returns `None`. Sorting copied rows makes the result invariant to the
order of neighbors supplied by the caller.

The returned tuple has no repeated closing vertex. Its length is both its
number of vertices and its number of directed edges, including length one for
a self-loop. Canonicalization compares rotations of the directed order only;
it never treats the reversed order as the same cycle. The function does not
enumerate all shortest cycles, report strongly connected components, accept
weighted edges, update an existing graph, or preserve caller-owned adjacency
objects in its result.

## Related Snippets

<!-- catalog:related:start -->
- [Partition a Bounded Directed Graph into Deterministic Strongly Connected Components](partition-a-bounded-directed-graph-into-deterministic-strongly-connected-components.md)
- [Compute Bounded All-Pairs Shortest Distances with Floyd-Warshall](compute-bounded-all-pairs-shortest-distances-with-floyd-warshall.md)
- [Decompose One Bounded Functional-Graph Orbit into Prefix and Cycle](decompose-one-bounded-functional-graph-orbit-into-prefix-and-cycle.md)
<!-- catalog:related:end -->
