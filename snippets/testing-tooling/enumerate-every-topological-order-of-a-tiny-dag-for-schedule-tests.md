---
title: "Enumerate Every Topological Order of a Tiny DAG for Schedule Tests"
snippet_type: testing-technique
use_cases:
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../algorithms-data-structures/resolve-stable-ordering-constraints-with-topological-sort.md
  - generate-a-deterministic-pairwise-covering-matrix-from-closed-string-parameters.md
---

# Enumerate Every Topological Order of a Tiny DAG for Schedule Tests

## Idea and Problem

Enumerate every valid execution order of a tiny dependency graph so schedule tests can expose hidden ordering assumptions.

A topological order may choose any currently available node. Backtracking over
those choices finds every complete order, while declaration-index iteration
makes the result sequence deterministic. Updating indegrees in place avoids
rechecking every precedence edge from scratch at each recursive prefix.

## When to Use

Use this technique for a fixed test fixture with no more than eight named steps
when code should behave correctly under every serial order permitted by its
dependencies. It is useful for testing planners, hooks, migrations, or staged
workflows that might accidentally rely on one incidental topological order.

The node declaration order is only a reproducible result-order policy; it does
not add precedence. Use an ordinary topological sort when one valid production
order is sufficient. Use sampling, pairwise coverage, or a domain-specific
concurrency model when exhaustive orders, durations, resources, or true
interleavings are part of the problem.

## Implementation

```python
_MAX_TOPOLOGICAL_NODES = 8
_MAX_NODE_CHARACTERS = 64
_MAX_TOTAL_NODE_CHARACTERS = 256


def enumerate_topological_orders(
    nodes: tuple[str, ...],
    edges: tuple[tuple[str, str], ...],
) -> tuple[tuple[str, ...], ...]:
    """Return every complete topological order in declaration-index order."""
    if type(nodes) is not tuple:
        raise TypeError("nodes must be an exact tuple")
    if len(nodes) > _MAX_TOPOLOGICAL_NODES:
        raise ValueError("node count exceeds the supported limit")

    positions: dict[str, int] = {}
    total_characters = 0
    for index, node in enumerate(nodes):
        if type(node) is not str:
            raise TypeError(f"nodes[{index}] must be an exact string")
        if not node:
            raise ValueError(f"nodes[{index}] must not be empty")
        if len(node) > _MAX_NODE_CHARACTERS:
            raise ValueError(f"nodes[{index}] exceeds the character limit")
        total_characters += len(node)
        if total_characters > _MAX_TOTAL_NODE_CHARACTERS:
            raise ValueError("aggregate node text exceeds the supported limit")
        if node in positions:
            raise ValueError(f"nodes[{index}] is duplicated")
        positions[node] = index

    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if len(edges) > len(nodes) * len(nodes):
        raise ValueError("edge count exceeds the possible unique endpoint pairs")

    seen_edges: set[tuple[str, str]] = set()
    for index, edge in enumerate(edges):
        if type(edge) is not tuple:
            raise TypeError(f"edges[{index}] must be an exact tuple")
        if len(edge) != 2:
            raise ValueError(f"edges[{index}] must contain two endpoints")
        before, after = edge
        if type(before) is not str:
            raise TypeError(f"edges[{index}].before must be an exact string")
        if type(after) is not str:
            raise TypeError(f"edges[{index}].after must be an exact string")
        if before not in positions or after not in positions:
            raise ValueError(f"edges[{index}] contains an undeclared endpoint")
        if edge in seen_edges:
            raise ValueError(f"edges[{index}] is duplicated")
        seen_edges.add(edge)

    outgoing: list[list[int]] = [[] for _ in nodes]
    indegrees = [0] * len(nodes)
    for before, after in edges:
        before_index = positions[before]
        after_index = positions[after]
        outgoing[before_index].append(after_index)
        indegrees[after_index] += 1

    used = [False] * len(nodes)
    prefix: list[int] = []
    orders: list[tuple[str, ...]] = []

    def visit() -> None:
        if len(prefix) == len(nodes):
            orders.append(tuple(nodes[index] for index in prefix))
            return

        for node_index in range(len(nodes)):
            if used[node_index] or indegrees[node_index] != 0:
                continue

            used[node_index] = True
            prefix.append(node_index)
            for dependent_index in outgoing[node_index]:
                indegrees[dependent_index] -= 1

            visit()

            for dependent_index in outgoing[node_index]:
                indegrees[dependent_index] += 1
            prefix.pop()
            used[node_index] = False

    visit()
    return tuple(orders)
```

## Example

```python
def topological_orders_by_permutation(
    nodes: tuple[str, ...],
    edges: tuple[tuple[str, str], ...],
) -> tuple[tuple[str, ...], ...]:
    from itertools import permutations

    valid: list[tuple[str, ...]] = []
    for order in permutations(nodes):
        ranks = {node: index for index, node in enumerate(order)}
        if all(ranks[before] < ranks[after] for before, after in edges):
            valid.append(order)
    return tuple(valid)


def exercise_every_tiny_graph() -> int:
    checked = 0
    for node_count in range(4):
        nodes = tuple(f"node-{index}" for index in range(node_count))
        possible_edges = tuple((before, after) for before in nodes for after in nodes)
        for mask in range(1 << len(possible_edges)):
            edges = tuple(
                edge for edge_index, edge in enumerate(possible_edges) if mask & (1 << edge_index)
            )
            assert enumerate_topological_orders(nodes, edges) == (
                topological_orders_by_permutation(nodes, edges)
            )
            checked += 1
    return checked


diamond_nodes = ("fetch", "lint", "test", "publish")
diamond_edges = (
    ("fetch", "lint"),
    ("fetch", "test"),
    ("lint", "publish"),
    ("test", "publish"),
)
diamond_orders = enumerate_topological_orders(diamond_nodes, diamond_edges)

maximum_nodes = tuple(f"step-{index}" for index in range(8))
maximum_orders = enumerate_topological_orders(maximum_nodes, ())
maximum_oracle = topological_orders_by_permutation(maximum_nodes, ())

type_error_count = 0
for invalid_nodes, invalid_edges in (
    ([], ()),
    (("a", 1), ()),
    (("a",), []),
    (("a", "b"), (["a", "b"],)),
    (("a", "b"), (("a", 1),)),
):
    try:
        enumerate_topological_orders(invalid_nodes, invalid_edges)
    except TypeError:
        type_error_count += 1

value_error_count = 0
for invalid_nodes, invalid_edges in (
    (tuple(f"n{index}" for index in range(9)), ()),
    (("a", "a"), ()),
    (("",), ()),
    (("x" * 65,), ()),
    (tuple(chr(97 + index) * 52 for index in range(5)), ()),
    (("a", "b"), (("a",),)),
    (("a",), (("a", "missing"),)),
    (("a", "b"), (("a", "b"), ("a", "b"))),
    (("a",), (("a", "a"), ("a", "a"))),
):
    try:
        enumerate_topological_orders(invalid_nodes, invalid_edges)
    except ValueError:
        value_error_count += 1

assert (
    diamond_orders,
    enumerate_topological_orders((), ()),
    enumerate_topological_orders(("a", "b"), (("a", "b"), ("b", "a"))),
    enumerate_topological_orders(("self",), (("self", "self"),)),
    exercise_every_tiny_graph(),
    len(maximum_orders),
    maximum_orders == maximum_oracle,
    maximum_orders[0],
    maximum_orders[-1],
    type_error_count,
    value_error_count,
) == (
    (
        ("fetch", "lint", "test", "publish"),
        ("fetch", "test", "lint", "publish"),
    ),
    ((),),
    (),
    (),
    531,
    40_320,
    True,
    maximum_nodes,
    maximum_nodes[::-1],
    5,
    9,
)
```

## Trade-offs and Limitations

Let `V` be the node count, `E` the edge count, `P` the number of recursive
prefixes visited, and `C` the number of complete orders. Validation takes
expected `O(V + E)` mapping and set work plus node-text scanning. Enumeration
uses conservative `O(1 + P * (V + E))` work, `O(V + E)` working memory, and
`O(1 + C * V)` output memory. Cyclic branches still contribute to `P` even
though they produce no complete order.

The unconstrained eight-node case returns 40,320 tuples, and all recursive
prefixes remain `O(V!)` under the fixed limit. The function deliberately
materializes every result; declaration-index ordering makes that result
reproducible but does not reduce its factorial size. A cyclic non-empty graph
has no valid order and returns an empty tuple, while an empty graph has the one
empty order `((),)`.

Node names are exact, normalization-sensitive Python strings. This technique
does not execute schedules, explore concurrent interleavings, return partial
orders or cycle witnesses, model durations or resources, sample the order
space, or scale beyond eight nodes.

## Related Snippets

<!-- catalog:related:start -->
- [Resolve Stable Ordering Constraints with Topological Sort](../algorithms-data-structures/resolve-stable-ordering-constraints-with-topological-sort.md)
- [Generate a Deterministic Pairwise-Covering Matrix from Closed String Parameters](generate-a-deterministic-pairwise-covering-matrix-from-closed-string-parameters.md)
<!-- catalog:related:end -->
