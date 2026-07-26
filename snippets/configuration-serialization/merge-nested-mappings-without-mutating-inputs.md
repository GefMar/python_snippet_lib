---
title: "Merge Nested Mappings Without Mutating Inputs"
snippet_type: recipe
use_cases:
  - configuration
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../storage-databases/build-and-apply-a-deterministic-mapping-delta.md
  - ../python-language/make-a-defensive-copy-at-a-mutable-input-boundary.md
---

# Merge Nested Mappings Without Mutating Inputs

## Idea and Problem

Recursively merge colliding mappings into new dictionary containers while letting every non-mapping override replace the corresponding base value.

Top-level `dict.update` replaces a nested mapping completely. A structural
merge can preserve unspecified nested keys, but it needs explicit collision
rules. This version recurses only when both values are mappings, clones all
mapping containers it traverses, and treats every other value as an indivisible
leaf.

## When to Use

Use this recipe for acyclic configuration-like mappings whose keys are strings
and whose nested objects follow the same merge rule. It works when lists,
tuples, scalars, and other leaves should be replaced whole. Use a schema-aware
merge when deletion markers, list-by-key behavior, conflict detection, or
different rules per field are required.

## Implementation

```python
from collections.abc import Mapping


def _require_string_keys(mapping: Mapping[object, object]) -> None:
    if any(not isinstance(key, str) for key in mapping):
        raise TypeError("mapping keys must be strings")


def _copy_mapping_containers(value: object) -> object:
    if not isinstance(value, Mapping):
        return value
    _require_string_keys(value)
    return {
        key: _copy_mapping_containers(child)
        for key, child in value.items()
    }


def merge_nested_mappings(
    base: Mapping[str, object],
    override: Mapping[str, object],
) -> dict[str, object]:
    _require_string_keys(base)
    _require_string_keys(override)

    merged = {
        key: _copy_mapping_containers(value)
        for key, value in base.items()
    }
    for key, override_value in override.items():
        if (
            key in base
            and isinstance(base[key], Mapping)
            and isinstance(override_value, Mapping)
        ):
            merged[key] = merge_nested_mappings(base[key], override_value)
        else:
            merged[key] = _copy_mapping_containers(override_value)
    return merged
```

## Example

```python
shared_ports = [8000, 8001]
replacement_modes = ["fast"]
base = {
    "service": {"host": "old", "ports": shared_ports},
    "mode": "safe",
    "replace": {"old": True},
}
override = {
    "service": {"host": "new"},
    "mode": {"level": 2},
    "replace": replacement_modes,
}

merged = merge_nested_mappings(base, override)
service = merged["service"]
service["host"] = "changed after merge"

assert (
    service["ports"] is shared_ports,
    merged["mode"],
    merged["replace"] is replacement_modes,
    base["service"],
    override["service"],
    merge_nested_mappings({}, {"nested": {"value": 1}}),
    merge_nested_mappings({"kept": 1}, {}),
) == (
    True,
    {"level": 2},
    True,
    {"host": "old", "ports": shared_ports},
    {"host": "new"},
    {"nested": {"value": 1}},
    {"kept": 1},
)
```

## Trade-offs and Limitations

Mapping containers become ordinary dictionaries, while non-mapping leaves are
shared by reference. Mutating a retained list or custom leaf can therefore
affect an input; copy those values under a domain-specific ownership policy.
Sequences are replaced rather than combined. Cyclic mappings recurse forever,
and very deep trees can exceed Python's recursion limit. The helper has no
deletion, schema validation, provenance, or conflict-resolution semantics.

## Related Snippets

<!-- catalog:related:start -->
- [Build and Apply a Deterministic Mapping Delta](../storage-databases/build-and-apply-a-deterministic-mapping-delta.md)
- [Make a Defensive Copy at a Mutable Input Boundary](../python-language/make-a-defensive-copy-at-a-mutable-input-boundary.md)
<!-- catalog:related:end -->
