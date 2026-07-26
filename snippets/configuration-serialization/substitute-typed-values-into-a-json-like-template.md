---
title: "Substitute Typed Values into a JSON-Like Template"
snippet_type: recipe
use_cases:
  - configuration
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - prune-empty-values-from-json-like-data.md
  - merge-nested-configuration-with-an-explicit-delete-sentinel.md
  - ../python-language/load-text-templates-from-package-resources.md
---

# Substitute Typed Values into a JSON-Like Template

## Idea and Problem

Replace exact structural placeholders with already typed JSON-compatible values without parsing values from strings or evaluating expressions.

A mapping whose only entry is `{"$value": "name"}` is a reserved placeholder
node. Its replacement may be a string, number, boolean, null, list, or mapping;
the replacement is cloned and is not scanned again for placeholders. Ordinary
mapping keys and string contents remain unchanged.

## When to Use

Use this recipe for a trusted, bounded configuration template built from
JSON-like Python values when an entire node needs a typed replacement. Reserve
the `$value` key in the template format and validate the rendered result with
the destination schema. Use a real template language when text interpolation,
conditionals, loops, expressions, or escaping rules are required.

## Implementation

```python
import math
import re
from collections.abc import Mapping


_NAME = re.compile(r"[A-Za-z_][A-Za-z0-9_]*", flags=re.ASCII)
_MARKER = "$value"


class MalformedTemplate(ValueError):
    pass


class MissingTemplateValue(KeyError):
    pass


def _require_name(value: object, *, context: str) -> str:
    if not isinstance(value, str) or _NAME.fullmatch(value) is None:
        raise MalformedTemplate(f"{context} must be an ASCII identifier")
    return value


def _clone_json_like(value: object, active: set[int]) -> object:
    if value is None or isinstance(value, (str, bool, int)):
        return value
    if isinstance(value, float):
        if not math.isfinite(value):
            raise ValueError("JSON-like numbers must be finite")
        return value
    if isinstance(value, Mapping):
        identity = id(value)
        if identity in active:
            raise ValueError("JSON-like values must not contain cycles")
        active.add(identity)
        try:
            cloned: dict[str, object] = {}
            for key, child in value.items():
                if not isinstance(key, str):
                    raise TypeError("JSON-like mapping keys must be strings")
                cloned[key] = _clone_json_like(child, active)
            return cloned
        finally:
            active.remove(identity)
    if isinstance(value, list):
        identity = id(value)
        if identity in active:
            raise ValueError("JSON-like values must not contain cycles")
        active.add(identity)
        try:
            return [_clone_json_like(child, active) for child in value]
        finally:
            active.remove(identity)
    raise TypeError(f"unsupported JSON-like value: {type(value).__name__}")


def _substitute_node(
    node: object,
    values: Mapping[str, object],
    active: set[int],
) -> object:
    if isinstance(node, Mapping):
        if _MARKER in node:
            if len(node) != 1:
                raise MalformedTemplate("a placeholder cannot have sibling keys")
            name = _require_name(node[_MARKER], context="placeholder name")
            if name not in values:
                raise MissingTemplateValue(name)
            return _clone_json_like(values[name], set())

        identity = id(node)
        if identity in active:
            raise ValueError("template must not contain cycles")
        active.add(identity)
        try:
            result: dict[str, object] = {}
            for key, child in node.items():
                if not isinstance(key, str):
                    raise TypeError("template mapping keys must be strings")
                result[key] = _substitute_node(child, values, active)
            return result
        finally:
            active.remove(identity)

    if isinstance(node, list):
        identity = id(node)
        if identity in active:
            raise ValueError("template must not contain cycles")
        active.add(identity)
        try:
            return [_substitute_node(child, values, active) for child in node]
        finally:
            active.remove(identity)

    return _clone_json_like(node, set())


def substitute_typed_values(
    template: object,
    values: Mapping[str, object],
) -> object:
    if not isinstance(values, Mapping):
        raise TypeError("values must be a mapping")

    prepared: dict[str, object] = {}
    for name, value in values.items():
        valid_name = _require_name(name, context="value name")
        prepared[valid_name] = _clone_json_like(value, set())
    return _substitute_node(template, prepared, set())
```

## Example

```python
template = {
    "limit": {"$value": "limit"},
    "enabled": {"$value": "enabled"},
    "fallback": {"$value": "fallback"},
    "labels": {"$value": "labels"},
    "literal": {"$value": "literal_marker"},
    "title": "No string interpolation",
}
values = {
    "limit": 5,
    "enabled": False,
    "fallback": None,
    "labels": ["blue"],
    "literal_marker": {"$value": "not_interpreted_again"},
}

rendered = substitute_typed_values(template, values)
rendered["labels"].append("green")

try:
    substitute_typed_values({"$value": "missing"}, values)
except MissingTemplateValue:
    missing_rejected = True
else:
    missing_rejected = False

try:
    substitute_typed_values({"$value": "limit", "extra": True}, values)
except MalformedTemplate:
    malformed_rejected = True
else:
    malformed_rejected = False

assert (
    rendered,
    values["labels"],
    template["labels"],
    missing_rejected,
    malformed_rejected,
) == (
    {
        "limit": 5,
        "enabled": False,
        "fallback": None,
        "labels": ["blue", "green"],
        "literal": {"$value": "not_interpreted_again"},
        "title": "No string interpolation",
    },
    ["blue"],
    {"$value": "labels"},
    True,
    True,
)
```

## Trade-offs and Limitations

The `$value` key is reserved at every template mapping node and cannot be used
as ordinary template data there. Values are validated and cloned once when
prepared and again for each insertion, so large repeated replacements consume
time and memory; aliases between inserted containers are not preserved. The
recipe accepts only mappings, lists, strings, booleans, integers, finite
floats, and `None`; it rejects tuples, custom scalar objects, non-string keys,
and cycles. It deliberately provides no inline interpolation, type coercion,
default values, expression evaluation, key substitution, schema validation,
output-size limit, or protection against very deep recursive input.

## Related Snippets

<!-- catalog:related:start -->
- [Prune Empty Values from JSON-Like Data](prune-empty-values-from-json-like-data.md)
- [Merge Nested Configuration with an Explicit Delete Sentinel](merge-nested-configuration-with-an-explicit-delete-sentinel.md)
- [Load Text Templates from Package Resources](../python-language/load-text-templates-from-package-resources.md)
<!-- catalog:related:end -->
