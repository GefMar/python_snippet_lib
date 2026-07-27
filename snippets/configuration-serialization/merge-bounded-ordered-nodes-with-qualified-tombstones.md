---
title: "Merge Bounded Ordered Nodes with Qualified Tombstones"
snippet_type: algorithm
use_cases:
  - configuration
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - merge-nested-configuration-with-an-explicit-delete-sentinel.md
  - merge-nested-mappings-without-mutating-inputs.md
  - parse-a-bounded-nested-bracket-tree.md
---

# Merge Bounded Ordered Nodes with Qualified Tombstones

## Idea and Problem

Merge two bounded immutable ordered node trees by qualified sibling identity while applying exact tombstones without changing either input.

Each identity is a `(name, qualifier)` pair, where the qualifier may be `None`.
An existing base node keeps its sibling position when it is replaced or merged,
while a new identity is appended in overlay order. This is not an ordinary
mapping merge: equal names with different qualifiers can coexist, and sibling
order is part of the result.

## When to Use

Use this algorithm when configuration has already been modeled as scalar and
branch nodes, overlays need explicit removal, and both producers agree that
qualified identity and stable order are meaningful. Roots and child collections
must be tuples. The base and overlay together may contain at most 128 nodes,
with nodes nested at most eight levels deep; every tree is validated completely
before merge construction begins.

## Implementation

```python
import math
import re
from dataclasses import dataclass


_NODE_NAME = re.compile(r"[a-z][a-z0-9-]{0,31}", re.ASCII)
_MAX_NODE_COUNT = 128
_MAX_NODE_DEPTH = 8

JsonScalar = str | int | float | bool | None


@dataclass(frozen=True, slots=True)
class ScalarNode:
    name: str
    qualifier: str | None
    value: JsonScalar


@dataclass(frozen=True, slots=True)
class Tombstone:
    name: str
    qualifier: str | None


@dataclass(frozen=True, slots=True)
class BranchNode:
    name: str
    qualifier: str | None
    children: tuple["ScalarNode | BranchNode | Tombstone", ...]


Node = ScalarNode | BranchNode
OverlayNode = ScalarNode | BranchNode | Tombstone


def _identity(node: OverlayNode) -> tuple[str, str | None]:
    return node.name, node.qualifier


def _validate_identity(node: OverlayNode) -> None:
    if type(node.name) is not str or _NODE_NAME.fullmatch(node.name) is None:
        raise ValueError("node name is invalid")
    qualifier = node.qualifier
    if qualifier is not None and (
        type(qualifier) is not str
        or _NODE_NAME.fullmatch(qualifier) is None
    ):
        raise ValueError("node qualifier is invalid")


def _validate_level(
    nodes: object,
    *,
    overlay: bool,
    depth: int,
    remaining: int,
) -> int:
    if type(nodes) is not tuple:
        raise TypeError("every node collection must be a tuple")

    identities: set[tuple[str, str | None]] = set()
    count = 0
    for node in nodes:
        if depth > _MAX_NODE_DEPTH:
            raise ValueError("tree exceeds the node depth limit")
        if type(node) not in (ScalarNode, BranchNode, Tombstone):
            raise TypeError("tree contains an unsupported node")
        if type(node) is Tombstone and not overlay:
            raise ValueError("the base tree cannot contain tombstones")

        _validate_identity(node)
        key = _identity(node)
        if key in identities:
            raise ValueError("sibling identities must be unique")
        identities.add(key)
        count += 1
        if count > remaining:
            raise ValueError("base and overlay exceed the combined node limit")

        if type(node) is ScalarNode:
            scalar_type = type(node.value)
            if scalar_type not in (str, int, float, bool, type(None)):
                raise TypeError("scalar value is not an immutable JSON scalar")
            if scalar_type is float and not math.isfinite(node.value):
                raise ValueError("floating-point scalar must be finite")
        elif type(node) is BranchNode:
            count += _validate_level(
                node.children,
                overlay=overlay,
                depth=depth + 1,
                remaining=remaining - count,
            )
    return count


def _validate_tombstone_targets(
    base: tuple[Node, ...],
    overlay: tuple[OverlayNode, ...],
) -> None:
    base_by_identity = {_identity(node): node for node in base}
    for patch in overlay:
        current = base_by_identity.get(_identity(patch))
        if type(patch) is Tombstone:
            if current is None:
                raise ValueError("tombstone targets a missing sibling")
        elif type(patch) is BranchNode:
            current_children = (
                current.children if type(current) is BranchNode else ()
            )
            _validate_tombstone_targets(current_children, patch.children)


def _copy_node(node: Node) -> Node:
    if type(node) is ScalarNode:
        return ScalarNode(node.name, node.qualifier, node.value)
    if type(node) is BranchNode:
        return BranchNode(
            node.name,
            node.qualifier,
            tuple(_copy_node(child) for child in node.children),
        )
    raise TypeError("cannot copy a tombstone into the result")


def _merge_level(
    base: tuple[Node, ...],
    overlay: tuple[OverlayNode, ...],
) -> tuple[Node, ...]:
    base_by_identity = {_identity(node): node for node in base}
    replacements: dict[tuple[str, str | None], Node] = {}
    removed: set[tuple[str, str | None]] = set()
    appended: list[Node] = []

    for patch in overlay:
        key = _identity(patch)
        current = base_by_identity.get(key)
        if type(patch) is Tombstone:
            removed.add(key)
        elif current is None:
            appended.append(_copy_node(patch))
        elif type(current) is BranchNode and type(patch) is BranchNode:
            replacements[key] = BranchNode(
                patch.name,
                patch.qualifier,
                _merge_level(current.children, patch.children),
            )
        else:
            replacements[key] = _copy_node(patch)

    merged: list[Node] = []
    for node in base:
        key = _identity(node)
        if key in removed:
            continue
        replacement = replacements.get(key)
        merged.append(_copy_node(node) if replacement is None else replacement)
    merged.extend(appended)
    return tuple(merged)


def merge_bounded_ordered_nodes(
    base: tuple[Node, ...],
    overlay: tuple[OverlayNode, ...],
) -> tuple[Node, ...]:
    base_count = _validate_level(
        base,
        overlay=False,
        depth=1,
        remaining=_MAX_NODE_COUNT,
    )
    _validate_level(
        overlay,
        overlay=True,
        depth=1,
        remaining=_MAX_NODE_COUNT - base_count,
    )
    _validate_tombstone_targets(base, overlay)
    return _merge_level(base, overlay)
```

## Example

```python
base = (
    BranchNode(
        "component",
        "live",
        (ScalarNode("weight", None, 1),),
    ),
    ScalarNode("component", "spare", "idle"),
)
overlay = (
    BranchNode(
        "component",
        "live",
        (
            ScalarNode("weight", None, 2),
            ScalarNode("enabled", None, True),
        ),
    ),
    Tombstone("component", "spare"),
    ScalarNode("component", "preview", "warming"),
)

merged = merge_bounded_ordered_nodes(base, overlay)

assert (
    merged,
    base[0].children,
    overlay[-1],
    merged[0] is base[0],
) == (
    (
        BranchNode(
            "component",
            "live",
            (
                ScalarNode("weight", None, 2),
                ScalarNode("enabled", None, True),
            ),
        ),
        ScalarNode("component", "preview", "warming"),
    ),
    (ScalarNode("weight", None, 1),),
    ScalarNode("component", "preview", "warming"),
    False,
)
```

## Trade-offs and Limitations

The deliberately small model accepts only lowercase ASCII names of at most 32
characters, optional qualifiers with the same grammar, tuple children, and
immutable JSON scalars (`str`, `int`, finite `float`, `bool`, or `None`). Scalar
strings are not size-bounded, so an outer input budget may still be necessary.
Every surviving node is copied; this makes the result independent but costs
linear time and space. A tombstone is exact rather than wildcarded, and a
tombstone below a new or scalar-replacing branch is rejected because no base
sibling exists there. Lookup, parsing, file I/O, schema-specific validation,
graph resolution, and publication of the merged configuration are out of scope.

## Related Snippets

<!-- catalog:related:start -->
- [Merge Nested Configuration with an Explicit Delete Sentinel](merge-nested-configuration-with-an-explicit-delete-sentinel.md)
- [Merge Nested Mappings Without Mutating Inputs](merge-nested-mappings-without-mutating-inputs.md)
- [Parse a Bounded Nested Bracket Tree](parse-a-bounded-nested-bracket-tree.md)
<!-- catalog:related:end -->
