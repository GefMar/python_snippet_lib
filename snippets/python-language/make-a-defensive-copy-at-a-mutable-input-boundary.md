---
title: "Make a Defensive Copy at a Mutable Input Boundary"
snippet_type: idiom
use_cases:
  - data-transformation
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - pass-constructor-only-context-with-initvar.md
---

# Make a Defensive Copy at a Mutable Input Boundary

## Idea and Problem

Copy a caller-owned mapping before retaining it so later top-level mutations cannot silently change the stored state.

Keeping a reference to mutable input creates shared ownership that is easy to
miss. An eager `dict` copy establishes a snapshot at the boundary, while
`MappingProxyType` prevents mutation through the returned top-level view.

## When to Use

Use a defensive snapshot when a function or object must retain mapping values
but should not observe later additions, removals, or replacements made by the
caller. This form is appropriate when a shallow copy is sufficient and the
mapping is small enough to copy eagerly. Define a deeper ownership policy when
values contain mutable objects that must also be isolated.

## Implementation

```python
from collections.abc import Mapping
from types import MappingProxyType
from typing import TypeVar


KeyT = TypeVar("KeyT")
ValueT = TypeVar("ValueT")


def read_only_snapshot(values: Mapping[KeyT, ValueT]) -> Mapping[KeyT, ValueT]:
    return MappingProxyType(dict(values))
```

## Example

```python
source = {"limit": 10, "labels": ["stable"]}
snapshot = read_only_snapshot(source)

source["limit"] = 25
source["labels"].append("shared")

assert (snapshot["limit"], snapshot["labels"]) == (10, ["stable", "shared"])
```

## Trade-offs and Limitations

Copying costs linear time and additional memory. This implementation is
deliberately shallow: replacing a top-level value in `source` is isolated, but
nested mutable values remain shared, as the example demonstrates. The proxy
also prevents top-level writes through the snapshot; return the copied `dict`
instead when the receiver should own and mutate its copy. Use an explicit deep
copy or immutable value model only when their extra cost and semantics are
appropriate.

## Related Snippets

<!-- catalog:related:start -->
- [Pass Constructor-Only Context with dataclasses.InitVar](pass-constructor-only-context-with-initvar.md)
<!-- catalog:related:end -->
