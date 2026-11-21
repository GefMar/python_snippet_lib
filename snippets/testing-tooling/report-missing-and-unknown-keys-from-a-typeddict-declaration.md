---
title: "Report Missing and Unknown Keys from a TypedDict Declaration"
snippet_type: testing-technique
use_cases:
  - configuration
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../configuration-serialization/normalize-bounded-named-options-with-explicit-default-semantics.md
  - report-missing-declared-protocol-members-without-binding-descriptors.md
---

# Report Missing and Unknown Keys from a TypedDict Declaration

## Idea and Problem

Report missing required keys and unexpected key names in one pass.

Read the required and optional key sets from a `TypedDict` declaration rather
than repeating the names in a second schema. Those sets account for
inheritance and per-key qualifiers, unlike the class's `__total__` flag.

## When to Use

Use this to diagnose typos and omissions in test fixtures or string-keyed
options. It fits when you own the declaration and deliberately accept only
the declared keys.

## Implementation

```python
from collections.abc import Mapping
from typing import is_typeddict


def report_keys(
    data: Mapping[str, object], declaration: type
) -> tuple[tuple[str, ...], tuple[str, ...]]:
    if not is_typeddict(declaration):
        raise TypeError("expected a TypedDict declaration")

    required = declaration.__required_keys__
    allowed = required | declaration.__optional_keys__
    actual = set(data)
    return tuple(sorted(required - actual)), tuple(sorted(actual - allowed))
```

## Example

```python
from typing import Required, TypedDict


class Named(TypedDict):
    name: str


class ExportOptions(Named, total=False):
    format: Required[str]
    label: str | None


assert report_keys({"label": None, "naem": "daily", "debug": True}, ExportOptions) == (
    ("format", "name"),
    ("debug", "naem"),
)
assert report_keys({"name": "daily", "format": "csv"}, ExportOptions) == ((), ())
assert report_keys({"name": "daily", "format": None}, ExportOptions) == ((), ())
```

## Trade-offs and Limitations

Only keys are checked. A present key with an invalid value is not missing;
the final example deliberately supplies a value that violates the annotation.
The helper does not coerce values, apply defaults, or establish that a
mapping conforms to the declaration. Rejecting extra keys is a local
policy, not a rule for all `TypedDict` values.

Use trusted declarations with key qualifiers such as `Required` and
`NotRequired` written directly, without quoted annotations or
`from __future__ import annotations`. Postponed qualifiers can leave the
runtime key sets incorrect; this helper only reads those sets.

## Related Snippets

<!-- catalog:related:start -->
- [Normalize Bounded Named Options with Explicit Default Semantics](../configuration-serialization/normalize-bounded-named-options-with-explicit-default-semantics.md)
- [Report Missing Declared Protocol Members Without Binding Descriptors](report-missing-declared-protocol-members-without-binding-descriptors.md)
<!-- catalog:related:end -->
