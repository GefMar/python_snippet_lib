---
title: "Find Repeated Literal Dictionary Keys Without Executing Python"
snippet_type: testing-technique
use_cases:
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - extract-function-default-expressions-from-python-source.md
  - ../configuration-serialization/parse-bounded-utf-8-json-without-duplicate-object-names-or-non-finite-numbers.md
---

# Find Repeated Literal Dictionary Keys Without Executing Python

## Idea and Problem

Report repeated direct scalar keys inside each Python dictionary display without importing the module.

Use the built-in AST to distinguish actual scalar literals from computed expressions and dictionary unpacking. Track each dictionary separately, and use Python dictionary key equality, so a key written as `True` matches one written as `1`.

## When to Use

Use this as a small source-review check on trusted, size-limited text. A result contains the later literal's representation, the line of its first matching occurrence, and its own line. Lines are one-based; there is no runtime value lookup.

## Implementation

```python
import ast


def repeated_literal_keys(source: str) -> list[tuple[str, int, int]]:
    repeats: list[tuple[str, int, int]] = []
    for node in ast.walk(ast.parse(source)):
        if not isinstance(node, ast.Dict):
            continue
        first_lines: dict[object, int] = {}
        for key in node.keys:
            if not isinstance(key, ast.Constant):
                continue
            value = key.value
            if value in first_lines:
                repeats.append((repr(value), first_lines[value], key.lineno))
            else:
                first_lines[value] = key.lineno
    return sorted(repeats, key=lambda repeat: repeat[2])
```

## Example

```python
source = """items = {
    "zone": 3,
    True: 4,
    **load_prior_values(),
    "zone": 8,
    1: 9,
    None: 1,
    None: 2,
}
other = {"zone": 0, -1: 4, -1: 5}
"""

assert repeated_literal_keys(source) == [
    ("'zone'", 2, 5),
    ("1", 3, 6),
    ("None", 7, 8),
]
```

## Trade-offs and Limitations

This is not a full duplicate-key checker. It skips expressions such as negative numbers parsed as unary operations, names, tuples, calls, and keys contributed by unpacking. An expression can overwrite a key without producing a report. Malformed input may raise `SyntaxError`; unrestricted source size and nesting are outside the scope.

## Related Snippets

<!-- catalog:related:start -->
- [Extract Function Default Expressions from Python Source](extract-function-default-expressions-from-python-source.md)
- [Parse Bounded UTF-8 JSON Without Duplicate Object Names or Non-Finite Numbers](../configuration-serialization/parse-bounded-utf-8-json-without-duplicate-object-names-or-non-finite-numbers.md)
<!-- catalog:related:end -->
