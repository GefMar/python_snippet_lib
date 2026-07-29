---
title: "Apply a Bounded RFC 7396 JSON Merge Patch without Mutating Inputs"
snippet_type: pattern
use_cases:
  - configuration
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - merge-nested-configuration-with-an-explicit-delete-sentinel.md
  - normalize-a-bounded-json-copy-before-standard-schema-validation.md
  - parse-bounded-utf-8-json-without-duplicate-object-names-or-non-finite-numbers.md
---

# Apply a Bounded RFC 7396 JSON Merge Patch without Mutating Inputs

## Idea and Problem

Apply the object-merge and replacement rules of RFC 7396 to already parsed JSON trees while returning a fully detached result.

An object patch recursively updates an object target, and a `null` object
member deletes the corresponding target member. Any non-object patch,
including an array or top-level `null`, replaces the complete target value.
Arrays are therefore atomic values rather than element-wise patch documents.

Both input trees are validated before any result is constructed. Explicit
depth, node, and UTF-8 text budgets reject oversized graphs, cycles, shared
mutable containers, non-finite numbers, and values outside the closed model.

## When to Use

Use this helper after strict JSON parsing when an API or configuration boundary
has explicitly selected [JSON Merge Patch](https://www.rfc-editor.org/rfc/rfc7396.html)
semantics. It works well for small documents where object-member updates and
deletions are sufficient and a detached result prevents accidental mutation of
either input.

Use a different protocol when individual array elements must be addressed,
operations need ordering or preconditions, or deletion must be distinguishable
from assigning JSON `null` to an object member.

## Implementation

```python
from math import isfinite

type JsonValue = (
    None
    | bool
    | int
    | float
    | str
    | list[JsonValue]
    | dict[str, JsonValue]
)

_MAX_DEPTH = 32
_MAX_NODES = 5_000
_MAX_TEXT_BYTES = 1 << 20
_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1


def _validate_json_tree(name: str, root: object) -> None:
    nodes = 0
    text_bytes = 0
    seen_containers: set[int] = set()

    def claim_text(kind: str, value: str) -> None:
        nonlocal text_bytes

        try:
            encoded_length = len(value.encode("utf-8"))
        except UnicodeEncodeError as error:
            raise ValueError(f"{name} {kind} contains a Unicode surrogate") from error
        text_bytes += encoded_length
        if text_bytes > _MAX_TEXT_BYTES:
            raise ValueError(f"{name} text exceeds the supported UTF-8 byte limit")

    def visit(value: object, depth: int) -> None:
        nonlocal nodes

        if depth > _MAX_DEPTH:
            raise ValueError(f"{name} exceeds the supported depth")
        nodes += 1
        if nodes > _MAX_NODES:
            raise ValueError(f"{name} exceeds the supported node count")

        if value is None or type(value) is bool:
            return
        if type(value) is int:
            if not _MIN_INT64 <= value <= _MAX_INT64:
                raise ValueError(f"{name} integer is outside the int64 range")
            return
        if type(value) is float:
            if not isfinite(value):
                raise ValueError(f"{name} float must be finite")
            return
        if type(value) is str:
            claim_text("string", value)
            return
        if type(value) is list:
            identity = id(value)
            if identity in seen_containers:
                raise ValueError(f"{name} must be a tree without shared containers")
            seen_containers.add(identity)
            for item in value:
                visit(item, depth + 1)
            return
        if type(value) is dict:
            identity = id(value)
            if identity in seen_containers:
                raise ValueError(f"{name} must be a tree without shared containers")
            seen_containers.add(identity)
            for key, item in value.items():
                if type(key) is not str:
                    raise TypeError(f"{name} object keys must be exact strings")
                claim_text("object key", key)
                visit(item, depth + 1)
            return
        raise TypeError(f"{name} contains a value outside the closed JSON model")

    visit(root, 1)


def _clone_json(value: JsonValue) -> JsonValue:
    if type(value) is list:
        return [_clone_json(item) for item in value]
    if type(value) is dict:
        return {key: _clone_json(item) for key, item in value.items()}
    return value


def _apply_merge_patch(target: JsonValue, patch: JsonValue) -> JsonValue:
    if type(patch) is not dict:
        return _clone_json(patch)

    target_object = target if type(target) is dict else {}
    result: dict[str, JsonValue] = {}
    for key, target_value in target_object.items():
        if key not in patch:
            result[key] = _clone_json(target_value)
        elif patch[key] is not None:
            result[key] = _apply_merge_patch(target_value, patch[key])

    for key, patch_value in patch.items():
        if key in target_object or patch_value is None:
            continue
        result[key] = _apply_merge_patch(None, patch_value)
    return result


def apply_json_merge_patch(target: JsonValue, patch: JsonValue) -> JsonValue:
    """Validate both bounded trees, then return a detached RFC 7396 result."""
    _validate_json_tree("target", target)
    _validate_json_tree("patch", patch)
    return _apply_merge_patch(target, patch)
```

## Example

```python
target = {
    "title": "Draft",
    "author": {"name": "Ada", "active": True},
    "tags": ["python", "json"],
}
patch = {
    "title": "Published",
    "author": {"active": None, "role": "editor"},
    "tags": ["reference"],
}

merged = apply_json_merge_patch(target, patch)

assert merged == {
    "title": "Published",
    "author": {"name": "Ada", "role": "editor"},
    "tags": ["reference"],
}
assert target["author"] == {"name": "Ada", "active": True}
assert patch["author"] == {"active": None, "role": "editor"}
assert merged is not target
assert merged["author"] is not target["author"]
assert merged["tags"] is not patch["tags"]

assert apply_json_merge_patch({"kept": 1}, [2, 3]) == [2, 3]
assert apply_json_merge_patch([1, 2], {"added": 3}) == {"added": 3}
assert apply_json_merge_patch({"removed": 1}, None) is None
```

## Trade-offs and Limitations

Validation visits `T + P` input nodes and merging constructs a result of `R`
nodes, for `O(T + P + R)` work and space apart from bounded recursion. The
implementation preserves the insertion order of surviving target members;
new members are appended in patch iteration order. It does not canonicalize
JSON text or sort object names.

The accepted model is deliberately narrower than arbitrary Python data: exact
containers only, exact string keys, signed 64-bit integers, finite exact
floats, surrogate-free strings, and no shared container identities. Each input
tree receives its own depth, node, and text budget.

This function does not parse or serialize JSON, implement JSON Patch, validate
an application schema, authorize fields, sign a patch, detect conflicts, apply
version preconditions, provide transactional storage, or coordinate concurrent
writers.

## Related Snippets

<!-- catalog:related:start -->
- [Merge Nested Configuration with an Explicit Delete Sentinel](merge-nested-configuration-with-an-explicit-delete-sentinel.md)
- [Normalize a Bounded JSON Copy Before Standard Schema Validation](normalize-a-bounded-json-copy-before-standard-schema-validation.md)
- [Parse Bounded UTF-8 JSON Without Duplicate Object Names or Non-Finite Numbers](parse-bounded-utf-8-json-without-duplicate-object-names-or-non-finite-numbers.md)
<!-- catalog:related:end -->
