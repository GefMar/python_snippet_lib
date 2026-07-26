---
title: "Pass Constructor-Only Context with dataclasses.InitVar"
snippet_type: idiom
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - make-a-defensive-copy-at-a-mutable-input-boundary.md
---

# Pass Constructor-Only Context with dataclasses.InitVar

## Idea and Problem

Use an InitVar when dataclass construction needs a temporary input that should not become part of the stored field model.

An `InitVar` appears in the generated constructor and is forwarded to
`__post_init__`, but it is not a regular dataclass field. This keeps temporary
construction policy separate from the values that describe the resulting
object.

## When to Use

Use this technique for deterministic construction inputs such as a parser,
normalizer, or validation policy that is needed once and is not part of object
identity. Prefer a regular field when later methods need the dependency, and
prefer a factory function when construction requires substantial orchestration
or side effects.

## Implementation

```python
from collections.abc import Callable
from dataclasses import InitVar, dataclass, field


@dataclass(frozen=True, slots=True)
class NormalizedText:
    source: str
    normalizer: InitVar[Callable[[str], str]]
    value: str = field(init=False)

    def __post_init__(self, normalizer: Callable[[str], str]) -> None:
        normalized = normalizer(self.source)
        if not normalized:
            raise ValueError("normalization produced an empty value")
        object.__setattr__(self, "value", normalized)
```

## Example

```python
from dataclasses import fields


record = NormalizedText(
    "  Clear   Name  ",
    lambda text: " ".join(text.split()).casefold(),
)

try:
    NormalizedText("   ", str.strip)
except ValueError as error:
    empty_value_rejected = str(error) == "normalization produced an empty value"
else:
    empty_value_rejected = False

field_names = tuple(item.name for item in fields(record))

assert (record.value, field_names, empty_value_rejected) == (
    "clear name",
    ("source", "value"),
    True,
)
```

## Trade-offs and Limitations

Constructor-only inputs are not available after initialization and are omitted
from helpers that inspect regular dataclass fields. Reconstructing an instance
with `dataclasses.replace` may require supplying an `InitVar` that has no
default. A frozen dataclass must use `object.__setattr__` inside `__post_init__`
to assign a derived field. Avoid hiding I/O or long-running work in this hook;
an explicit factory makes those effects easier to observe and test.

## Related Snippets

<!-- catalog:related:start -->
- [Make a Defensive Copy at a Mutable Input Boundary](make-a-defensive-copy-at-a-mutable-input-boundary.md)
<!-- catalog:related:end -->
