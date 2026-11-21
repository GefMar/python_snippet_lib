---
title: "Normalize Raw Docstring Indentation with inspect.cleandoc"
snippet_type: idiom
use_cases:
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../testing-tooling/extract-bounded-doctest-examples-without-executing-them.md
  - ../testing-tooling/extract-function-default-expressions-from-python-source.md
---

# Normalize Raw Docstring Indentation with inspect.cleandoc

## Idea and Problem

Turn a raw docstring into readable help text without letting an inline first line prevent dedentation of the rest.

Clean the string already in hand instead of looking up documentation on a live object. The first line and the remaining lines have different indentation rules, which preserves indents used for nested examples.

## When to Use

Use this in documentation rendering when the input is a raw docstring or follows the same indentation convention. Keep an explicit distinction between an absent docstring and a present but empty one.

## Implementation

```python
from inspect import cleandoc


def normalize_docstring(raw: str | None) -> str | None:
    return None if raw is None else cleandoc(raw)
```

## Example

```python
raw = "  Overview\n\tDetails:\n\t    sample()\n\n"
assert (
    normalize_docstring(raw),
    normalize_docstring(None),
    normalize_docstring(""),
    normalize_docstring("\n    \n"),
) == ("Overview\nDetails:\n    sample()", None, "", "    ")
```

## Trade-offs and Limitations

This treats tabs as indentation, removes empty lines at the outer edges and can change whitespace. It is not a general strip operation: a sole whitespace-only line may retain spaces. It does not render markup or inspect class inheritance. If retrieving documentation from a live object's hierarchy is desired, use inspect.getdoc; if a literal block must be preserved exactly, do not clean it.

## Related Snippets

<!-- catalog:related:start -->
- [Extract Bounded Doctest Examples Without Executing Them](../testing-tooling/extract-bounded-doctest-examples-without-executing-them.md)
- [Extract Function Default Expressions from Python Source](../testing-tooling/extract-function-default-expressions-from-python-source.md)
<!-- catalog:related:end -->
