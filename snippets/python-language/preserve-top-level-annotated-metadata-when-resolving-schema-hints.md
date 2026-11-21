---
title: "Preserve Top-Level Annotated Metadata When Resolving Schema Hints"
snippet_type: recipe
use_cases:
  - automation
  - configuration
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - inspect-trusted-runtime-annotations-while-preserving-unresolved-names.md
  - ../testing-tooling/build-configuration-help-from-dataclass-field-docs.md
---

# Preserve Top-Level Annotated Metadata When Resolving Schema Hints

## Idea and Problem

Keep each field's resolved type and metadata when reading a trusted schema.

By default, `get_type_hints()` strips `Annotated` wrappers. Pass
`include_extras=True`, then use `get_origin()` and `get_args()` to separate
the type from its metadata. Check the outermost hint so metadata on a
container element is not mistaken for metadata on the whole field.

## When to Use

Use this when a class you control declares units, display labels, or other
field information in `Annotated`, and a documentation or testing tool
needs that information alongside the type.

## Implementation

```python
from typing import Annotated, get_args, get_origin, get_type_hints


def annotated_fields(schema: type) -> dict[str, tuple[object, tuple[object, ...]]]:
    fields: dict[str, tuple[object, tuple[object, ...]]] = {}
    for name, hint in get_type_hints(schema, include_extras=True).items():
        if get_origin(hint) is Annotated:
            value_type, *metadata = get_args(hint)
            fields[name] = value_type, tuple(metadata)
    return fields
```

## Example

```python
class TimingOptions:
    timeout: Annotated[int, "milliseconds", ("suggested_minimum", 1)]
    retry_delays: list[Annotated[int, "milliseconds"]]
    attempts: int


assert get_type_hints(TimingOptions) == {
    "timeout": int,
    "retry_delays": list[int],
    "attempts": int,
}
assert annotated_fields(TimingOptions) == {
    "timeout": (int, ("milliseconds", ("suggested_minimum", 1))),
}
```

## Trade-offs and Limitations

`get_type_hints()` may execute annotation expressions and can fail on an
unresolved reference. Pass only trusted classes. It also merges class hints
from base classes, so returned fields need not be declared on the supplied
class itself.

The helper returns only outermost `Annotated` metadata, in order. It does
not traverse containers, apply a default, or enforce the meaning of any
metadata. Give metadata semantics to a separate consumer.

## Related Snippets

<!-- catalog:related:start -->
- [Inspect Trusted Runtime Annotations while Preserving Unresolved Names](inspect-trusted-runtime-annotations-while-preserving-unresolved-names.md)
- [Build Configuration Help from Dataclass Field Docs](../testing-tooling/build-configuration-help-from-dataclass-field-docs.md)
<!-- catalog:related:end -->
