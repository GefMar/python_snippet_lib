---
title: "Partition Tagged Items into Minimum Stable Conflict-Free Groups"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../data-processing/group-items-by-an-exact-compatibility-signature-and-report-unmatched-inputs.md
  - choose-the-first-eligible-candidate-from-ordered-priority-groups.md
  - resolve-stable-ordering-constraints-with-topological-sort.md
---

# Partition Tagged Items into Minimum Stable Conflict-Free Groups

## Idea and Problem

Partition tagged items so that no group repeats a tag while using the fewest possible groups and preserving stable order.

When one group may contain at most one item of each tag, the first occurrence
of every tag can share group zero, the second occurrence can share group one,
and so on. This occurrence-index construction keeps every item exactly once.
Its group count is the largest multiplicity of any tag, which is also a lower
bound and therefore proves that the partition is minimal.

## When to Use

Use this algorithm when the complete bounded input is available, item names are
unique, and the conflict rule depends on one exact tag. It fits deterministic
test partitions, display rows, or work waves where duplicate tags cannot coexist
but items with different tags are compatible.

Use a graph-coloring or constraint solver when conflicts depend on pairs of
items, capacities, weights, several attributes, or external state. This simple
proof applies only to the one-tag uniqueness rule.

## Implementation

```python
import re
from dataclasses import dataclass


_MAX_ITEMS = 256
_TOKEN = re.compile(r"[a-z][a-z0-9_-]{0,63}", re.ASCII)


@dataclass(frozen=True, slots=True)
class TaggedItem:
    name: str
    tag: str


def partition_tagged_items(
    items: tuple[TaggedItem, ...],
) -> tuple[tuple[TaggedItem, ...], ...]:
    if type(items) is not tuple:
        raise TypeError("items must be an exact tuple")
    if len(items) > _MAX_ITEMS:
        raise ValueError("item limit exceeded")

    names: set[str] = set()
    for item in items:
        if type(item) is not TaggedItem:
            raise TypeError("items must contain exact TaggedItem records")
        if type(item.name) is not str or type(item.tag) is not str:
            raise TypeError("item names and tags must be exact strings")
        if _TOKEN.fullmatch(item.name) is None or _TOKEN.fullmatch(item.tag) is None:
            raise ValueError("item name or tag is invalid")
        if item.name in names:
            raise ValueError("item names must be unique")
        names.add(item.name)

    occurrences: dict[str, int] = {}
    groups: list[list[TaggedItem]] = []
    for item in items:
        group_index = occurrences.get(item.tag, 0)
        occurrences[item.tag] = group_index + 1
        if group_index == len(groups):
            groups.append([])
        groups[group_index].append(item)

    return tuple(tuple(group) for group in groups)
```

## Example

```python
items = (
    TaggedItem("north", "blue"),
    TaggedItem("south", "green"),
    TaggedItem("east", "blue"),
    TaggedItem("west", "green"),
    TaggedItem("center", "blue"),
)

groups = partition_tagged_items(items)

assert groups == (
    (TaggedItem("north", "blue"), TaggedItem("south", "green")),
    (TaggedItem("east", "blue"), TaggedItem("west", "green")),
    (TaggedItem("center", "blue"),),
)
```

## Trade-offs and Limitations

The function performs expected `O(n)` work and uses `O(n + t)` additional
memory for at most 256 items and `t` distinct tags. It validates the complete
input before constructing groups, preserves input order inside every group,
and silently discards nothing. The output is immutable, but it contains the
original immutable item records.

If a tag occurs `k` times, those `k` items require at least `k` different
groups. The algorithm creates exactly one group per occurrence index, so it
uses `max(k)` groups and is minimal. It does not balance group sizes, optimize
cost, dispatch on runtime types, resolve dependencies, create resources, or
check compatibility beyond exact tag equality.

## Related Snippets

<!-- catalog:related:start -->
- [Group Items by an Exact Compatibility Signature and Report Unmatched Inputs](../data-processing/group-items-by-an-exact-compatibility-signature-and-report-unmatched-inputs.md)
- [Choose the First Eligible Candidate from Ordered Priority Groups](choose-the-first-eligible-candidate-from-ordered-priority-groups.md)
- [Resolve Stable Ordering Constraints with Topological Sort](resolve-stable-ordering-constraints-with-topological-sort.md)
<!-- catalog:related:end -->
