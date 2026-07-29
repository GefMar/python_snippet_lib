---
title: "Count Exact Source-to-Target Paths in a Bounded DAG"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-the-transitive-reduction-of-a-bounded-directed-acyclic-graph.md
  - resolve-stable-ordering-constraints-with-topological-sort.md
  - ../testing-tooling/enumerate-every-topological-order-of-a-tiny-dag-for-schedule-tests.md
---

# Count Exact Source-to-Target Paths in a Bounded DAG

## Idea and Problem

Count every directed path between two declared vertices in a bounded acyclic graph without enumerating the paths themselves.

A topological order places every predecessor before its successors. Starting
with one path at the source, forward dynamic programming can therefore add each
vertex's completed count to every outgoing neighbor exactly once. The source
counts as one empty path to itself; acyclicity prevents any additional path from
returning to it.

The complete graph is validated with Kahn's algorithm before counting. A cycle
in a disconnected or otherwise irrelevant region is still rejected instead of
being hidden by the selected source and target.

## When to Use

Use this algorithm for one fully materialized dependency or state-transition
DAG when only the exact number of routes matters. It is useful for measuring
bounded combinatorial alternatives, validating generated DAGs, or checking that
a transformation preserves a known route count.

Use path enumeration when the routes themselves are required, or a shortest-
or longest-path algorithm when path weights or lengths determine the answer.
Define a different finite-walk policy for cyclic graphs. Unrestricted
source-to-target walks are unbounded when a reachable cycle can also reach the
target.

## Implementation

```python
from collections import deque

_MAX_DAG_VERTICES = 10_000
_MAX_DAG_EDGES = 100_000


def count_dag_paths(
    vertex_count: int,
    edges: tuple[tuple[int, int], ...],
    *,
    source: int,
    target: int,
) -> int:
    """Return the exact number of directed source-to-target paths."""
    if type(vertex_count) is not int:
        raise TypeError("vertex_count must be an exact non-boolean integer")
    if type(source) is not int:
        raise TypeError("source must be an exact non-boolean integer")
    if type(target) is not int:
        raise TypeError("target must be an exact non-boolean integer")
    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")

    if not 1 <= vertex_count <= _MAX_DAG_VERTICES:
        raise ValueError("vertex_count is outside the supported range")
    if not 0 <= source < vertex_count:
        raise ValueError("source is outside the graph")
    if not 0 <= target < vertex_count:
        raise ValueError("target is outside the graph")
    if len(edges) > _MAX_DAG_EDGES:
        raise ValueError("edge count exceeds the supported limit")

    seen_edges: set[tuple[int, int]] = set()
    for edge_index, edge in enumerate(edges):
        if type(edge) is not tuple:
            raise TypeError(f"edges[{edge_index}] must be an exact tuple")
        if len(edge) != 2:
            raise ValueError(f"edges[{edge_index}] must contain two endpoints")

        edge_source, edge_target = edge
        if type(edge_source) is not int:
            raise TypeError(f"edges[{edge_index}].source must be an exact integer")
        if type(edge_target) is not int:
            raise TypeError(f"edges[{edge_index}].target must be an exact integer")
        if not 0 <= edge_source < vertex_count:
            raise ValueError(f"edges[{edge_index}].source is outside the graph")
        if not 0 <= edge_target < vertex_count:
            raise ValueError(f"edges[{edge_index}].target is outside the graph")
        if edge_source == edge_target:
            raise ValueError(f"edges[{edge_index}] must not be a self-loop")
        if edge in seen_edges:
            raise ValueError(f"edges[{edge_index}] duplicates a directed edge")
        seen_edges.add(edge)

    del seen_edges

    outgoing: list[list[int]] = [[] for _ in range(vertex_count)]
    indegrees = [0] * vertex_count
    for edge_source, edge_target in edges:
        outgoing[edge_source].append(edge_target)
        indegrees[edge_target] += 1

    ready = deque(vertex for vertex, degree in enumerate(indegrees) if degree == 0)
    topological_order: list[int] = []
    while ready:
        vertex = ready.popleft()
        topological_order.append(vertex)
        for successor in outgoing[vertex]:
            indegrees[successor] -= 1
            if indegrees[successor] == 0:
                ready.append(successor)

    if len(topological_order) != vertex_count:
        raise ValueError("edges must form an acyclic directed graph")

    path_counts = [0] * vertex_count
    path_counts[source] = 1
    for vertex in topological_order:
        for successor in outgoing[vertex]:
            path_counts[successor] += path_counts[vertex]
    return path_counts[target]
```

## Example

```python
def count_paths_by_memoized_search(
    vertex_count: int,
    edges: tuple[tuple[int, int], ...],
    source: int,
    target: int,
) -> int:
    from functools import cache

    outgoing = [[] for _ in range(vertex_count)]
    for edge_source, edge_target in edges:
        outgoing[edge_source].append(edge_target)

    @cache
    def visit(vertex: int) -> int:
        if vertex == target:
            return 1
        return sum(visit(successor) for successor in outgoing[vertex])

    return visit(source)


def exercise_small_ordered_dags() -> int:
    checked = 0
    for vertex_count in range(1, 6):
        possible_edges = tuple(
            (edge_source, edge_target)
            for edge_source in range(vertex_count)
            for edge_target in range(edge_source + 1, vertex_count)
        )
        for mask in range(1 << len(possible_edges)):
            edges = tuple(
                edge for edge_index, edge in enumerate(possible_edges) if mask & (1 << edge_index)
            )
            for source in range(vertex_count):
                for target in range(vertex_count):
                    assert count_dag_paths(
                        vertex_count,
                        edges,
                        source=source,
                        target=target,
                    ) == count_paths_by_memoized_search(
                        vertex_count,
                        edges,
                        source,
                        target,
                    )
                    checked += 1
    return checked


def raises(error_type: type[Exception], operation: object) -> bool:
    try:
        operation()  # type: ignore[operator]
    except error_type:
        return True
    return False


base_edges = ((0, 1), (0, 2), (1, 3), (2, 3), (3, 4))
relabel = (3, 0, 4, 1, 2)
relabeled_edges = tuple(
    (relabel[edge_source], relabel[edge_target])
    for edge_source, edge_target in reversed(base_edges)
)

fibonacci_edges = tuple(
    (vertex, successor)
    for vertex in range(_MAX_DAG_VERTICES)
    for successor in (vertex + 1, vertex + 2)
    if successor < _MAX_DAG_VERTICES
)
fibonacci_path_count = count_dag_paths(
    _MAX_DAG_VERTICES,
    fibonacci_edges,
    source=0,
    target=_MAX_DAG_VERTICES - 1,
)


def build_edge_cap_graph() -> tuple[tuple[int, int], ...]:
    from itertools import islice

    return tuple(
        islice(
            (
                (edge_source, edge_target)
                for edge_source in range(448)
                for edge_target in range(edge_source + 1, 448)
            ),
            _MAX_DAG_EDGES,
        )
    )


edge_cap_graph = build_edge_cap_graph()

assert (
    exercise_small_ordered_dags(),
    count_dag_paths(5, base_edges, source=0, target=4),
    count_dag_paths(5, relabeled_edges, source=relabel[0], target=relabel[4]),
    count_dag_paths(4, (), source=0, target=3),
    count_dag_paths(5, base_edges, source=2, target=2),
    fibonacci_path_count.bit_length(),
    len(edge_cap_graph),
    count_dag_paths(448, edge_cap_graph, source=447, target=0),
    raises(
        ValueError,
        lambda: count_dag_paths(5, ((0, 1), (2, 3), (3, 2)), source=0, target=0),
    ),
    raises(ValueError, lambda: count_dag_paths(2, ((0, 1), (0, 1)), source=0, target=1)),
    raises(ValueError, lambda: count_dag_paths(1, ((0, 0),), source=0, target=0)),
    raises(TypeError, lambda: count_dag_paths(2, ((0, True),), source=0, target=1)),
    raises(
        ValueError,
        lambda: count_dag_paths(
            448,
            (*edge_cap_graph, (447, 447)),
            source=0,
            target=1,
        ),
    ),
) == (
    26_705,
    2,
    2,
    0,
    1,
    6_942,
    100_000,
    0,
    True,
    True,
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

Complete validation, adjacency construction, Kahn traversal, and dynamic
programming take expected `O(V + E)` graph operations and additions. Duplicate
detection uses expected constant-cost set membership. The adjacency lists,
indegrees, topological order, and counts occupy `O(V + E)` auxiliary graph
slots.

The operation count does not make arithmetic unit-cost. A path count can need
`O(V)` bits in a simple DAG, and all stored counts can occupy `O(V^2)` bits in
the worst case. Adding and retaining those Python integers therefore costs more
than constant time and space as counts grow, even though the graph traversal is
linear.

Edge declaration order can change the internal topological order, but exact
addition makes the returned count invariant. Every cycle is rejected before
counting, including one outside the source-to-target region and one presented
when source equals target. The function does not enumerate paths, accept
parallel edges, weights or probabilities, count cyclic walks, return shortest
or longest paths, or maintain an incrementally changing graph.

## Related Snippets

<!-- catalog:related:start -->
- [Compute the Transitive Reduction of a Bounded Directed Acyclic Graph](compute-the-transitive-reduction-of-a-bounded-directed-acyclic-graph.md)
- [Resolve Stable Ordering Constraints with Topological Sort](resolve-stable-ordering-constraints-with-topological-sort.md)
- [Enumerate Every Topological Order of a Tiny DAG for Schedule Tests](../testing-tooling/enumerate-every-topological-order-of-a-tiny-dag-for-schedule-tests.md)
<!-- catalog:related:end -->
