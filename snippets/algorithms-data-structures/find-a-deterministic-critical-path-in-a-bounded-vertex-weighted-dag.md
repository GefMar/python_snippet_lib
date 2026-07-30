---
title: "Find a Deterministic Critical Path in a Bounded Vertex-Weighted DAG"
snippet_type: algorithm
use_cases:
  - automation
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - count-exact-source-to-target-paths-in-a-bounded-dag.md
  - compute-the-transitive-reduction-of-a-bounded-directed-acyclic-graph.md
  - classify-bounded-dag-stages-as-ready-waiting-or-blocked.md
---

# Find a Deterministic Critical Path in a Bounded Vertex-Weighted DAG

## Idea and Problem

Find the maximum-total-weight directed path anywhere in a bounded acyclic dependency graph and resolve equal totals reproducibly.

Dynamic programming follows one deterministic topological order. For every
vertex it retains the greatest-weight path ending there; a tie keeps the
lexicographically smaller complete vertex tuple. Comparing all endpoints then
finds a global critical path even when the graph has several disconnected
components or several sources and sinks.

## When to Use

Use this algorithm for a small operation DAG whose non-negative integer vertex
weights represent declared durations, costs, or another additive quantity. It
is useful for bounded plan inspection, deterministic fixtures, and identifying
one chain that determines the largest serial total under unlimited parallel
capacity.

Use a scheduling or optimization system when resources, calendars, edge lags,
uncertain durations, deadlines, or alternative execution modes matter. This
function does not calculate CPM slack, enumerate every critical path, or choose
a path between caller-specified endpoints.

## Implementation

```python
from dataclasses import dataclass
from heapq import heapify, heappop, heappush

_MAX_INT64 = 2**63 - 1
_MAX_VERTICES = 128
_MAX_EDGES = 4_096


class DirectedCycleError(ValueError):
    pass


@dataclass(frozen=True, slots=True)
class CriticalPath:
    total_weight: int
    vertices: tuple[int, ...]


def find_critical_path(
    vertex_weights: tuple[int, ...],
    edges: tuple[tuple[int, int], ...],
) -> CriticalPath:
    if type(vertex_weights) is not tuple:
        raise TypeError("vertex_weights must be an exact tuple")
    if len(vertex_weights) > _MAX_VERTICES:
        raise ValueError("vertex count exceeds the supported limit")

    aggregate_weight = 0
    for index, weight in enumerate(vertex_weights):
        if type(weight) is not int:
            raise TypeError(f"vertex_weights[{index}] must be an exact integer")
        if not 0 <= weight <= _MAX_INT64:
            raise ValueError(f"vertex_weights[{index}] is outside 0..2**63-1")
        aggregate_weight += weight
        if aggregate_weight > _MAX_INT64:
            raise ValueError("aggregate vertex weight exceeds 2**63-1")

    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if len(edges) > _MAX_EDGES:
        raise ValueError("edge count exceeds the supported limit")

    vertex_count = len(vertex_weights)
    adjacency: list[list[int]] = [[] for _ in range(vertex_count)]
    indegree = [0] * vertex_count
    seen: set[tuple[int, int]] = set()
    for edge_index, edge in enumerate(edges):
        field = f"edges[{edge_index}]"
        if type(edge) is not tuple:
            raise TypeError(f"{field} must be an exact tuple")
        if len(edge) != 2:
            raise ValueError(f"{field} must contain two endpoints")
        source, target = edge
        if type(source) is not int or type(target) is not int:
            raise TypeError(f"{field} endpoints must be exact integers")
        if not 0 <= source < vertex_count or not 0 <= target < vertex_count:
            raise ValueError(f"{field} endpoint is outside the graph")
        if source == target:
            raise ValueError(f"{field} is a self-edge")
        if edge in seen:
            raise ValueError(f"{field} duplicates a directed edge")
        seen.add(edge)
        adjacency[source].append(target)
        indegree[target] += 1

    ready = [vertex for vertex, degree in enumerate(indegree) if degree == 0]
    heapify(ready)
    topological_order: list[int] = []
    while ready:
        vertex = heappop(ready)
        topological_order.append(vertex)
        for target in sorted(adjacency[vertex]):
            indegree[target] -= 1
            if indegree[target] == 0:
                heappush(ready, target)
    if len(topological_order) != vertex_count:
        raise DirectedCycleError("edges must form a directed acyclic graph")
    if vertex_count == 0:
        return CriticalPath(0, ())

    best_weights = list(vertex_weights)
    best_paths = [(vertex,) for vertex in range(vertex_count)]
    for source in topological_order:
        for target in sorted(adjacency[source]):
            candidate_weight = best_weights[source] + vertex_weights[target]
            candidate_path = (*best_paths[source], target)
            if candidate_weight > best_weights[target] or (
                candidate_weight == best_weights[target]
                and candidate_path < best_paths[target]
            ):
                best_weights[target] = candidate_weight
                best_paths[target] = candidate_path

    endpoint = min(
        range(vertex_count),
        key=lambda vertex: (-best_weights[vertex], best_paths[vertex]),
    )
    return CriticalPath(best_weights[endpoint], best_paths[endpoint])
```

## Example

```python
def enumerate_paths(
    vertex_count: int,
    edges: tuple[tuple[int, int], ...],
) -> tuple[tuple[int, ...], ...]:
    adjacency = [[] for _ in range(vertex_count)]
    for source, target in edges:
        adjacency[source].append(target)

    paths: list[tuple[int, ...]] = []
    pending = [(vertex,) for vertex in range(vertex_count)]
    while pending:
        path = pending.pop()
        paths.append(path)
        pending.extend((*path, target) for target in adjacency[path[-1]])
    return tuple(paths)


def brute_critical_path(
    weights: tuple[int, ...],
    edges: tuple[tuple[int, int], ...],
) -> CriticalPath:
    if not weights:
        return CriticalPath(0, ())
    paths = enumerate_paths(len(weights), edges)
    return min(
        (CriticalPath(sum(weights[vertex] for vertex in path), path) for path in paths),
        key=lambda result: (-result.total_weight, result.vertices),
    )


def check_tiny_dags() -> int:
    from itertools import combinations, product

    checked = 0
    for vertex_count in range(5):
        possible_edges = tuple(combinations(range(vertex_count), 2))
        for edge_mask in range(1 << len(possible_edges)):
            edges = tuple(
                edge
                for bit, edge in enumerate(possible_edges)
                if edge_mask & (1 << bit)
            )
            for weights in product(range(3), repeat=vertex_count):
                assert find_critical_path(weights, edges) == brute_critical_path(
                    weights,
                    edges,
                )
                checked += 1
    return checked


checked = check_tiny_dags()

sample = find_critical_path(
    (3, 5, 2, 4, 1),
    ((0, 2), (1, 2), (2, 3), (1, 4)),
)
try:
    find_critical_path((1, 1), ((0, 1), (1, 0)))
except DirectedCycleError:
    cycle_rejected = True
else:
    cycle_rejected = False

assert (
    checked == 5_422
    and sample == CriticalPath(11, (1, 2, 3))
    and find_critical_path((0, 0, 0), ((0, 1), (1, 2)))
    == CriticalPath(0, (0,))
    and find_critical_path((), ()) == CriticalPath(0, ())
    and cycle_rejected
)
```

## Trade-offs and Limitations

Validation and topological ordering take `O((V + E) log V)` time after sorted
neighbor traversal. Retaining complete path tuples makes each relaxation copy
and potentially compare up to `V` integers, so the dynamic program takes
`O(EV)` additional time and `O(V^2 + E)` memory in the declared bounded model.

Only vertex weights contribute to the total. The lexicographic tie rule is a
reproducibility policy, not a scheduling preference. A valid edge set with a
cycle raises instead of returning a partial answer, and a disconnected graph
is intentionally searched as one global candidate space.

## Related Snippets

<!-- catalog:related:start -->
- [Count Exact Source-to-Target Paths in a Bounded DAG](count-exact-source-to-target-paths-in-a-bounded-dag.md)
- [Compute the Transitive Reduction of a Bounded Directed Acyclic Graph](compute-the-transitive-reduction-of-a-bounded-directed-acyclic-graph.md)
- [Classify Bounded DAG Stages as Ready, Waiting, or Blocked](classify-bounded-dag-stages-as-ready-waiting-or-blocked.md)
<!-- catalog:related:end -->
