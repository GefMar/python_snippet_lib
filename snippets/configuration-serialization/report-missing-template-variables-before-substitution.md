---
title: "Report Missing Template Variables Before Substitution"
snippet_type: recipe
use_cases:
  - validation
  - configuration
tested_python:
  - "3.14"
dependencies: []
related:
  - parse-a-bounded-flat-placeholder-template.md
  - substitute-typed-values-into-a-json-like-template.md
---

# Report Missing Template Variables Before Substitution

## Idea and Problem

Report every missing input for a flat string template before producing any rendered output.

Check the template grammar and ask the template for its recognized names. Return the missing names in first-appearance order, or perform ordinary substitution when they are all supplied.

## When to Use

Use this for short trusted templates using the standard dollar-sign placeholders and for string values that are inserted as literal text. It helps a caller report omissions together instead of raising on the first missing key.

## Implementation

```python
from collections.abc import Mapping
from string import Template


def render_if_complete(
    source: str, values: Mapping[str, str]
) -> tuple[str | None, tuple[str, ...]]:
    template = Template(source)
    if not template.is_valid():
        raise ValueError("invalid dollar-sign placeholder")

    missing = tuple(name for name in template.get_identifiers() if name not in values)
    if missing:
        return None, missing
    return template.substitute(values), ()
```

## Example

```python
source = "$city / ${room} / $city / $$"
incomplete = render_if_complete(source, {})
complete = render_if_complete(source, {"city": "$not_recursive", "room": ""})
try:
    render_if_complete("$!", {})
except ValueError:
    malformed = True
else:
    malformed = False

assert (incomplete, complete, malformed) == (
    (None, ("city", "room")),
    ("$not_recursive /  / $not_recursive / $", ()),
    True,
)
```

## Trade-offs and Limitations

The supported grammar has flat names and escaping but no format expressions, conditionals, or escaping for a downstream interpreter. Extra mapping values are ignored. Do not re-run substitution on the rendered data unless interpreting inserted dollar signs is intended.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Bounded Flat Placeholder Template](parse-a-bounded-flat-placeholder-template.md)
- [Substitute Typed Values into a JSON-Like Template](substitute-typed-values-into-a-json-like-template.md)
<!-- catalog:related:end -->
