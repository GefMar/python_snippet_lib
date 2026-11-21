---
title: "Read Direct Generic Base Arguments at Runtime"
snippet_type: recipe
use_cases:
  - interoperability
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-c3-linearizations-for-a-bounded-indexed-class-hierarchy.md
  - inspect-trusted-runtime-annotations-while-preserving-unresolved-names.md
---

# Read Direct Generic Base Arguments at Runtime

## Idea and Problem

Recover the type arguments written on a class's direct generic base.

`__bases__` tells you which classes became runtime bases, but it can lose
arguments such as `str` and `int` from `dict[str, int]`. Use
`types.get_original_bases()` to see the declarations before Python
resolved the actual bases. Match a base by origin identity, then read its
arguments.

## When to Use

Use this in a developer diagnostic or a test for trusted classes you own,
when a class directly declares the generic specialization you want to
report. Do not use it to check the contents of an instance.

## Implementation

```python
from types import get_original_bases
from typing import get_args, get_origin


def direct_base_args(cls: type, origin: type) -> tuple[object, ...] | None:
    for base in get_original_bases(cls):
        if get_origin(base) is origin:
            return get_args(base)
    return None
```

## Example

```python
class Labeled:
    pass


class LabelIndex(Labeled, dict[str, int]):
    pass


class DerivedIndex(LabelIndex):
    pass


assert LabelIndex.__bases__ == (Labeled, dict)
assert direct_base_args(LabelIndex, dict) == (str, int)
assert direct_base_args(LabelIndex, list) is None
assert get_original_bases(DerivedIndex) == (LabelIndex,)
assert direct_base_args(DerivedIndex, dict) is None
```

## Trade-offs and Limitations

The helper returns the first direct matching alias or `None` if there is
none. A plain base such as `dict` does not have type arguments. On Python
3.14, a class without its own recorded original bases falls back to its
direct `__bases__`. Looking up `cls.__orig_bases__` instead can find a value
inherited from a parent.

This does not walk ancestors or resolve type variables across inheritance.
The returned arguments are declaration metadata, not constraints enforced
on instances.

## Related Snippets

<!-- catalog:related:start -->
- [Compute C3 Linearizations for a Bounded Indexed Class Hierarchy](compute-c3-linearizations-for-a-bounded-indexed-class-hierarchy.md)
- [Inspect Trusted Runtime Annotations while Preserving Unresolved Names](inspect-trusted-runtime-annotations-while-preserving-unresolved-names.md)
<!-- catalog:related:end -->
