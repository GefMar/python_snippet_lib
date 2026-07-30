---
title: "Count Topological Orders of a Bounded Directed Graph with Subset DP"
snippet_type: algorithm
use_cases:
  - data-transformation
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - resolve-stable-ordering-constraints-with-topological-sort.md
  - ../testing-tooling/enumerate-every-topological-order-of-a-tiny-dag-for-schedule-tests.md
  - count-exact-source-to-target-paths-in-a-bounded-dag.md
---

# Count Topological Orders of a Bounded Directed Graph with Subset DP

## Idea and Problem

Count the exact labeled vertex orders that place every directed predecessor before its target without materializing those orders.

Represent a placement prefix by a bit mask. The number stored for that mask can
advance to any unplaced vertex whose predecessor mask is already contained in
the prefix. Different prefixes that reach the same subset are combined because
their legal continuations depend only on which vertices have been placed, not
on their earlier order.

The full-mask count is zero for any cycle, including a self-loop. The empty
graph on zero vertices has one topological order: the empty permutation.

## When to Use

Use this subset dynamic program for a small dependency graph when the number of
valid schedules matters but enumerating every schedule would produce too much
output. It is useful for exhaustive-test planning, measuring ordering freedom,
or checking a closed small instance against another scheduler.

Use an ordinary topological sort when only one valid order or a cycle decision
is needed. Enumeration is more appropriate when every order itself must be
visited. The exponential state table makes this function unsuitable above its
explicit 20-vertex cap even when the graph is sparse.

## Implementation

```python
_MAX_TOPOLOGICAL_VERTEX_COUNT = 20
_MAX_TOPOLOGICAL_EDGE_COUNT = 400


def count_topological_orders(
    vertex_count: int,
    edges: tuple[tuple[int, int], ...],
) -> int:
    """Return the exact number of labeled topological vertex permutations."""
    if type(vertex_count) is not int:
        raise TypeError("vertex_count must be an exact integer")
    if not 0 <= vertex_count <= _MAX_TOPOLOGICAL_VERTEX_COUNT:
        raise ValueError("vertex_count is outside 0..20")
    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if len(edges) > _MAX_TOPOLOGICAL_EDGE_COUNT:
        raise ValueError("edge count exceeds 400")

    predecessor_masks = [0] * vertex_count
    seen_edges: set[tuple[int, int]] = set()
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
        if edge in seen_edges:
            raise ValueError("edges must be unique")
        seen_edges.add(edge)
        predecessor_masks[target] |= 1 << source

    full_mask = (1 << vertex_count) - 1
    prefix_counts = [0] * (1 << vertex_count)
    prefix_counts[0] = 1

    for placed_mask, prefix_count in enumerate(prefix_counts):
        if not prefix_count:
            continue
        unplaced_mask = full_mask ^ placed_mask
        remaining = unplaced_mask
        while remaining:
            vertex_bit = remaining & -remaining
            vertex = vertex_bit.bit_length() - 1
            if not predecessor_masks[vertex] & unplaced_mask:
                prefix_counts[placed_mask | vertex_bit] += prefix_count
            remaining ^= vertex_bit

    return prefix_counts[full_mask]
```

## Example

```python
def count_topological_orders_by_permutation(
    vertex_count: int,
    edges: tuple[tuple[int, int], ...],
) -> int:
    from itertools import permutations

    valid_count = 0
    for order in permutations(range(vertex_count)):
        positions = [0] * vertex_count
        for position, vertex in enumerate(order):
            positions[vertex] = position
        if all(positions[source] < positions[target] for source, target in edges):
            valid_count += 1
    return valid_count


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
        assert count_topological_orders(3, edges) == (
            count_topological_orders_by_permutation(3, edges)
        )
        assert count_topological_orders(3, tuple(reversed(edges))) == (
            count_topological_orders_by_permutation(3, edges)
        )
        checked += 1
    return checked


chain = tuple((vertex, vertex + 1) for vertex in range(19))
complete_with_self_loops = tuple(
    (source, target) for source in range(20) for target in range(20)
)

duplicate_rejected = False
try:
    count_topological_orders(2, ((0, 1), (0, 1)))
except ValueError:
    duplicate_rejected = True

assert (
    exercise_all_three_vertex_graphs() == 512
    and count_topological_orders(0, ()) == 1
    and count_topological_orders(5, ()) == 120
    and count_topological_orders(20, chain) == 1
    and count_topological_orders(20, complete_with_self_loops) == 0
    and count_topological_orders(3, ((0, 1), (1, 2), (2, 0))) == 0
    and duplicate_rejected
)
```

## Trade-offs and Limitations

Validation takes `O(E)` time and stores `O(V + E)` predecessor and uniqueness
state. The dynamic program performs `O(V * 2**V)` availability checks in the
worst case and retains `2**V` Python integer references. Counts can approach
`V!`, so additions and stored integers are not constant-cost as their bit
lengths grow.

Vertices are distinct labels `0` through `V - 1`. Edge declaration order does
not affect the answer, but duplicate edge tuples are rejected so the input has
one canonical multiplicity convention. A self-loop is accepted as a real
cycle and therefore makes the count zero.

The function returns only a count. It does not enumerate orders, select one
witness, explain a cycle, count unlabeled partial orders, sample uniformly,
approximate a large instance, or update the graph incrementally.

## Related Snippets

<!-- catalog:related:start -->
- [Resolve Stable Ordering Constraints with Topological Sort](resolve-stable-ordering-constraints-with-topological-sort.md)
- [Enumerate Every Topological Order of a Tiny DAG for Schedule Tests](../testing-tooling/enumerate-every-topological-order-of-a-tiny-dag-for-schedule-tests.md)
- [Count Exact Source-to-Target Paths in a Bounded DAG](count-exact-source-to-target-paths-in-a-bounded-dag.md)
<!-- catalog:related:end -->
