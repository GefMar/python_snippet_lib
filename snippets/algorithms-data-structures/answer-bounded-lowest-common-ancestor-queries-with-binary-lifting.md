---
title: "Answer Bounded Lowest-Common-Ancestor Queries with Binary Lifting"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - repair-selected-hierarchy-parents-through-a-bounded-ancestor-map.md
  - build-a-canonical-min-cartesian-tree-parent-map-for-bounded-integers.md
  - traverse-a-parent-graph-with-breadth-first-search.md
---

# Answer Bounded Lowest-Common-Ancestor Queries with Binary Lifting

## Idea and Problem

Find the deepest shared ancestor of each requested vertex pair in one validated static rooted tree.

Following parent links separately for every query repeats the same climbs.
Binary lifting instead stores each vertex's ancestors at distances one, two,
four, and successive powers of two. A query first raises the deeper vertex to
the same depth, then tests powers from largest to smallest until both vertices
stand immediately below their lowest common ancestor.

The root is part of the input semantics: changing it can change every ancestor
relationship. Query answers retain declaration order, including repeated and
symmetrical pairs.

## When to Use

Use binary lifting when one complete rooted tree is static and a bounded batch
contains enough lowest-common-ancestor queries to justify preprocessing. The
parent tuple must cover the closed vertex range from zero through `n - 1`,
contain exactly one `None` root, and make every other vertex's parent chain
reach that root.

For a handful of shallow queries, direct parent climbing is simpler and uses
less memory. Use an Euler-tour range-minimum index for a different
preprocessing/query trade-off, and choose a dynamic-tree structure when parent
relationships can change between queries.

## Implementation

```python
from collections import deque

_MAX_LCA_VERTICES = 50_000
_MAX_LCA_QUERIES = 100_000
_UNREACHED_DEPTH = -1


def answer_lowest_common_ancestor_queries(
    parents: tuple[int | None, ...],
    queries: tuple[tuple[int, int], ...],
) -> tuple[int, ...]:
    """Return one lowest common ancestor for every query in declaration order."""
    if type(parents) is not tuple:
        raise TypeError("parents must be an exact tuple")
    if not 1 <= len(parents) <= _MAX_LCA_VERTICES:
        raise ValueError("parent count is outside the supported range")

    vertex_count = len(parents)
    root = -1
    root_count = 0
    direct_parents = [0] * vertex_count
    children: list[list[int]] = [[] for _ in range(vertex_count)]

    for vertex, parent in enumerate(parents):
        if parent is None:
            root = vertex
            root_count += 1
            direct_parents[vertex] = vertex
            continue
        if type(parent) is not int:
            raise TypeError(f"parents[{vertex}] must be an exact integer or None")
        if not 0 <= parent < vertex_count:
            raise ValueError(f"parents[{vertex}] is outside the rooted tree")
        if parent == vertex:
            raise ValueError(f"parents[{vertex}] must not refer to itself")
        direct_parents[vertex] = parent
        children[parent].append(vertex)

    if root_count != 1:
        raise ValueError("parents must contain exactly one None root")

    if type(queries) is not tuple:
        raise TypeError("queries must be an exact tuple")
    if len(queries) > _MAX_LCA_QUERIES:
        raise ValueError("query count exceeds the supported limit")
    for query_index, query in enumerate(queries):
        if type(query) is not tuple:
            raise TypeError(f"queries[{query_index}] must be an exact tuple")
        if len(query) != 2:
            raise ValueError(f"queries[{query_index}] must contain two vertices")
        first, second = query
        if type(first) is not int:
            raise TypeError(f"queries[{query_index}].first must be an exact integer")
        if type(second) is not int:
            raise TypeError(f"queries[{query_index}].second must be an exact integer")
        if not 0 <= first < vertex_count:
            raise ValueError(f"queries[{query_index}].first is outside the rooted tree")
        if not 0 <= second < vertex_count:
            raise ValueError(f"queries[{query_index}].second is outside the rooted tree")

    depths = [_UNREACHED_DEPTH] * vertex_count
    depths[root] = 0
    reached_count = 1
    pending: deque[int] = deque([root])
    while pending:
        vertex = pending.popleft()
        for child in children[vertex]:
            depths[child] = depths[vertex] + 1
            reached_count += 1
            pending.append(child)
    if reached_count != vertex_count:
        raise ValueError("every parent chain must reach the declared root")

    level_count = max(1, max(depths).bit_length())
    ancestors = [direct_parents]
    for _ in range(1, level_count):
        previous_level = ancestors[-1]
        ancestors.append([previous_level[previous_level[vertex]] for vertex in range(vertex_count)])

    answers: list[int] = []
    for first, second in queries:
        if depths[first] < depths[second]:
            first, second = second, first

        depth_gap = depths[first] - depths[second]
        level = 0
        while depth_gap:
            if depth_gap & 1:
                first = ancestors[level][first]
            depth_gap >>= 1
            level += 1

        if first == second:
            answers.append(first)
            continue

        for level in range(level_count - 1, -1, -1):
            first_ancestor = ancestors[level][first]
            second_ancestor = ancestors[level][second]
            if first_ancestor == second_ancestor:
                continue
            first = first_ancestor
            second = second_ancestor
        answers.append(direct_parents[first])

    return tuple(answers)
```

## Example

```python
def tree_from_prufer(
    code: tuple[int, ...],
) -> tuple[tuple[int, int], ...]:
    from heapq import heapify, heappop, heappush

    vertex_count = len(code) + 2
    degrees = [1] * vertex_count
    for vertex in code:
        degrees[vertex] += 1
    leaves = [vertex for vertex, degree in enumerate(degrees) if degree == 1]
    heapify(leaves)

    edges: list[tuple[int, int]] = []
    for neighbor in code:
        leaf = heappop(leaves)
        edges.append((leaf, neighbor))
        degrees[leaf] -= 1
        degrees[neighbor] -= 1
        if degrees[neighbor] == 1:
            heappush(leaves, neighbor)
    edges.append((heappop(leaves), heappop(leaves)))
    return tuple(edges)


def root_tree(
    vertex_count: int,
    edges: tuple[tuple[int, int], ...],
    root: int,
) -> tuple[int | None, ...]:
    adjacency: list[list[int]] = [[] for _ in range(vertex_count)]
    for first, second in edges:
        adjacency[first].append(second)
        adjacency[second].append(first)

    parents: list[int | None] = [None] * vertex_count
    visited = [False] * vertex_count
    visited[root] = True
    pending: deque[int] = deque([root])
    while pending:
        vertex = pending.popleft()
        for neighbor in adjacency[vertex]:
            if visited[neighbor]:
                continue
            visited[neighbor] = True
            parents[neighbor] = vertex
            pending.append(neighbor)
    return tuple(parents)


def naive_lowest_common_ancestor(
    parents: tuple[int | None, ...],
    first: int,
    second: int,
) -> int:
    first_ancestors: set[int] = set()
    current: int | None = first
    while current is not None:
        first_ancestors.add(current)
        current = parents[current]

    current = second
    while current not in first_ancestors:
        parent = parents[current]
        assert parent is not None
        current = parent
    return current


def exercise_small_rooted_trees() -> int:
    from itertools import product

    assert answer_lowest_common_ancestor_queries(
        (None,),
        ((0, 0),),
    ) == (0,)
    checked = 1

    for vertex_count in range(2, 7):
        for code in product(
            range(vertex_count),
            repeat=vertex_count - 2,
        ):
            edges = tree_from_prufer(code)
            for root in range(vertex_count):
                parents = root_tree(vertex_count, edges, root)
                queries = tuple(
                    (first, second)
                    for first in range(vertex_count)
                    for second in range(vertex_count)
                )
                expected = tuple(
                    naive_lowest_common_ancestor(parents, first, second)
                    for first, second in queries
                )
                assert (
                    answer_lowest_common_ancestor_queries(
                        parents,
                        queries,
                    )
                    == expected
                )
                checked += len(queries)
    return checked


ordinary_parents = (None, 0, 0, 1, 1, 2, 2)
ordinary_queries = ((3, 4), (3, 5), (5, 6), (4, 4), (6, 0))
ordinary_answers = answer_lowest_common_ancestor_queries(
    ordinary_parents,
    ordinary_queries,
)

maximum_chain = (
    None,
    *(vertex - 1 for vertex in range(1, _MAX_LCA_VERTICES)),
)
chain_answers = answer_lowest_common_ancestor_queries(
    maximum_chain,
    (
        (49_999, 25_000),
        (49_999, 49_998),
        (0, 49_999),
        (12_345, 12_345),
    ),
)

maximum_star = (None,) + (0,) * (_MAX_LCA_VERTICES - 1)
maximum_queries = ((49_999, 49_998),) * _MAX_LCA_QUERIES
star_answers = answer_lowest_common_ancestor_queries(
    maximum_star,
    maximum_queries,
)

value_errors = 0
for invalid_parents in (
    (),
    (None, None),
    (1, 0),
    (None, 1),
    (None, 2),
    (None, 2, 1),
):
    try:
        answer_lowest_common_ancestor_queries(invalid_parents, ())
    except ValueError:
        value_errors += 1

for parents, invalid_queries in (
    ((None,), ((0, 0),) * (_MAX_LCA_QUERIES + 1)),
    ((None,), ((0, 1),)),
    ((None,), ((0, 0, 0),)),
):
    try:
        answer_lowest_common_ancestor_queries(parents, invalid_queries)
    except ValueError:
        value_errors += 1

type_errors = 0
for invalid_parents, queries in (
    ([None], ()),
    ((None, False), ()),
):
    try:
        answer_lowest_common_ancestor_queries(invalid_parents, queries)
    except TypeError:
        type_errors += 1

for invalid_queries in (
    [(0, 0)],
    ([0, 0],),
    ((0, False),),
):
    try:
        answer_lowest_common_ancestor_queries((None,), invalid_queries)
    except TypeError:
        type_errors += 1

assert (
    exercise_small_rooted_trees(),
    ordinary_answers,
    chain_answers,
    len(star_answers),
    set(star_answers),
    value_errors,
    type_errors,
) == (
    296_675,
    (1, 0, 2, 4, 0),
    (25_000, 49_998, 0, 12_345),
    100_000,
    {0},
    9,
    5,
)
```

## Trade-offs and Limitations

Building depths and the binary-ancestor table takes `O(n log n)` time and
memory. Each query takes `O(log n)` time, and the returned tuple uses `O(q)`
memory. The complete batch therefore uses `O(n log n + q log n)` time and
`O(n log n + q)` memory.

The method spends considerably more memory than direct parent climbing. This
function also rebuilds the table for each call; a reusable index object is a
better interface when several query batches share one tree. The root and
integer labels are observable parts of the contract.

Only one complete, static rooted tree is accepted. The implementation does not
handle forests, multiple parents, directed acyclic graphs, changing parents,
path aggregates, distances, `k`-th ancestor queries, or online insertion. It
preserves duplicate and reversed query declarations instead of deduplicating
them.

## Related Snippets

<!-- catalog:related:start -->
- [Repair Selected Hierarchy Parents Through a Bounded Ancestor Map](repair-selected-hierarchy-parents-through-a-bounded-ancestor-map.md)
- [Build a Canonical Min-Cartesian-Tree Parent Map for Bounded Integers](build-a-canonical-min-cartesian-tree-parent-map-for-bounded-integers.md)
- [Traverse a Parent Graph with Breadth-First Search](traverse-a-parent-graph-with-breadth-first-search.md)
<!-- catalog:related:end -->
