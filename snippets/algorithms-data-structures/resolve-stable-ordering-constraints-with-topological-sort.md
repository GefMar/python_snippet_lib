---
title: "Resolve Stable Ordering Constraints with Topological Sort"
snippet_type: algorithm
use_cases:
  - validation
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - traverse-a-parent-graph-with-breadth-first-search.md
  - ../storage-databases/build-and-apply-a-deterministic-mapping-delta.md
---

# Resolve Stable Ordering Constraints with Topological Sort

## Idea and Problem

Resolve partial before-and-after requirements into a deterministic total order while retaining original order wherever constraints leave a choice.

A precedence edge increases the later item's incoming count. Kahn's
topological-sort algorithm repeatedly chooses an available item, and a heap of
original positions provides a stable tie-breaker without inventing extra
dependencies.

## When to Use

Use this algorithm for finite plugin, pipeline, build-step, or registry names
whose configuration provides ordinary precedence constraints. Item names must
be unique and every constraint endpoint must be declared. Use a different
model when adjacency, groups, weights, or mutually exclusive choices are part
of correctness.

## Implementation

```python
import heapq
from collections.abc import Iterable
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class Precedence:
    before: str
    after: str


def stable_topological_order(
    items: Iterable[str],
    constraints: Iterable[Precedence],
) -> list[str]:
    ordered_items = list(items)
    positions = {}
    for index, item in enumerate(ordered_items):
        if not isinstance(item, str) or not item:
            raise ValueError("item names must be non-empty text")
        if item in positions:
            raise ValueError(f"duplicate item: {item}")
        positions[item] = index

    outgoing = {item: set() for item in ordered_items}
    incoming_count = {item: 0 for item in ordered_items}
    for constraint in constraints:
        if not isinstance(constraint, Precedence):
            raise TypeError("constraints must contain Precedence values")
        before = constraint.before
        after = constraint.after
        if not isinstance(before, str) or not before:
            raise ValueError("constraint endpoints must be non-empty text")
        if not isinstance(after, str) or not after:
            raise ValueError("constraint endpoints must be non-empty text")
        if before not in positions or after not in positions:
            raise ValueError(f"unknown constraint endpoint: {before!r} -> {after!r}")
        if before == after:
            raise ValueError(f"self precedence is not allowed: {before}")
        if after not in outgoing[before]:
            outgoing[before].add(after)
            incoming_count[after] += 1

    available = [
        positions[item]
        for item in ordered_items
        if incoming_count[item] == 0
    ]
    heapq.heapify(available)
    result = []

    while available:
        item = ordered_items[heapq.heappop(available)]
        result.append(item)
        for dependent in outgoing[item]:
            incoming_count[dependent] -= 1
            if incoming_count[dependent] == 0:
                heapq.heappush(available, positions[dependent])

    if len(result) != len(ordered_items):
        unresolved = [
            item for item in ordered_items if incoming_count[item] > 0
        ]
        raise ValueError(f"a precedence cycle prevents ordering: {unresolved}")
    return result
```

## Example

```python
items = ["notify", "store", "enrich", "validate", "parse"]
constraints = [
    Precedence("parse", "validate"),
    Precedence("parse", "enrich"),
    Precedence("validate", "store"),
    Precedence("store", "notify"),
    Precedence("parse", "validate"),
]
resolved = stable_topological_order(items, constraints)

try:
    stable_topological_order(
        ["left", "right"],
        [Precedence("left", "right"), Precedence("right", "left")],
    )
except ValueError:
    cycle_rejected = True
else:
    cycle_rejected = False

try:
    stable_topological_order(["known"], [Precedence("known", "missing")])
except ValueError:
    unknown_rejected = True
else:
    unknown_rejected = False

assert (
    resolved,
    stable_topological_order(["a", "b", "c"], []),
    cycle_rejected,
    unknown_rejected,
) == (
    ["parse", "enrich", "validate", "store", "notify"],
    ["a", "b", "c"],
    True,
    True,
)
```

## Trade-offs and Limitations

Runtime is `O((V + E) log V)` because each available item passes through the
position heap. Repeated edges are harmless, but the result reports only the
unresolved members rather than one minimal cycle path. Stability is a
tie-break policy, not an additional precedence promise: a newly available
earlier item may run before an item that was already available. Immediate
adjacency, force-last rules, grouping, and weighted priorities require richer
constraints instead of post-processing this result.

## Related Snippets

<!-- catalog:related:start -->
- [Traverse a Parent Graph with Breadth-First Search](traverse-a-parent-graph-with-breadth-first-search.md)
- [Build and Apply a Deterministic Mapping Delta](../storage-databases/build-and-apply-a-deterministic-mapping-delta.md)
<!-- catalog:related:end -->
