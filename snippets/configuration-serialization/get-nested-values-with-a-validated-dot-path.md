---
title: "Get Nested Values with a Validated Dot Path"
snippet_type: recipe
use_cases:
  - configuration
  - parsing
tested_python:
  - "3.14"
dependencies: []
related: []
---

# Get Nested Values with a Validated Dot Path

## Idea and Problem

Resolve a small dot-path grammar against JSON-like mappings and lists while keeping missing values distinct from malformed paths.

Configuration readers often need concise access to nested values, but an
unwritten path grammar creates ambiguous failures. This helper defines mapping
segments as string keys, list segments as ASCII non-negative integers, and
empty segments as invalid syntax. An optional default handles only absent keys
and out-of-range indexes.

## When to Use

Use this recipe for trusted, simple dot paths over data made only of mappings,
lists, and scalar leaves. It fits configuration lookups when keys do not need
escaped dots and callers need a clear distinction between missing data and an
invalid path. Use a token sequence or a query library if keys can contain dots,
negative indexes are meaningful, or the expression language needs filters and
wildcards.

## Implementation

```python
from collections.abc import Mapping


_MISSING = object()


def get_by_dot_path(
    document: object,
    path: str,
    *,
    default: object = _MISSING,
) -> object:
    segments = path.split(".")
    if not path or any(not segment for segment in segments):
        raise ValueError("path must contain non-empty dot-separated segments")

    current = document
    for segment in segments:
        if isinstance(current, Mapping):
            if segment in current:
                current = current[segment]
                continue
            if default is _MISSING:
                raise KeyError(segment)
            return default

        if isinstance(current, list):
            if not (segment.isascii() and segment.isdecimal()):
                raise ValueError("list path segments must be non-negative integers")
            index = int(segment)
            if index < len(current):
                current = current[index]
                continue
            if default is _MISSING:
                raise IndexError(index)
            return default

        raise TypeError("path continues beyond a mapping or list")

    return current
```

## Example

```python
document = {
    "pipelines": [{"name": "clean"}, {"name": "publish"}],
    "0": {"name": "string key"},
}

selected = get_by_dot_path(document, "pipelines.1.name")
numeric_mapping_key = get_by_dot_path(document, "0.name")
missing_key_fallback = get_by_dot_path(
    document,
    "pipelines.1.enabled",
    default=False,
)

try:
    get_by_dot_path(document, "pipelines..name")
except ValueError:
    malformed_path_rejected = True
else:
    malformed_path_rejected = False

try:
    get_by_dot_path(document, "pipelines.first.name")
except ValueError:
    invalid_index_rejected = True
else:
    invalid_index_rejected = False

try:
    get_by_dot_path(document, "pipelines.4.name")
except IndexError:
    out_of_range_rejected = True
else:
    out_of_range_rejected = False

try:
    get_by_dot_path(document, "pipelines.0.name.extra")
except TypeError:
    scalar_traversal_rejected = True
else:
    scalar_traversal_rejected = False

assert (
    selected,
    numeric_mapping_key,
    missing_key_fallback,
    malformed_path_rejected,
    invalid_index_rejected,
    out_of_range_rejected,
    scalar_traversal_rejected,
) == ("publish", "string key", False, True, True, True, True)
```

## Trade-offs and Limitations

The grammar cannot address a mapping key that contains a dot, and integer
mapping keys are outside the contract. A numeric segment selects a string key
while the current node is a mapping and a list index while it is a list. The
default applies only to a missing key or out-of-range index; malformed syntax
and traversal through a scalar still raise. The return type is `object`, so a
caller that expects a particular value type must validate it separately.

## Related Snippets

<!-- catalog:related:start -->
No related snippets.
<!-- catalog:related:end -->
