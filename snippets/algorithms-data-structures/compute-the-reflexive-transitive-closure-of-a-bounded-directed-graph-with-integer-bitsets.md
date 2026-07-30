---
title: "Compute the Reflexive Transitive Closure of a Bounded Directed Graph with Integer Bitsets"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-bounded-all-pairs-shortest-distances-with-floyd-warshall.md
  - compute-the-transitive-reduction-of-a-bounded-directed-acyclic-graph.md
  - partition-a-bounded-directed-graph-into-deterministic-strongly-connected-components.md
---

# Compute the Reflexive Transitive Closure of a Bounded Directed Graph with Integer Bitsets

## Idea and Problem

Compute which indexed vertices can reach which others, including every vertex reaching itself by a path of length zero.

Represent each source row as one Python integer whose bit `j` records whether
vertex `j` is reachable. Warshall's recurrence then needs no inner destination
loop: whenever source `i` already reaches intermediate vertex `k`, one integer
OR adds everything currently reachable from `k` to row `i`.

## When to Use

Use this function when a small static directed graph needs repeated
reachability queries, such as validating dependency implications or building a
closure fixture. Query whether `source` reaches `target` with
`rows[source] & (1 << target) != 0` after computing the rows once.

The reflexive result deliberately includes each vertex's own bit even when the
input has no self-loop. That makes the relation describe paths of zero or more
edges and keeps isolated vertices distinguishable from absent vertices.

## Implementation

```python
_MAX_VERTEX_COUNT = 512
_MAX_EDGE_COUNT = 65_536


def reflexive_transitive_closure(
    vertex_count: int,
    edges: tuple[tuple[int, int], ...],
) -> tuple[int, ...]:
    """Return one reachability bit mask for every indexed source vertex."""
    if type(vertex_count) is not int:
        raise TypeError("vertex_count must be an exact integer")
    if not 0 <= vertex_count <= _MAX_VERTEX_COUNT:
        raise ValueError("vertex_count is outside 0..512")
    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if len(edges) > _MAX_EDGE_COUNT:
        raise ValueError("edge count exceeds 65536")

    rows = [1 << source for source in range(vertex_count)]
    for edge_index, edge in enumerate(edges):
        if type(edge) is not tuple:
            raise TypeError(f"edges[{edge_index}] must be an exact tuple")
        if len(edge) != 2:
            raise ValueError(f"edges[{edge_index}] must contain two endpoints")

        source, target = edge
        if type(source) is not int or type(target) is not int:
            raise TypeError(f"edges[{edge_index}] endpoints must be exact integers")
        if not 0 <= source < vertex_count or not 0 <= target < vertex_count:
            raise ValueError(f"edges[{edge_index}] endpoint is outside the graph")
        rows[source] |= 1 << target

    for intermediate in range(vertex_count):
        intermediate_bit = 1 << intermediate
        intermediate_row = rows[intermediate]
        for source in range(vertex_count):
            if rows[source] & intermediate_bit:
                rows[source] |= intermediate_row

    return tuple(rows)
```

## Example

```python
def traversal_oracle(
    vertex_count: int,
    edges: tuple[tuple[int, int], ...],
) -> tuple[int, ...]:
    from collections import deque

    adjacency = [[] for _ in range(vertex_count)]
    for source, target in edges:
        adjacency[source].append(target)

    answer = []
    for start in range(vertex_count):
        reached = 1 << start
        pending = deque([start])
        while pending:
            source = pending.popleft()
            for target in adjacency[source]:
                target_bit = 1 << target
                if not reached & target_bit:
                    reached |= target_bit
                    pending.append(target)
        answer.append(reached)
    return tuple(answer)


def exercise_all_three_vertex_graphs() -> int:
    possible_edges = tuple(
        (source, target) for source in range(3) for target in range(3)
    )
    checked = 0
    for edge_flags in range(1 << len(possible_edges)):
        edges = tuple(
            edge
            for edge_index, edge in enumerate(possible_edges)
            if edge_flags & (1 << edge_index)
        )
        assert reflexive_transitive_closure(3, edges) == traversal_oracle(3, edges)
        checked += 1
    return checked


cycle_with_duplicates = ((0, 1), (1, 2), (2, 0), (0, 1), (3, 3))
cycle_rows = reflexive_transitive_closure(4, cycle_with_duplicates)
reversed_rows = reflexive_transitive_closure(4, tuple(reversed(cycle_with_duplicates)))
isolated_boundary_rows = reflexive_transitive_closure(_MAX_VERTEX_COUNT, ())

invalid_endpoint_rejected = False
try:
    reflexive_transitive_closure(2, ((0, 2),))
except ValueError:
    invalid_endpoint_rejected = True

assert (
    exercise_all_three_vertex_graphs() == 512
    and reflexive_transitive_closure(0, ()) == ()
    and cycle_rows == (0b0111, 0b0111, 0b0111, 0b1000)
    and reversed_rows == cycle_rows
    and isolated_boundary_rows[0] == 1
    and isolated_boundary_rows[-1] == 1 << (_MAX_VERTEX_COUNT - 1)
    and invalid_endpoint_rejected
)
```

## Trade-offs and Limitations

Validation takes `O(E)` time for `E` declared edges. Warshall propagation makes
`O(V**2)` Python row checks and uses OR operands containing up to `V` bits.
Those integer operations are not constant time: a word-level model gives a
worst-case bound of `O(V**3 / W)` for machine word width `W`. The returned rows
occupy `O(V**2)` bits in the dense case, in addition to Python object overhead.

The output is deterministic and independent of edge order. Duplicate edges and
self-loops do not alter it, but they still count toward the edge cap. Integer
bit masks are compact for bounded dense closure; a traversal per query can be
more appropriate for a large sparse graph that receives few queries.

The function returns no path, predecessor, minimum-hop count, component
partition, or proof of why a bit is set. It does not update closure
incrementally, remove redundant edges, or compute a transitive reduction.

## Related Snippets

<!-- catalog:related:start -->
- [Compute Bounded All-Pairs Shortest Distances with Floyd-Warshall](compute-bounded-all-pairs-shortest-distances-with-floyd-warshall.md)
- [Compute the Transitive Reduction of a Bounded Directed Acyclic Graph](compute-the-transitive-reduction-of-a-bounded-directed-acyclic-graph.md)
- [Partition a Bounded Directed Graph into Deterministic Strongly Connected Components](partition-a-bounded-directed-graph-into-deterministic-strongly-connected-components.md)
<!-- catalog:related:end -->
