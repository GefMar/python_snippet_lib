---
title: "Merge Nested Configuration with an Explicit Delete Sentinel"
snippet_type: recipe
use_cases:
  - configuration
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - merge-nested-mappings-without-mutating-inputs.md
  - ../python-language/apply-partial-dataclass-updates-with-an-omitted-value-sentinel.md
  - ../storage-databases/build-and-apply-a-deterministic-mapping-delta.md
---

# Merge Nested Configuration with an Explicit Delete Sentinel

## Idea and Problem

Merge nested configuration mappings while distinguishing deletion from ordinary values such as None, false, zero, and an empty mapping.

A unique sentinel marks a mapping entry for removal. Every override mapping is
traversed even when the base has no matching mapping, so nested deletion
markers are consumed instead of becoming output values. Mapping containers are
copied, while non-mapping leaves keep their original identity.

## When to Use

Use this recipe for acyclic, string-keyed configuration patches where an
omitted key means "leave unchanged" and an explicit marker means "delete".
It works when mappings merge recursively but sequences and other values should
replace a prior value whole. Keep the sentinel inside trusted Python code; use
a schema-defined wire representation when patches cross a serialization or
trust boundary.

## Implementation

```python
from collections.abc import Mapping


DELETE = object()


def _require_text_keys(mapping: Mapping[object, object]) -> None:
    if any(not isinstance(key, str) for key in mapping):
        raise TypeError("configuration keys must be strings")


def _clone_mapping_containers(value: object) -> object:
    if value is DELETE:
        raise ValueError("DELETE is not valid in the base configuration")
    if not isinstance(value, Mapping):
        return value
    _require_text_keys(value)
    return {
        key: _clone_mapping_containers(child)
        for key, child in value.items()
    }


def merge_config_with_delete(
    base: Mapping[str, object],
    override: Mapping[str, object],
) -> dict[str, object]:
    if not isinstance(base, Mapping) or not isinstance(override, Mapping):
        raise TypeError("base and override must be mappings")
    _require_text_keys(base)
    _require_text_keys(override)

    merged = {
        key: _clone_mapping_containers(value)
        for key, value in base.items()
    }
    for key, override_value in override.items():
        if override_value is DELETE:
            merged.pop(key, None)
        elif isinstance(override_value, Mapping):
            base_value = base.get(key)
            nested_base = base_value if isinstance(base_value, Mapping) else {}
            merged[key] = merge_config_with_delete(nested_base, override_value)
        else:
            merged[key] = override_value
    return merged
```

## Example

```python
shared_ports = [8000, 8001]
base = {
    "service": {
        "host": "old.example",
        "ports": shared_ports,
        "debug": True,
    },
    "replaced": {"legacy": True},
    "keep-none": None,
    "keep-zero": 0,
}
override = {
    "service": {"host": "new.example", "debug": DELETE},
    "replaced": False,
    "new-section": {"discard": DELETE, "enabled": True},
    "absent": DELETE,
    "keep-none": None,
}

merged = merge_config_with_delete(base, override)
merged_service = merged["service"]
merged_service["host"] = "changed.example"

assert (
    merged_service,
    merged_service["ports"] is shared_ports,
    merged["replaced"],
    merged["new-section"],
    merged["keep-none"],
    merged["keep-zero"],
    "absent" in merged,
    base["service"],
    override["service"]["debug"] is DELETE,
) == (
    {"host": "changed.example", "ports": shared_ports},
    True,
    False,
    {"enabled": True},
    None,
    0,
    False,
    {"host": "old.example", "ports": shared_ports, "debug": True},
    True,
)
```

## Trade-offs and Limitations

Only a value that is exactly `DELETE` and directly belongs to an override
mapping has deletion semantics; non-mapping leaves are opaque and shared by
reference. Mapping containers become ordinary dictionaries, and an absent
deletion is a no-op. A `DELETE` value anywhere in the base mapping hierarchy is
rejected, but callers must prevent the process-local sentinel from being
embedded inside an opaque leaf. Cyclic mappings recurse forever, and very deep
trees can exceed Python's recursion limit. This structural rule does not
perform schema validation, merge sequences, record provenance, or make
publication of a resulting configuration atomic.

## Related Snippets

<!-- catalog:related:start -->
- [Merge Nested Mappings Without Mutating Inputs](merge-nested-mappings-without-mutating-inputs.md)
- [Apply Partial Dataclass Updates with an Omitted-Value Sentinel](../python-language/apply-partial-dataclass-updates-with-an-omitted-value-sentinel.md)
- [Build and Apply a Deterministic Mapping Delta](../storage-databases/build-and-apply-a-deterministic-mapping-delta.md)
<!-- catalog:related:end -->
