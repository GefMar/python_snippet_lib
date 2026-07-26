---
title: "Prune Empty Values from JSON-Like Data"
snippet_type: recipe
use_cases:
  - data-transformation
  - serialization
tested_python:
  - "3.14"
dependencies: []
related:
  - merge-nested-mappings-without-mutating-inputs.md
---

# Prune Empty Values from JSON-Like Data

## Idea and Problem

Remove explicitly defined empty values from generated JSON-like data while preserving meaningful falsy scalars and leaving the input untouched.

The recursive copy drops `None`, empty dictionaries, empty lists, and optionally
empty strings from parent containers. Values such as `0` and `False` remain
because emptiness is tested by type and identity rather than generic truthiness.
Containers that become empty after their children are pruned are removed too.

## When to Use

Use this recipe at a generated configuration or payload boundary where absence
has intentionally different semantics from explicit empty values. Decide the
empty-string policy for that exact boundary and apply the helper before
serialization. Do not apply it to arbitrary user data or patch documents where
`null`, an empty list, or an empty string may be a meaningful instruction.

## Implementation

```python
from typing import TypeAlias


JsonScalar: TypeAlias = bool | int | float | str | None
JsonValue: TypeAlias = JsonScalar | list["JsonValue"] | dict[str, "JsonValue"]


def _is_prunable(value: JsonValue, *, drop_empty_strings: bool) -> bool:
    if value is None:
        return True
    if drop_empty_strings and isinstance(value, str) and value == "":
        return True
    return isinstance(value, (dict, list)) and not value


def prune_empty_values(
    value: JsonValue,
    *,
    drop_empty_strings: bool = True,
) -> JsonValue:
    if isinstance(value, dict):
        cleaned_mapping: dict[str, JsonValue] = {}
        for key, item in value.items():
            if not isinstance(key, str):
                raise TypeError("JSON-like object keys must be strings")
            cleaned = prune_empty_values(
                item,
                drop_empty_strings=drop_empty_strings,
            )
            if not _is_prunable(cleaned, drop_empty_strings=drop_empty_strings):
                cleaned_mapping[key] = cleaned
        return cleaned_mapping

    if isinstance(value, list):
        cleaned_list: list[JsonValue] = []
        for item in value:
            cleaned = prune_empty_values(
                item,
                drop_empty_strings=drop_empty_strings,
            )
            if not _is_prunable(cleaned, drop_empty_strings=drop_empty_strings):
                cleaned_list.append(cleaned)
        return cleaned_list

    if value is None or isinstance(value, (bool, int, float, str)):
        return value
    raise TypeError("value must contain only JSON-like data")
```

## Example

```python
payload = {
    "name": "report",
    "metadata": {"owner": None, "labels": []},
    "options": [None, {"enabled": False, "retries": 0}, ""],
}
cleaned = prune_empty_values(payload)
keep_empty_text = prune_empty_values(
    {"label": "", "nested": [""]},
    drop_empty_strings=False,
)

try:
    prune_empty_values({"invalid": (1, 2)})
except TypeError:
    tuple_rejected = True
else:
    tuple_rejected = False

assert (
    cleaned,
    payload,
    keep_empty_text,
    prune_empty_values({}),
    prune_empty_values([]),
    prune_empty_values(None),
    tuple_rejected,
) == (
    {"name": "report", "options": [{"enabled": False, "retries": 0}]},
    {
        "name": "report",
        "metadata": {"owner": None, "labels": []},
        "options": [None, {"enabled": False, "retries": 0}, ""],
    },
    {"label": "", "nested": [""]},
    {},
    [],
    None,
    True,
)
```

## Trade-offs and Limitations

The helper materializes a full copy and recursively visits every element, so
deep inputs can exceed Python's recursion limit and cyclic containers recurse
forever. A root empty value has no parent from which to remove it and is
returned unchanged. Numeric validity, schema rules, duplicate-key handling,
and JSON serialization remain separate concerns. Most importantly, pruning is
a lossy policy and is unsafe where explicit emptiness carries domain meaning.

## Related Snippets

<!-- catalog:related:start -->
- [Merge Nested Mappings Without Mutating Inputs](merge-nested-mappings-without-mutating-inputs.md)
<!-- catalog:related:end -->
