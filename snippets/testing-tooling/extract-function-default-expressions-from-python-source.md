---
title: "Extract Function Default Expressions from Python Source"
snippet_type: testing-technique
use_cases:
  - parsing
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - extract-a-bounded-static-python-annotation-index-without-evaluation.md
  - decode-bounded-python-source-bytes-with-tokenize-detect-encoding.md
---

# Extract Function Default Expressions from Python Source

## Idea and Problem

Inspect the written defaults of a function without importing its module or evaluating any of the expressions.

Positional defaults belong to the last parameters that can be passed
positionally. Keyword-only defaults line up with their parameters, with
a missing AST node for a required parameter. Recover the expression text
using AST source locations, which account for multibyte characters.

## When to Use

Use this for a review tool that compares the defaults of an explicitly named,
top-level function in trusted Python source. It can report that a default is
a function call even when importing the module would perform work.

## Implementation

```python
import ast


def default_expressions(source: str, function_name: str) -> dict[str, str]:
    definitions = [
        node
        for node in ast.parse(source).body
        if isinstance(node, (ast.FunctionDef, ast.AsyncFunctionDef)) and node.name == function_name
    ]
    if len(definitions) != 1:
        raise ValueError("expected exactly one matching top-level definition")

    arguments = definitions[0].args
    positional = [*arguments.posonlyargs, *arguments.args]
    first_default = len(positional) - len(arguments.defaults)
    defaults = dict(
        zip(
            (parameter.arg for parameter in positional[first_default:]),
            arguments.defaults,
            strict=True,
        )
    )
    for parameter, default in zip(arguments.kwonlyargs, arguments.kw_defaults, strict=True):
        if default is not None:
            defaults[parameter.arg] = default

    expressions = {}
    for parameter, default in defaults.items():
        expression = ast.get_source_segment(source, default)
        if expression is None:
            raise ValueError("default expression is missing a source location")
        expressions[parameter] = expression
    return expressions
```

## Example

```python
source = """\
@register
async def export(résumé, label="café", /, delimiter=select_delimiter(), *, limit, fallback=None):
    raise RuntimeError("not called")
"""

assert default_expressions(source, "export") == {
    "label": '"café"',
    "delimiter": "select_delimiter()",
    "fallback": "None",
}
```

## Trade-offs and Limitations

This parses the whole source once per call and selects definitions directly
in the module body. It ignores nested definitions, assignments and decorators
that change the runtime signature. It reports the AST expression span, which
can omit redundant outer parentheses. Use runtime introspection when actual
default objects or decorator effects matter.

Pass decoded source and use the parser version appropriate to its grammar.
Parsing has resource costs, so use trusted source or enforce input limits
before parsing.

## Related Snippets

<!-- catalog:related:start -->
- [Extract a Bounded Static Python Annotation Index without Evaluation](extract-a-bounded-static-python-annotation-index-without-evaluation.md)
- [Decode Bounded Python Source Bytes with Python Encoding Detection](decode-bounded-python-source-bytes-with-tokenize-detect-encoding.md)
<!-- catalog:related:end -->
