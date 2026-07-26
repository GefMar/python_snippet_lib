---
title: "Elide Nested Mapping Values That Match Explicit Defaults"
snippet_type: recipe
use_cases:
  - configuration
  - serialization
tested_python:
  - "3.14"
dependencies: []
related:
  - merge-nested-mappings-without-mutating-inputs.md
  - merge-nested-configuration-with-an-explicit-delete-sentinel.md
  - prune-empty-values-from-json-like-data.md
---

# Elide Nested Mapping Values That Match Explicit Defaults

## Idea and Problem

Store only the type-sensitive differences between a complete JSON mapping and explicit defaults, then reconstruct an independent complete document from the same defaults.

Mapping keys equal to their defaults are omitted recursively. Arrays are
atomic values, so an unequal array is retained in full and its positions never
shift. Unknown keys and meaningful falsy values remain. Before compaction, the
function requires every default key to exist in the full document; otherwise
missing data could be silently reintroduced during expansion.

## When to Use

Use this representation when writers hold a complete defaults-expanded
configuration but persisted review files should show only overrides. Version
the compact document together with its default set, and expand it before
ordinary runtime access. Use a patch format with explicit deletion semantics
when a configuration may intentionally omit a defaulted key.

## Implementation

```python
import math


def _clone_json(value: object, active: set[int] | None = None) -> object:
    active = set() if active is None else active
    if value is None or type(value) in (bool, int, str):
        return value
    if type(value) is float:
        if not math.isfinite(value):
            raise ValueError("JSON numbers must be finite")
        return value
    if type(value) is list:
        identity = id(value)
        if identity in active:
            raise ValueError("cyclic lists are not valid JSON")
        active.add(identity)
        try:
            return [_clone_json(item, active) for item in value]
        finally:
            active.remove(identity)
    if type(value) is dict:
        identity = id(value)
        if identity in active:
            raise ValueError("cyclic mappings are not valid JSON")
        if any(type(key) is not str for key in value):
            raise TypeError("JSON object keys must be strings")
        active.add(identity)
        try:
            return {key: _clone_json(item, active) for key, item in value.items()}
        finally:
            active.remove(identity)
    raise TypeError(f"unsupported JSON value: {type(value).__name__}")


def _json_equal(left: object, right: object) -> bool:
    if type(left) is not type(right):
        return False
    if type(left) is dict:
        return left.keys() == right.keys() and all(
            _json_equal(left[key], right[key]) for key in left
        )
    if type(left) is list:
        return len(left) == len(right) and all(
            _json_equal(left_item, right_item)
            for left_item, right_item in zip(left, right, strict=True)
        )
    return left == right


def _require_complete(
    full: dict[str, object],
    defaults: dict[str, object],
    path: tuple[str, ...] = (),
) -> None:
    for key, default_value in defaults.items():
        child_path = path + (key,)
        if key not in full:
            raise ValueError(f"full document is missing default path {child_path!r}")
        full_value = full[key]
        if type(full_value) is dict and type(default_value) is dict:
            _require_complete(full_value, default_value, child_path)


def _elide(
    full: dict[str, object],
    defaults: dict[str, object],
) -> dict[str, object]:
    result: dict[str, object] = {}
    for key, full_value in full.items():
        if key not in defaults:
            result[key] = _clone_json(full_value)
            continue

        default_value = defaults[key]
        if _json_equal(full_value, default_value):
            continue
        if type(full_value) is dict and type(default_value) is dict:
            result[key] = _elide(full_value, default_value)
        else:
            result[key] = _clone_json(full_value)
    return result


def elide_json_defaults(
    full_document: dict[str, object],
    default_document: dict[str, object],
) -> dict[str, object]:
    if type(full_document) is not dict or type(default_document) is not dict:
        raise TypeError("full_document and default_document must be JSON objects")
    full = _clone_json(full_document)
    defaults = _clone_json(default_document)
    _require_complete(full, defaults)
    return _elide(full, defaults)


def _expand(
    overrides: dict[str, object],
    defaults: dict[str, object],
) -> dict[str, object]:
    result = {key: _clone_json(value) for key, value in defaults.items()}
    for key, override_value in overrides.items():
        default_value = defaults.get(key)
        if type(override_value) is dict and type(default_value) is dict:
            result[key] = _expand(override_value, default_value)
        else:
            result[key] = _clone_json(override_value)
    return result


def expand_json_defaults(
    compact_document: dict[str, object],
    default_document: dict[str, object],
) -> dict[str, object]:
    if type(compact_document) is not dict or type(default_document) is not dict:
        raise TypeError("compact_document and default_document must be JSON objects")
    compact = _clone_json(compact_document)
    defaults = _clone_json(default_document)
    return _expand(compact, defaults)
```

## Example

```python
defaults = {
    "enabled": True,
    "limits": {"soft": 10, "hard": 20},
    "labels": [],
    "mode": None,
}
full = {
    "enabled": True,
    "limits": {"soft": 10, "hard": 25},
    "labels": [],
    "mode": False,
    "extra": {"owner": "team"},
}
compact = elide_json_defaults(full, defaults)
expanded = expand_json_defaults(compact, defaults)
expanded["extra"]["owner"] = "changed"

try:
    elide_json_defaults(
        {"limits": {"hard": 25}},
        defaults,
    )
except ValueError:
    incomplete_rejected = True
else:
    incomplete_rejected = False

assert (
    compact,
    expand_json_defaults(compact, defaults),
    full["extra"],
    defaults["limits"],
    incomplete_rejected,
) == (
    {
        "limits": {"hard": 25},
        "mode": False,
        "extra": {"owner": "team"},
    },
    full,
    {"owner": "team"},
    {"soft": 10, "hard": 20},
    True,
)
```

## Trade-offs and Limitations

Compaction and expansion traverse and copy the JSON tree, using memory
proportional to the result. Arrays are intentionally all-or-nothing overrides;
large arrays with one change remain large. Equality is type-sensitive, so
`true`, `1`, and `1.0` are distinct despite Python's ordinary equality rules.
The compact form cannot represent deletion of a defaulted key, and applying it
to changed defaults can change the reconstructed document. Persist a defaults
version or migrate compact data whenever defaults evolve. Deeply nested input
can still hit Python's recursion limit.

## Related Snippets

<!-- catalog:related:start -->
- [Merge Nested Mappings Without Mutating Inputs](merge-nested-mappings-without-mutating-inputs.md)
- [Merge Nested Configuration with an Explicit Delete Sentinel](merge-nested-configuration-with-an-explicit-delete-sentinel.md)
- [Prune Empty Values from JSON-Like Data](prune-empty-values-from-json-like-data.md)
<!-- catalog:related:end -->
