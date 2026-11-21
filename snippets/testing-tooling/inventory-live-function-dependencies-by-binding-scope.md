---
title: "Inventory Live Function Dependencies by Binding Scope"
snippet_type: testing-technique
use_cases:
  - automation
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - index-bounded-python-scope-bindings-with-symtable-without-execution.md
  - ../python-language/bind-call-arguments-and-materialize-defaults-from-a-signature.md
---

# Inventory Live Function Dependencies by Binding Scope

## Idea and Problem

List the external names a live Python function resolves from its closure, module globals, or builtins.

`inspect.getclosurevars()` reads the current bindings without running the
function body. Keep only the names when you want a concise diagnostic that
does not expose the bound values.

## When to Use

Use this when refactoring a trusted callback or testing whether a factory
keeps configuration in a closure rather than looking it up as a module
global. Use a static scope analyzer when you only have source code.

## Implementation

```python
from inspect import getclosurevars
from types import FunctionType, MethodType


def dependency_names(function: FunctionType | MethodType) -> dict[str, tuple[str, ...]]:
    bindings = getclosurevars(function)
    return {
        "nonlocals": tuple(sorted(bindings.nonlocals)),
        "globals": tuple(sorted(bindings.globals)),
        "builtins": tuple(sorted(bindings.builtins)),
    }
```

## Example

```python
from collections.abc import Callable

LABEL = "limit"


def make_summary(limit: int) -> Callable[[int], str]:
    def summarize(value: int) -> str:
        capped = min(value, limit)
        return LABEL.upper() + ": " + str(capped)

    return summarize


assert dependency_names(make_summary(6)) == {
    "nonlocals": ("limit",),
    "globals": ("LABEL",),
    "builtins": ("min", "str"),
}
```

## Trade-offs and Limitations

The report is a snapshot of current bindings for one Python function or
bound method. It excludes locals such as `capped` and parameters such as
`value`. It does not follow other functions, resolve object attributes, or
tell you which branch will actually run. Inspect functions with initialized
closure cells; an empty cell can make introspection fail.

The helper omits `unbound` on purpose. In CPython 3.14 that set can include
attribute names such as `upper`, so it is not a reliable list of missing
globals.

## Related Snippets

<!-- catalog:related:start -->
- [Index Bounded Python Scope Bindings with symtable without Execution](index-bounded-python-scope-bindings-with-symtable-without-execution.md)
- [Bind Call Arguments and Materialize Defaults from a Signature](../python-language/bind-call-arguments-and-materialize-defaults-from-a-signature.md)
<!-- catalog:related:end -->
