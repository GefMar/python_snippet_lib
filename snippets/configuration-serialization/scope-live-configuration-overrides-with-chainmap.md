---
title: "Scope Live Configuration Overrides with ChainMap"
snippet_type: idiom
use_cases:
  - configuration
tested_python:
  - "3.14"
dependencies: []
related:
  - resolve-a-bounded-configuration-fragment-graph.md
  - resolve-bounded-configuration-through-dependent-adapters.md
---

# Scope Live Configuration Overrides with ChainMap

## Idea and Problem

Add one local layer of settings without copying the parent mappings or overwriting their values.

A chain of mappings reads from the first layer containing a key. Its child adds a fresh first mapping. Changes through the child write only into that first mapping, while changes to a shared parent are still visible for unshadowed keys.

## When to Use

Use this for process-local settings where scopes need live fallbacks and independently writable overrides. A setting explicitly set to None is present and shadows the parent.

## Implementation

```python
from collections import ChainMap


defaults = {"page_size": 20, "label": "standard"}
profile = {"page_size": 50}
parent = ChainMap(profile, defaults)
request = parent.new_child({"label": None})
```

## Example

```python
request["page_size"] = 5
defaults["tag"] = "preview"
during_override = (request["page_size"], request["label"], request["tag"])
del request["page_size"]
after_removal = (request["page_size"], parent["label"], profile["page_size"])

assert (during_override, after_removal) == ((5, None, "preview"), (50, "standard", 50))
```

## Trade-offs and Limitations

A ChainMap is live, not a snapshot or a deep merge. Deleting a name removes it from the first mapping only; it can reveal an inherited value instead of hiding it. Mutating a nested object can still affect the underlying mappings. For a stable shallow snapshot of resolved values, materialize a plain dict and define how nested values are owned.

## Related Snippets

<!-- catalog:related:start -->
- [Resolve a Bounded Configuration Fragment Graph](resolve-a-bounded-configuration-fragment-graph.md)
- [Resolve Bounded Configuration Through Dependent Adapters](resolve-bounded-configuration-through-dependent-adapters.md)
<!-- catalog:related:end -->
