---
title: "Recognize a Bounded Chordal Graph and Return a Deterministic Perfect Elimination Order"
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
  - compute-core-numbers-and-a-deterministic-degeneracy-order-for-a-bounded-undirected-graph.md
  - color-a-bounded-simple-undirected-graph-greedily-with-deterministic-dsatur.md
  - enumerate-bounded-maximal-cliques-deterministically-with-bron-kerbosch.md
---

# Recognize a Bounded Chordal Graph and Return a Deterministic Perfect Elimination Order

## Idea and Problem

Decide whether a simple undirected graph is chordal and retain a reproducible vertex order that certifies every successful result.

A graph is chordal exactly when it has a perfect elimination order: each
vertex's neighbors that occur later in the order form a clique. Maximum-
cardinality search repeatedly chooses the unnumbered vertex with the most
already selected neighbors. Reversing its selection order produces a candidate
PEO that succeeds exactly for chordal graphs.

Fixed smallest-vertex tie-breaking makes the candidate deterministic. On
failure, the first failed earliest-parent check returns its vertex, chosen
parent, and one nonadjacent later neighbor.

## When to Use

Use this recognizer before applying algorithms that rely on a chordal graph's
elimination structure, or when a small indexed graph needs deterministic
validation evidence. A successful order is independently checkable without
rerunning maximum-cardinality search.

Use a graph library for very large, mutable, directed, or multigraph inputs.
If diagnostics require an induced chordless cycle, run a separate witness-
reconstruction algorithm; a failed PEO triple is not itself such a cycle.

## Implementation

```python
from dataclasses import dataclass
from itertools import combinations, permutations
from random import Random

_MAX_CHORDAL_VERTICES = 256


@dataclass(frozen=True, slots=True)
class PeoViolation:
    vertex: int
    parent: int
    other_neighbor: int


@dataclass(frozen=True, slots=True)
class ChordalRecognition:
    candidate_order: tuple[int, ...]
    violation: PeoViolation | None


def recognize_chordal_graph(
    vertex_count: int,
    edges: tuple[tuple[int, int], ...],
) -> ChordalRecognition:
    """Return a deterministic MCS candidate and its first parent-check failure."""
    if type(vertex_count) is not int:
        raise TypeError("vertex_count must be an exact integer")
    if not 1 <= vertex_count <= _MAX_CHORDAL_VERTICES:
        raise ValueError("vertex_count is outside 1..256")
    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if len(edges) > vertex_count * (vertex_count - 1) // 2:
        raise ValueError("edge count exceeds a simple undirected graph")

    adjacency = [set() for _ in range(vertex_count)]
    seen: set[tuple[int, int]] = set()
    for edge_index, edge in enumerate(edges):
        if type(edge) is not tuple or len(edge) != 2:
            raise TypeError(f"edges[{edge_index}] must be an exact pair")
        left, right = edge
        if type(left) is not int or type(right) is not int:
            raise TypeError("edge endpoints must be exact integers")
        if not 0 <= left < right < vertex_count:
            raise ValueError("edges must use canonical endpoints 0 <= u < v < V")
        if edge in seen:
            raise ValueError("edges must be unique")
        seen.add(edge)
        adjacency[left].add(right)
        adjacency[right].add(left)

    scores = [0] * vertex_count
    selected = [False] * vertex_count
    selection_order: list[int] = []
    for _ in range(vertex_count):
        vertex = min(
            (candidate for candidate in range(vertex_count) if not selected[candidate]),
            key=lambda candidate: (-scores[candidate], candidate),
        )
        selected[vertex] = True
        selection_order.append(vertex)
        for neighbor in adjacency[vertex]:
            if not selected[neighbor]:
                scores[neighbor] += 1

    candidate_order = tuple(reversed(selection_order))
    position = [0] * vertex_count
    for index, vertex in enumerate(candidate_order):
        position[vertex] = index

    for index, vertex in enumerate(candidate_order):
        later_neighbors = tuple(
            neighbor for neighbor in adjacency[vertex] if position[neighbor] > index
        )
        if len(later_neighbors) < 2:
            continue
        parent = min(later_neighbors, key=position.__getitem__)
        for other_neighbor in candidate_order[index + 1 :]:
            if (
                other_neighbor != parent
                and other_neighbor in adjacency[vertex]
                and other_neighbor not in adjacency[parent]
            ):
                return ChordalRecognition(
                    candidate_order,
                    PeoViolation(vertex, parent, other_neighbor),
                )
    return ChordalRecognition(candidate_order, None)
```

## Example

```python
def is_perfect_elimination_order(
    vertex_count: int,
    edges: tuple[tuple[int, int], ...],
    order: tuple[int, ...],
) -> bool:
    edge_set = set(edges)

    def adjacent(left: int, right: int) -> bool:
        return (min(left, right), max(left, right)) in edge_set

    position = {vertex: index for index, vertex in enumerate(order)}
    for vertex in range(vertex_count):
        later = tuple(
            neighbor
            for neighbor in range(vertex_count)
            if adjacent(vertex, neighbor) and position[neighbor] > position[vertex]
        )
        if any(not adjacent(left, right) for left, right in combinations(later, 2)):
            return False
    return True


def has_any_peo(vertex_count: int, edges: tuple[tuple[int, int], ...]) -> bool:
    return any(
        is_perfect_elimination_order(vertex_count, edges, order)
        for order in permutations(range(vertex_count))
    )


tree_edges = ((0, 1), (1, 2), (1, 3))
tree_result = recognize_chordal_graph(4, tree_edges)
assert tree_result.violation is None
assert is_perfect_elimination_order(4, tree_edges, tree_result.candidate_order)
cycle_result = recognize_chordal_graph(4, ((0, 1), (0, 3), (1, 2), (2, 3)))
assert cycle_result.violation is not None

parent_check_edges = (
    (0, 2),
    (0, 3),
    (0, 4),
    (1, 2),
    (1, 3),
    (1, 4),
    (3, 4),
)
parent_check_result = recognize_chordal_graph(5, parent_check_edges)
assert parent_check_result.candidate_order == (4, 3, 1, 2, 0)
assert parent_check_result.violation == PeoViolation(3, 1, 0)

checked = 0
for vertex_count in range(1, 6):
    possible_edges = tuple(combinations(range(vertex_count), 2))
    for mask in range(1 << len(possible_edges)):
        edges = tuple(
            edge for edge_index, edge in enumerate(possible_edges) if mask >> edge_index & 1
        )
        result = recognize_chordal_graph(vertex_count, edges)
        expected = has_any_peo(vertex_count, edges)
        assert (result.violation is None) == expected
        if result.violation is None:
            assert is_perfect_elimination_order(vertex_count, edges, result.candidate_order)
        checked += 1

rng = Random(0xC0_DA)
for _ in range(1_000):
    vertex_count = rng.randrange(1, 8)
    edges = tuple(edge for edge in combinations(range(vertex_count), 2) if rng.randrange(2))
    result = recognize_chordal_graph(vertex_count, edges)
    assert (result.violation is None) == has_any_peo(vertex_count, edges)
    checked += 1

assert checked == 2_099
```

## Trade-offs and Limitations

The simple deterministic vertex selection scans all unnumbered vertices at
every step, taking `O(V² + E)` time. Adjacency sets, scores, and ordering state
use `O(V + E)` memory. Heap or bucket implementations can improve sparse-graph
constants but add stale-entry or bucket-order policy.

A successful candidate is a complete perfect-elimination-order certificate.
On failure, the triple records the first failure encountered by the standard
earliest-parent check; it need not identify the earliest vertex whose complete
later-neighbor set is not a clique. It proves that this MCS-derived candidate
violates the PEO condition, while the MCS theorem supplies the recognition
result. It is not an induced-hole witness. The helper does not construct tree
decompositions, color graphs, enumerate maximum cliques, add fill edges, handle
directed or parallel edges, or maintain a graph under updates.

## Related Snippets

<!-- catalog:related:start -->
- [Compute Core Numbers and a Deterministic Degeneracy Order for a Bounded Undirected Graph](compute-core-numbers-and-a-deterministic-degeneracy-order-for-a-bounded-undirected-graph.md)
- [Color a Bounded Simple Undirected Graph Greedily with Deterministic DSATUR](color-a-bounded-simple-undirected-graph-greedily-with-deterministic-dsatur.md)
- [Enumerate Bounded Maximal Cliques Deterministically with Bron-Kerbosch](enumerate-bounded-maximal-cliques-deterministically-with-bron-kerbosch.md)
<!-- catalog:related:end -->
