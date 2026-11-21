---
title: "Report Multiple Invalid Options in One ExceptionGroup"
snippet_type: recipe
use_cases:
  - validation
  - configuration
tested_python:
  - "3.14"
dependencies: []
related:
  - split-a-bounded-exceptiongroup-by-closed-exception-types.md
  - add-bounded-stage-context-without-replacing-an-exception.md
---

# Report Multiple Invalid Options in One ExceptionGroup

## Idea and Problem

Return all independent option failures in one exception so a caller can fix several values at once.

Build a small error for each bad option in schema order. Raise an exception group only if the list is nonempty. Let unexpected failures propagate rather than classifying every exception as a user validation problem.

## When to Use

Use this for a finite set of independent required integer options, where each option has a known positive upper bound. Missing values and values outside the bound are reported together, while unknown keys remain outside this check.

## Implementation

```python
from collections.abc import Mapping


def validate_options(values: Mapping[str, object], schema: Mapping[str, int]) -> None:
    errors: list[ValueError] = []

    for name, maximum in schema.items():
        value = values.get(name)
        if type(value) is not int or not 1 <= value <= maximum:
            errors.append(ValueError(f"{name}: expected an integer from 1 to {maximum}"))

    if errors:
        raise ExceptionGroup("invalid options", errors)
```

## Example

```python
schema = {"copies": 5, "workers": 8, "pages": 20}
validate_options({"copies": 2, "workers": 3, "pages": 10}, schema)
messages = ()
try:
    validate_options({"copies": True, "pages": 27}, schema)
except* ValueError as group:
    messages = tuple(str(error) for error in group.exceptions)

assert messages == (
    "copies: expected an integer from 1 to 5",
    "workers: expected an integer from 1 to 8",
    "pages: expected an integer from 1 to 20",
)
```

## Trade-offs and Limitations

The checks are independent. Rules that depend on other fields need a separate policy to avoid reporting cascades of secondary errors. The schema is trusted and validated separately. The caller must handle the group as a group or use except* to select member exceptions; a regular except ValueError does not catch the group.

## Related Snippets

<!-- catalog:related:start -->
- [Split a Bounded ExceptionGroup by Closed Exception Types](split-a-bounded-exceptiongroup-by-closed-exception-types.md)
- [Add Bounded Stage Context Without Replacing an Exception](add-bounded-stage-context-without-replacing-an-exception.md)
<!-- catalog:related:end -->
