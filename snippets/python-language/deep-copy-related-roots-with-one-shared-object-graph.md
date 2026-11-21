---
title: "Deep Copy Related Roots with One Shared Object Graph"
snippet_type: idiom
use_cases:
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - make-a-defensive-copy-at-a-mutable-input-boundary.md
  - ../data-processing/reconcile-a-bounded-partial-record-under-explicit-field-policies.md
---

# Deep Copy Related Roots with One Shared Object Graph

## Idea and Problem

Copy related roots together so references shared between them stay shared in the copied graph.

The same deepcopy call remembers copies already made while traversing the container graph. It can preserve a shared mutable child and reconnect a cycle to the copied child, with no reference to that original child.

## When to Use

Use this when forking a trusted graph of built-in containers and shared identity is part of its structure. Put all related roots into the input for one copy call. Separate calls copy the same shared child separately.

## Implementation

```python
from copy import deepcopy


shared = {"labels": ["initial"]}
shared["self"] = shared
roots = {"input": [shared], "summary": [shared]}
fork = deepcopy(roots)
```

## Example

```python
copied_child = fork["input"][0]
copied_child["labels"].append("local")
original_child = roots["summary"][0]

assert (
    copied_child is fork["summary"][0],
    copied_child["self"] is copied_child,
    copied_child is not original_child,
    original_child["labels"],
    fork["summary"][0]["labels"],
) == (True, True, True, ["initial"], ["initial", "local"])
```

## Trade-offs and Limitations

Deep copies allocate a new mutable container graph and may be costly. Arbitrary classes can customize copy behavior, and objects such as open files cannot simply be copied this way. If only one shallow boundary needs ownership, copy at that boundary instead.

## Related Snippets

<!-- catalog:related:start -->
- [Make a Defensive Copy at a Mutable Input Boundary](make-a-defensive-copy-at-a-mutable-input-boundary.md)
- [Reconcile a Bounded Partial Record Under Explicit Field Policies](../data-processing/reconcile-a-bounded-partial-record-under-explicit-field-policies.md)
<!-- catalog:related:end -->
