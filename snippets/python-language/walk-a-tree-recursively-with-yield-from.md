---
title: "Walk a Tree Recursively with yield from"
snippet_type: idiom
use_cases:
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - read-fixed-size-blocks-with-iter-sentinel.md
  - ../algorithms-data-structures/traverse-a-parent-graph-with-breadth-first-search.md
---

# Walk a Tree Recursively with yield from

## Idea and Problem

Delegate each recursive subtree to another generator to produce a lazy preorder traversal with little coordination code.

A recursive function can yield the current node and then use `yield from` for
each node's children. The caller receives one flat iterator even though the
implementation follows the nested structure of the tree.

## When to Use

Use this pattern for a finite, acyclic tree when preorder traversal is the
required order and each node has an iterable of child nodes. It fits nested
menus, syntax trees, and other structures where processing should happen before
descending into children. Use a graph traversal with a visited set when nodes
can be shared or cycles are possible.

## Implementation

```python
from collections.abc import Iterable, Iterator
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class TreeNode:
    label: str
    children: tuple["TreeNode", ...] = ()


def walk_preorder(roots: Iterable[TreeNode]) -> Iterator[TreeNode]:
    for node in roots:
        yield node
        yield from walk_preorder(node.children)
```

## Example

```python
tree = TreeNode(
    "root",
    (
        TreeNode("guide"),
        TreeNode("reference", (TreeNode("types"), TreeNode("errors"))),
    ),
)

labels = [node.label for node in walk_preorder((tree,))]

assert labels == ["root", "guide", "reference", "types", "errors"]
```

## Trade-offs and Limitations

The recursion depth grows with the height of the tree, so a very deep tree can
exceed Python's recursion limit. The function has no cycle detection: a cycle
would recurse indefinitely, and a shared node would be yielded once per path.
Preorder is fixed by the implementation and by the order of each `children`
tuple; use an explicit stack or queue when traversal order, pruning, depth
limits, or cycle handling must be configurable.

## Related Snippets

<!-- catalog:related:start -->
- [Read Fixed-Size Blocks with iter() and a Sentinel](read-fixed-size-blocks-with-iter-sentinel.md)
- [Traverse a Parent Graph with Breadth-First Search](../algorithms-data-structures/traverse-a-parent-graph-with-breadth-first-search.md)
<!-- catalog:related:end -->
