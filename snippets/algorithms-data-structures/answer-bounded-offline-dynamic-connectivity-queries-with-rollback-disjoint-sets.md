---
title: "Answer Bounded Offline Dynamic Connectivity Queries with Rollback Disjoint Sets"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - maintain-bounded-disjoint-sets-with-rollback-snapshots.md
  - partition-a-bounded-undirected-graph-into-deterministic-components-with-union-find.md
  - find-articulation-points-and-bridges-in-a-bounded-undirected-graph.md
---

# Answer Bounded Offline Dynamic Connectivity Queries with Rollback Disjoint Sets

## Idea and Problem

Answer connectivity queries across a known sequence of undirected edge additions and removals without rebuilding the graph after every change.

Each edge is active over one or more half-open time intervals. A segment tree
over event indexes stores an interval in the few nodes whose ranges partition
it. A depth-first traversal applies those nodes' edges to a rollback disjoint-set
structure, answers queries at leaves, and then restores the state that existed
before entering each node.

This separates time from connectivity: the segment tree determines which edges
belong to a time, while rollback union-find maintains the corresponding
connected components. Query results retain declaration order.

## When to Use

Use this algorithm when the complete add, remove, and connectivity-query log is
available before any answer is needed. It is especially useful when removals
make ordinary forward-only union-find insufficient and repeatedly running a
graph search for every query would revisit too many edges.

Use adjacency-set breadth-first search for a small log, because it is easier to
read and supports immediate answers. Choose an online dynamic-connectivity
structure when events arrive incrementally and answers cannot wait for the
complete log. The event lifecycle here describes a simple graph: an edge must
be inactive before addition and active before removal.

## Implementation

```python
from dataclasses import dataclass

_MAX_DYNAMIC_VERTICES = 2_048
_MAX_DYNAMIC_EVENTS = 20_000


@dataclass(frozen=True, slots=True)
class AddEdge:
    first: int
    second: int


@dataclass(frozen=True, slots=True)
class RemoveEdge:
    first: int
    second: int


@dataclass(frozen=True, slots=True)
class ConnectedQuery:
    first: int
    second: int


type ConnectivityEvent = AddEdge | RemoveEdge | ConnectedQuery


def answer_offline_connectivity_queries(
    vertex_count: int,
    events: tuple[ConnectivityEvent, ...],
) -> tuple[bool, ...]:
    """Return query answers after applying every preceding graph event."""
    if type(vertex_count) is not int:
        raise TypeError("vertex_count must be an exact integer")
    if not 1 <= vertex_count <= _MAX_DYNAMIC_VERTICES:
        raise ValueError("vertex_count is outside 1..2048")
    if type(events) is not tuple:
        raise TypeError("events must be an exact tuple")
    if len(events) > _MAX_DYNAMIC_EVENTS:
        raise ValueError("event count exceeds the supported limit")

    active_starts: dict[tuple[int, int], int] = {}
    intervals: list[tuple[int, int, int, int]] = []

    def validated_vertex(vertex: object, label: str) -> int:
        if type(vertex) is not int:
            raise TypeError(f"{label} must be an exact integer")
        if not 0 <= vertex < vertex_count:
            raise ValueError(f"{label} is outside the graph")
        return vertex

    def normalized_edge(event: AddEdge | RemoveEdge, index: int) -> tuple[int, int]:
        first = validated_vertex(event.first, f"events[{index}].first")
        second = validated_vertex(event.second, f"events[{index}].second")
        if first == second:
            raise ValueError(f"events[{index}] must not contain a self-edge")
        return (first, second) if first < second else (second, first)

    for event_index, event in enumerate(events):
        if type(event) is AddEdge:
            edge = normalized_edge(event, event_index)
            if edge in active_starts:
                raise ValueError(f"events[{event_index}] adds an active edge")
            active_starts[edge] = event_index
        elif type(event) is RemoveEdge:
            edge = normalized_edge(event, event_index)
            start = active_starts.pop(edge, None)
            if start is None:
                raise ValueError(f"events[{event_index}] removes an inactive edge")
            intervals.append((start, event_index, edge[0], edge[1]))
        elif type(event) is ConnectedQuery:
            validated_vertex(event.first, f"events[{event_index}].first")
            validated_vertex(event.second, f"events[{event_index}].second")
        else:
            raise TypeError(f"events[{event_index}] has an unsupported exact type")

    event_count = len(events)
    for (first, second), start in active_starts.items():
        intervals.append((start, event_count, first, second))

    if event_count == 0:
        return ()

    leaf_count = 1
    while leaf_count < event_count:
        leaf_count <<= 1
    buckets: list[list[tuple[int, int]]] = [
        [] for _ in range(2 * leaf_count)
    ]
    for start, stop, first, second in intervals:
        left = start + leaf_count
        right = stop + leaf_count
        while left < right:
            if left & 1:
                buckets[left].append((first, second))
                left += 1
            if right & 1:
                right -= 1
                buckets[right].append((first, second))
            left >>= 1
            right >>= 1

    parents = list(range(vertex_count))
    sizes = [1] * vertex_count
    history: list[tuple[int, int, int]] = []

    def find(vertex: int) -> int:
        while parents[vertex] != vertex:
            vertex = parents[vertex]
        return vertex

    def union(first: int, second: int) -> None:
        first_root = find(first)
        second_root = find(second)
        if first_root == second_root:
            return
        if sizes[first_root] < sizes[second_root] or (
            sizes[first_root] == sizes[second_root]
            and first_root > second_root
        ):
            first_root, second_root = second_root, first_root
        history.append((second_root, first_root, sizes[first_root]))
        parents[second_root] = first_root
        sizes[first_root] += sizes[second_root]

    def rollback(depth: int) -> None:
        while len(history) > depth:
            child, parent, old_parent_size = history.pop()
            parents[child] = child
            sizes[parent] = old_parent_size

    answers: list[bool] = []

    def visit(node: int, start: int, stop: int) -> None:
        snapshot = len(history)
        for first, second in buckets[node]:
            union(first, second)

        if stop - start == 1:
            if start < event_count:
                event = events[start]
                if type(event) is ConnectedQuery:
                    answers.append(find(event.first) == find(event.second))
        else:
            middle = (start + stop) // 2
            visit(node * 2, start, middle)
            visit(node * 2 + 1, middle, stop)
        rollback(snapshot)

    visit(1, 0, leaf_count)
    return tuple(answers)
```

## Example

```python
def naive_connectivity_answers(
    vertex_count: int,
    events: tuple[ConnectivityEvent, ...],
) -> tuple[bool, ...]:
    from collections import deque

    active: set[tuple[int, int]] = set()
    answers: list[bool] = []
    for event in events:
        if type(event) in (AddEdge, RemoveEdge):
            assert event.first != event.second
            edge = tuple(sorted((event.first, event.second)))
            if type(event) is AddEdge:
                if edge in active:
                    raise ValueError
                active.add(edge)
            else:
                if edge not in active:
                    raise ValueError
                active.remove(edge)
            continue

        adjacency = [[] for _ in range(vertex_count)]
        for first, second in active:
            adjacency[first].append(second)
            adjacency[second].append(first)
        reached = {event.first}
        pending: deque[int] = deque([event.first])
        while pending:
            vertex = pending.popleft()
            for neighbor in adjacency[vertex]:
                if neighbor not in reached:
                    reached.add(neighbor)
                    pending.append(neighbor)
        answers.append(event.second in reached)
    return tuple(answers)


def exercise_small_logs() -> int:
    from itertools import product

    edges = ((0, 1), (0, 2), (1, 2))
    alphabet: tuple[ConnectivityEvent, ...] = tuple(
        AddEdge(*edge) for edge in edges
    ) + tuple(RemoveEdge(*edge) for edge in edges) + tuple(
        ConnectedQuery(first, second)
        for first in range(3)
        for second in range(3)
    )

    checked = 0
    for length in range(4):
        for events in product(alphabet, repeat=length):
            try:
                expected = naive_connectivity_answers(3, events)
            except ValueError:
                continue
            assert answer_offline_connectivity_queries(3, events) == expected
            checked += 1
    return checked


checked_logs = exercise_small_logs()

scenario = (
    AddEdge(0, 1),
    ConnectedQuery(0, 2),
    AddEdge(1, 2),
    ConnectedQuery(0, 2),
    RemoveEdge(1, 2),
    ConnectedQuery(0, 2),
    AddEdge(2, 1),
    ConnectedQuery(2, 0),
    ConnectedQuery(3, 3),
)
boundary_events = tuple(
    ConnectedQuery(0, _MAX_DYNAMIC_VERTICES - 1)
    for _ in range(_MAX_DYNAMIC_EVENTS)
)

invalid_add_rejected = False
try:
    answer_offline_connectivity_queries(
        2,
        (AddEdge(0, 1), AddEdge(1, 0)),
    )
except ValueError:
    invalid_add_rejected = True

invalid_remove_rejected = False
try:
    answer_offline_connectivity_queries(2, (RemoveEdge(0, 1),))
except ValueError:
    invalid_remove_rejected = True

assert (
    checked_logs == 1_885
    and answer_offline_connectivity_queries(4, scenario)
    == (False, True, False, True, True)
    and answer_offline_connectivity_queries(
        _MAX_DYNAMIC_VERTICES,
        boundary_events,
    )
    == (False,) * _MAX_DYNAMIC_EVENTS
    and invalid_add_rejected
    and invalid_remove_rejected
)
```

## Trade-offs and Limitations

For `T` events, `A` active intervals, `Q` queries, and `V` vertices, interval
construction takes `O(T)` time. Each interval enters `O(log T)` segment-tree
nodes. Union by size without path compression gives `O(log V)` worst-case
finds, for total `O(T + (A log T + Q) log V)` time. Buckets, the event tree,
answers, active-edge map, and rollback structure use
`O(T + A log T + V + Q)` memory.

The complete log and interval buckets must fit in memory before traversal.
Parallel-edge multiplicity is deliberately unsupported, and malformed edge
lifecycles fail instead of being normalized. Remaining active edges last to
the end of the log; queries observe all preceding events, not simultaneous
batch semantics.

This returns only connectivity booleans. It does not report component sizes,
paths, or edge witnesses, and it does not support directed edges, dynamic
vertices, online answers, persistence, or concurrent mutation.

## Related Snippets

<!-- catalog:related:start -->
- [Maintain Bounded Disjoint Sets with Rollback Snapshots](maintain-bounded-disjoint-sets-with-rollback-snapshots.md)
- [Partition a Bounded Undirected Graph into Deterministic Components with Union-Find](partition-a-bounded-undirected-graph-into-deterministic-components-with-union-find.md)
- [Find Articulation Points and Bridges in a Bounded Undirected Graph](find-articulation-points-and-bridges-in-a-bounded-undirected-graph.md)
<!-- catalog:related:end -->
