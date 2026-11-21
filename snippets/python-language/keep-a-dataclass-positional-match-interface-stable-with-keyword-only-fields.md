---
title: "Keep a Dataclass Positional Match Interface Stable with Keyword-Only Fields"
snippet_type: idiom
use_cases:
  - interoperability
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - pass-constructor-only-context-with-initvar.md
  - apply-partial-dataclass-updates-with-an-omitted-value-sentinel.md
---

# Keep a Dataclass Positional Match Interface Stable with Keyword-Only Fields

## Idea and Problem

Extend a dataclass with optional settings while preserving its existing positional constructor and class-pattern fields.

Dataclasses generate `__match_args__` from their non-keyword-only init
parameters. Put `KW_ONLY` after the existing positional fields so new
settings appear in the constructor and patterns by name without adding
positional match slots.

## When to Use

Use this when a dataclass already has positional callers or patterns and
you want to add settings they can opt into by name. Preserve the original
position, name, and meaning of the existing fields.

## Implementation

```python
from dataclasses import KW_ONLY, dataclass


@dataclass
class ExportJob:
    name: str
    _: KW_ONLY
    compress: bool = False


def describe_job(job: ExportJob) -> str:
    match job:
        case ExportJob(name, compress=True):
            return f"{name}: compressed"
        case ExportJob(name):
            return name
    raise TypeError("expected an ExportJob")
```

## Example

```python
base = ExportJob("daily")
configured = ExportJob("weekly", compress=True)

try:
    ExportJob("weekly", True)
except TypeError:
    constructor_rejected_extra_position = True
else:
    constructor_rejected_extra_position = False

try:
    match configured:
        case ExportJob(_, _):
            raise AssertionError("compress should not occupy a positional pattern slot")
except TypeError:
    pattern_rejected_extra_position = True
else:
    pattern_rejected_extra_position = False

assert (
    ExportJob.__match_args__,
    describe_job(base),
    describe_job(configured),
    constructor_rejected_extra_position,
    pattern_rejected_extra_position,
) == (("name",), "daily", "weekly: compressed", True, True)
```

## Trade-offs and Limitations

`KW_ONLY` applies to the fields that follow it in this dataclass. It does
not protect existing positional callers if you later reorder or remove
their fields. Tests can assert `__match_args__` directly when positional
patterns are part of the public interface.

Keyword-only fields still participate in dataclass behavior such as equality
and representation unless you change the field options. Review those
contracts separately when extending the class.

## Related Snippets

<!-- catalog:related:start -->
- [Pass Constructor-Only Context with dataclasses.InitVar](pass-constructor-only-context-with-initvar.md)
- [Apply Partial Dataclass Updates with an Omitted-Value Sentinel](apply-partial-dataclass-updates-with-an-omitted-value-sentinel.md)
<!-- catalog:related:end -->
