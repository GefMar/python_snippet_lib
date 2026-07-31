---
title: "Select a Canonical Maximum-Weight Independent Set in a Bounded Forest"
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
  - find-a-deterministic-minimum-vertex-cover-in-a-bounded-bipartite-graph.md
  - build-a-deterministic-minimum-spanning-forest-with-kruskals-algorithm.md
  - solve-a-bounded-zero-one-knapsack-with-canonical-item-ties.md
---

# Select a Canonical Maximum-Weight Independent Set in a Bounded Forest

## Idea and Problem

Choose a maximum-weight set of nonadjacent forest vertices with one global, edge-order-independent tie rule.

Root every component by its smallest still-unseen vertex and process the forest
in iterative postorder. For each vertex, dynamic programming retains the best
solution when that vertex is included and when it is excluded. Including a
vertex forces every child into its excluded state; excluding it admits the
better state of each child.

An `n`-bit integer records the selected vertices, with vertex `0` in the most
significant position. Minimizing that integer on equal weight is exactly the
same as minimizing the fixed-width Boolean membership tuple in vertex-index
order with `False < True`. Masks from disjoint subtrees compose with bitwise OR,
so a local equal-weight decision remains valid in the complete forest.

## When to Use

Use this algorithm for a small static forest when vertices carry nonnegative
benefits and adjacent choices conflict. It fits bounded hierarchy, dependency,
and tree-shaped allocation snapshots where every vertex already has a stable
zero-based index and one reproducible optimum is required.

Use a simpler traversal when only maximality, rather than maximum total weight,
is needed. Use a specialized graph optimizer for general graphs, changing
topology, additional capacity constraints, or requests to enumerate every
optimal set.

## Implementation

```python
from dataclasses import dataclass

_MAX_FOREST_VERTICES = 256
_MAX_VERTEX_WEIGHT = (1 << 31) - 1
_UNSEEN_PARENT = -2

_ForestEdge = tuple[int, int]


@dataclass(frozen=True, slots=True)
class MaximumWeightIndependentSet:
    total_weight: int
    vertex_indexes: tuple[int, ...]


def _prefer_weight_then_mask(
    first_weight: int,
    first_mask: int,
    second_weight: int,
    second_mask: int,
) -> tuple[int, int]:
    if first_weight > second_weight:
        return first_weight, first_mask
    if second_weight > first_weight:
        return second_weight, second_mask
    if first_mask <= second_mask:
        return first_weight, first_mask
    return second_weight, second_mask


def canonical_maximum_weight_independent_set(
    weights: tuple[int, ...],
    edges: tuple[_ForestEdge, ...],
) -> MaximumWeightIndependentSet:
    """Return the fixed-membership-tied maximum-weight forest selection."""
    if type(weights) is not tuple:
        raise TypeError("weights must be an exact tuple")
    if not 1 <= len(weights) <= _MAX_FOREST_VERTICES:
        raise ValueError("vertex count is outside 1..256")
    for vertex, weight in enumerate(weights):
        if type(weight) is not int:
            raise TypeError(f"weights[{vertex}] must be an exact integer")
        if not 0 <= weight <= _MAX_VERTEX_WEIGHT:
            raise ValueError(f"weights[{vertex}] is outside the signed 32-bit range")

    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if len(edges) > len(weights) - 1:
        raise ValueError("a forest cannot contain more than n - 1 edges")

    adjacency: list[list[int]] = [[] for _ in weights]
    seen_pairs: set[_ForestEdge] = set()
    for edge_index, edge in enumerate(edges):
        if type(edge) is not tuple:
            raise TypeError(f"edges[{edge_index}] must be an exact tuple")
        if len(edge) != 2:
            raise ValueError(f"edges[{edge_index}] must contain two endpoints")
        first, second = edge
        if type(first) is not int or type(second) is not int:
            raise TypeError(f"edges[{edge_index}] endpoints must be exact integers")
        if not 0 <= first < len(weights) or not 0 <= second < len(weights):
            raise ValueError(f"edges[{edge_index}] has an out-of-range endpoint")
        if first == second:
            raise ValueError(f"edges[{edge_index}] is a self-loop")
        pair = (min(first, second), max(first, second))
        if pair in seen_pairs:
            raise ValueError(f"edges[{edge_index}] duplicates an undirected edge")
        seen_pairs.add(pair)
        adjacency[first].append(second)
        adjacency[second].append(first)

    for neighbors in adjacency:
        neighbors.sort()

    parents = [_UNSEEN_PARENT] * len(weights)
    roots: list[int] = []
    traversal: list[int] = []
    for root in range(len(weights)):
        if parents[root] != _UNSEEN_PARENT:
            continue
        roots.append(root)
        parents[root] = -1
        pending = [root]
        while pending:
            vertex = pending.pop()
            traversal.append(vertex)
            for neighbor in reversed(adjacency[vertex]):
                if neighbor == parents[vertex]:
                    continue
                if parents[neighbor] != _UNSEEN_PARENT:
                    raise ValueError("edges must form an acyclic undirected graph")
                parents[neighbor] = vertex
                pending.append(neighbor)

    included_weights = list(weights)
    included_masks = [1 << (len(weights) - 1 - vertex) for vertex in range(len(weights))]
    excluded_weights = [0] * len(weights)
    excluded_masks = [0] * len(weights)

    for vertex in reversed(traversal):
        included_weight = included_weights[vertex]
        included_mask = included_masks[vertex]
        excluded_weight = 0
        excluded_mask = 0
        for child in adjacency[vertex]:
            if parents[child] != vertex:
                continue
            included_weight += excluded_weights[child]
            included_mask |= excluded_masks[child]
            child_weight, child_mask = _prefer_weight_then_mask(
                included_weights[child],
                included_masks[child],
                excluded_weights[child],
                excluded_masks[child],
            )
            excluded_weight += child_weight
            excluded_mask |= child_mask
        included_weights[vertex] = included_weight
        included_masks[vertex] = included_mask
        excluded_weights[vertex] = excluded_weight
        excluded_masks[vertex] = excluded_mask

    total_weight = 0
    selected_mask = 0
    for root in roots:
        component_weight, component_mask = _prefer_weight_then_mask(
            included_weights[root],
            included_masks[root],
            excluded_weights[root],
            excluded_masks[root],
        )
        total_weight += component_weight
        selected_mask |= component_mask

    return MaximumWeightIndependentSet(
        total_weight=total_weight,
        vertex_indexes=tuple(
            vertex
            for vertex in range(len(weights))
            if selected_mask & (1 << (len(weights) - 1 - vertex))
        ),
    )
```

## Example

```python

def exhaustive_independent_set(
    weights: tuple[int, ...],
    edges: tuple[_ForestEdge, ...],
) -> MaximumWeightIndependentSet:
    best_weight = -1
    best_membership: tuple[bool, ...] | None = None
    best_indexes: tuple[int, ...] = ()
    for selected_mask in range(1 << len(weights)):
        if any(
            selected_mask & (1 << first) and selected_mask & (1 << second)
            for first, second in edges
        ):
            continue
        membership = tuple(bool(selected_mask & (1 << vertex)) for vertex in range(len(weights)))
        total_weight = sum(weight for vertex, weight in enumerate(weights) if membership[vertex])
        if total_weight > best_weight or (
            total_weight == best_weight
            and (best_membership is None or membership < best_membership)
        ):
            best_weight = total_weight
            best_membership = membership
            best_indexes = tuple(vertex for vertex, selected in enumerate(membership) if selected)
    return MaximumWeightIndependentSet(best_weight, best_indexes)


def edge_set_is_a_forest(
    vertex_count: int,
    edges: tuple[_ForestEdge, ...],
) -> bool:
    parents = list(range(vertex_count))

    def find(vertex: int) -> int:
        while parents[vertex] != vertex:
            vertex = parents[vertex]
        return vertex

    for first, second in edges:
        first_root = find(first)
        second_root = find(second)
        if first_root == second_root:
            return False
        parents[second_root] = first_root
    return True


def exercise_every_tiny_forest() -> int:
    from itertools import combinations, product

    checked = 0
    for vertex_count in range(1, 5):
        possible_edges = tuple(combinations(range(vertex_count), 2))
        for edge_flags in range(1 << len(possible_edges)):
            edges = tuple(
                edge
                for edge_index, edge in enumerate(possible_edges)
                if edge_flags & (1 << edge_index)
            )
            if not edge_set_is_a_forest(vertex_count, edges):
                continue
            for weights in product(range(3), repeat=vertex_count):
                assert canonical_maximum_weight_independent_set(
                    weights,
                    edges,
                ) == exhaustive_independent_set(weights, edges)
                checked += 1
    return checked


def exercise_seeded_forests() -> int:
    from random import Random

    random = Random(20_260_731)
    checked = 0
    for _ in range(400):
        vertex_count = random.randint(1, 9)
        weights = tuple(random.randint(0, 7) for _ in range(vertex_count))
        canonical_edges = tuple(
            (vertex, random.randrange(vertex))
            for vertex in range(1, vertex_count)
            if random.randrange(4) != 0
        )
        declared_edges = [
            (second, first) if random.randrange(2) else (first, second)
            for first, second in canonical_edges
        ]
        random.shuffle(declared_edges)
        edges = tuple(declared_edges)
        expected = exhaustive_independent_set(weights, edges)
        assert canonical_maximum_weight_independent_set(weights, edges) == expected
        assert (
            canonical_maximum_weight_independent_set(
                weights,
                tuple((second, first) for first, second in reversed(edges)),
            )
            == expected
        )
        checked += 1
    return checked


tie = canonical_maximum_weight_independent_set(
    (0, 5, 5, 0),
    ((1, 2),),
)
all_isolated = canonical_maximum_weight_independent_set((2, 1, 3), ())
all_zero = canonical_maximum_weight_independent_set((0, 0, 0), ())
boundary_weights = (_MAX_VERTEX_WEIGHT,) * _MAX_FOREST_VERTICES
boundary_edges = tuple((vertex, vertex + 1) for vertex in range(_MAX_FOREST_VERTICES - 1))
boundary = canonical_maximum_weight_independent_set(
    boundary_weights,
    boundary_edges,
)

rejected = 0
for invalid_weights, invalid_edges in (
    ([], ()),
    ((), ()),
    ((0,) * (_MAX_FOREST_VERTICES + 1), ()),
    ((True,), ()),
    ((-1,), ()),
    ((_MAX_VERTEX_WEIGHT + 1,), ()),
    ((0,), []),
    ((0, 0), ([0, 1],)),
    ((0, 0), ((0,),)),
    ((0, 0), ((0, True),)),
    ((0, 0), ((0, 2),)),
    ((0, 0), ((0, 0),)),
    ((0, 0, 0), ((0, 1), (0, 1))),
    ((0, 0, 0), ((0, 1), (1, 0))),
    ((1, 1, 1, 1), ((0, 1), (1, 2), (2, 0))),
    ((0, 0), ((0, 1), (0, 1))),
):
    try:
        canonical_maximum_weight_independent_set(
            invalid_weights,
            invalid_edges,
        )
    except (TypeError, ValueError):
        rejected += 1

assert (
    exercise_every_tiny_forest(),
    exercise_seeded_forests(),
    tie,
    all_isolated,
    all_zero,
    boundary.total_weight,
    boundary.vertex_indexes,
    rejected,
) == (
    3_288,
    400,
    MaximumWeightIndependentSet(5, (2,)),
    MaximumWeightIndependentSet(6, (0, 1, 2)),
    MaximumWeightIndependentSet(0, ()),
    128 * _MAX_VERTEX_WEIGHT,
    tuple(range(1, _MAX_FOREST_VERTICES, 2)),
    16,
)
```

## Trade-offs and Limitations

Validation, rooting, and numeric dynamic programming are linear after the
sorted adjacency lists are available; sorting costs `O(n log n)` because a
forest has fewer than `n` edges. Each stored mask has `n` bits, and mask OR and
comparison are not constant-cost with respect to that width. The honest worst
case is therefore `O(n^2)` bit work and `O(n^2)` retained mask bits within the
256-vertex limit.

The returned indexes are increasing. The equal-weight rule compares complete
fixed-width membership tuples, not variable-length index tuples: at the first
differing vertex, the solution that excludes it wins. Mapping vertex `0` to the
most significant mask bit makes ordinary integer comparison implement that
rule. As a consequence, a zero-weight vertex is never selected, and an
all-zero forest returns the empty set.

Endpoint orientation and edge declaration order do not affect the result, but
renumbering vertices can change equal-weight choices. Weight aggregation uses
exact Python integers. The function accepts only static simple forests with
nonnegative signed-32-bit vertex weights; it rejects cycles and parallel
undirected edges, and it does not solve maximum-weight independent set in a
general graph, support updates, or enumerate alternative optima.

## Related Snippets

<!-- catalog:related:start -->
- [Find a Deterministic Minimum Vertex Cover in a Bounded Bipartite Graph](find-a-deterministic-minimum-vertex-cover-in-a-bounded-bipartite-graph.md)
- [Build a Deterministic Minimum Spanning Forest with Kruskal's Algorithm](build-a-deterministic-minimum-spanning-forest-with-kruskals-algorithm.md)
- [Solve a Bounded Zero-One Knapsack with Canonical Item Ties](solve-a-bounded-zero-one-knapsack-with-canonical-item-ties.md)
<!-- catalog:related:end -->
