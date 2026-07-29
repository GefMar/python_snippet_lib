---
title: "Compare Bounded Python Expressions Structurally without Execution"
snippet_type: testing-technique
use_cases:
  - parsing
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - extract-a-bounded-static-python-annotation-index-without-evaluation.md
  - ../configuration-serialization/evaluate-a-bounded-boolean-tag-expression-with-an-ast-allowlist.md
  - ../data-processing/count-static-imports-across-bounded-python-notebook-cells.md
---

# Compare Bounded Python Expressions Structurally without Execution

## Idea and Problem

Compare the syntax trees of two bounded Python expressions while ignoring source locations and never executing either expression.

Text equality is often too strict for source-oriented tests: whitespace,
comments, and redundant parentheses can change while the parsed expression
stays the same. Python 3.14's `ast.compare` can compare the tree fields that
represent syntax. Explicit source, aggregate-byte, node-count, and tree-depth
limits keep the admitted comparison small before the recursive comparison.

## When to Use

Use this helper in codemod tests, generated-code checks, or developer tools
that need to decide whether two standalone expressions have the same AST
shape. Both strings must be trusted as data to the Python parser, and a syntax
error is deliberately left visible to the caller.

This is structural equality only. It does not prove runtime or algebraic
equivalence, resolve names, fold constants, compare inferred types, or account
for side effects. For example, `left + right` and `right + left` are different
trees even when a particular value type happens to make the operation
commutative.

## Implementation

```python
import ast

_MAX_EXPRESSION_CHARACTERS = 32_768
_MAX_COMBINED_BYTES = 65_536
_MAX_AST_NODES = 4_096
_MAX_AST_DEPTH = 64


def _encoded_expression(expression: str, *, name: str) -> bytes:
    if type(expression) is not str:
        raise TypeError(f"{name} must be an exact string")
    if not expression:
        raise ValueError(f"{name} must not be empty")
    if len(expression) > _MAX_EXPRESSION_CHARACTERS:
        raise ValueError(f"{name} exceeds the supported character count")
    try:
        return expression.encode("utf-8")
    except UnicodeEncodeError as error:
        raise ValueError(f"{name} must not contain Unicode surrogates") from error


def _parse_bounded_expression(expression: str) -> ast.Expression:
    try:
        tree = ast.parse(expression, mode="eval")
    except (MemoryError, RecursionError) as error:
        raise ValueError("expression exceeds the parser's supported depth") from error

    stack: list[tuple[ast.AST, int]] = [(tree, 0)]
    node_count = 0
    while stack:
        node, depth = stack.pop()
        node_count += 1
        if node_count > _MAX_AST_NODES:
            raise ValueError("expression exceeds the supported AST node count")
        if depth > _MAX_AST_DEPTH:
            raise ValueError("expression exceeds the supported AST depth")
        stack.extend((child, depth + 1) for child in ast.iter_child_nodes(node))
    return tree


def structurally_equal_expressions(left: str, right: str) -> bool:
    left_bytes = _encoded_expression(left, name="left expression")
    right_bytes = _encoded_expression(right, name="right expression")
    if len(left_bytes) + len(right_bytes) > _MAX_COMBINED_BYTES:
        raise ValueError("expressions exceed the combined UTF-8 byte count")

    left_tree = _parse_bounded_expression(left)
    right_tree = _parse_bounded_expression(right)
    return ast.compare(left_tree, right_tree, compare_attributes=False)
```

## Example

```python
def explode() -> None:
    raise AssertionError("the expression must never be called")


assert structurally_equal_expressions(
    "total + (price * 2)",
    "total+price*2  # formatting is not part of the AST",
)
assert structurally_equal_expressions("explode()", "(explode())")
assert not structurally_equal_expressions("left + right", "right + left")
assert not structurally_equal_expressions("limit >= 10", "limit > 10")

wide_accepted = ",".join("0" for _ in range(4_093))
wide_rejected = ",".join("0" for _ in range(4_094))
deep_accepted = "~" * 62 + "x"
deep_rejected = "~" * 63 + "x"

assert structurally_equal_expressions(wide_accepted, wide_accepted)
assert structurally_equal_expressions(deep_accepted, deep_accepted)
for expression in (wide_rejected, deep_rejected):
    try:
        structurally_equal_expressions(expression, expression)
    except ValueError:
        pass
    else:
        raise AssertionError("AST workload limit must be enforced")

assert structurally_equal_expressions("result", "(result)")
```

## Trade-offs and Limitations

For `B` admitted source bytes and `N` visited AST nodes, parsing, checking, and
comparison use `O(B + N)` time and retained state. The fixed budgets bound both
trees together at the public entry point.

The Python parser runs before the node and depth walk, so those post-parse
limits are workload bounds rather than a hard parser-containment boundary.
Parser stack `MemoryError` and `RecursionError` are normalized to `ValueError`,
but ordinary `SyntaxError` retains its original location details. Use process
isolation as well when parsing must cross a hostile-input trust boundary.

Ignoring AST attributes intentionally ignores source positions. It does not
ignore operators, literal values, argument order, comprehension structure, or
other semantic fields. The helper accepts one expression rather than a module,
does not preserve spelling such as quote style, and never compiles or evaluates
the parsed trees.

## Related Snippets

<!-- catalog:related:start -->
- [Extract a Bounded Static Python Annotation Index without Evaluation](extract-a-bounded-static-python-annotation-index-without-evaluation.md)
- [Evaluate a Bounded Boolean Tag Expression with an AST Allowlist](../configuration-serialization/evaluate-a-bounded-boolean-tag-expression-with-an-ast-allowlist.md)
- [Count Static Imports Across Bounded Python Notebook Cells](../data-processing/count-static-imports-across-bounded-python-notebook-cells.md)
<!-- catalog:related:end -->
