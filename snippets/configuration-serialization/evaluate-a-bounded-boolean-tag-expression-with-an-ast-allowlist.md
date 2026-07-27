---
title: "Evaluate a Bounded Boolean Tag Expression with an AST Allowlist"
snippet_type: algorithm
use_cases:
  - configuration
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - expand-bounded-nested-brace-alternatives.md
  - ../python-language/dispatch-named-strategies-with-an-explicit-function-mapping.md
---

# Evaluate a Bounded Boolean Tag Expression with an AST Allowlist

## Idea and Problem

Evaluate a tiny boolean tag language by validating and walking Python's expression AST without executing compiled code.

The public grammar contains lowercase tag names, `&`, `|`, unary `~`, spaces,
and parentheses. A complete AST allowlist rejects calls, attributes, constants,
subscripts, comprehensions, and Python's `and` and `or` operators before direct
boolean evaluation. Independent source, node, and nesting limits bound work.

## When to Use

Use this algorithm for small, local configuration predicates when Python's bit
operator precedence is an acceptable documented grammar. Supply a finite set
of allowed names and a subset that is currently selected. Prefer a dedicated
parser when the syntax must evolve, error locations matter, or inputs cross a
high-risk trust boundary; an AST allowlist is not an authorization engine.

## Implementation

```python
import ast
import keyword
import re
from collections.abc import Iterable


_TAG_NAME = re.compile(r"[a-z][a-z0-9_]{0,31}", re.ASCII)
_EXPRESSION_CHARACTERS = frozenset(
    "abcdefghijklmnopqrstuvwxyz0123456789_&|~() "
)
_ALLOWED_AST_NODES = (
    ast.Expression,
    ast.Name,
    ast.Load,
    ast.BinOp,
    ast.BitAnd,
    ast.BitOr,
    ast.UnaryOp,
    ast.Invert,
)


def _bounded_positive_int(value: int, *, name: str, hard_maximum: int) -> int:
    if isinstance(value, bool) or not isinstance(value, int):
        raise TypeError(f"{name} must be an integer")
    if not 1 <= value <= hard_maximum:
        raise ValueError(f"{name} is outside its supported range")
    return value


def _validated_tags(
    values: Iterable[str],
    *,
    name: str,
    allow_empty: bool,
) -> frozenset[str]:
    if isinstance(values, (str, bytes)):
        raise TypeError(f"{name} must be an iterable of tag names")

    result: set[str] = set()
    for value in values:
        if not isinstance(value, str):
            raise TypeError(f"{name} must contain text values")
        if _TAG_NAME.fullmatch(value) is None or keyword.iskeyword(value):
            raise ValueError(f"{name} contains an invalid tag name")
        if value in result:
            raise ValueError(f"{name} contains a duplicate tag name")
        result.add(value)
        if len(result) > 256:
            raise ValueError(f"{name} contains too many tag names")
    if not result and not allow_empty:
        raise ValueError(f"{name} must not be empty")
    return frozenset(result)


def _validate_expression_tree(
    tree: ast.Expression,
    *,
    allowed_tags: frozenset[str],
    max_nodes: int,
    max_depth: int,
) -> None:
    visited = 0
    pending: list[tuple[ast.AST, int]] = [(tree, 1)]
    while pending:
        node, depth = pending.pop()
        visited += 1
        if visited > max_nodes:
            raise ValueError("expression exceeds the AST node limit")
        if depth > max_depth:
            raise ValueError("expression exceeds the AST depth limit")
        if not isinstance(node, _ALLOWED_AST_NODES):
            raise ValueError("expression contains unsupported syntax")
        if isinstance(node, ast.Name) and node.id not in allowed_tags:
            raise ValueError("expression contains an unknown tag")
        pending.extend(
            (child, depth + 1) for child in ast.iter_child_nodes(node)
        )


def _evaluate_expression_node(
    node: ast.expr,
    *,
    selected_tags: frozenset[str],
) -> bool:
    if isinstance(node, ast.Name):
        return node.id in selected_tags
    if isinstance(node, ast.BinOp):
        left = _evaluate_expression_node(node.left, selected_tags=selected_tags)
        right = _evaluate_expression_node(node.right, selected_tags=selected_tags)
        if isinstance(node.op, ast.BitAnd):
            return left and right
        if isinstance(node.op, ast.BitOr):
            return left or right
    if isinstance(node, ast.UnaryOp) and isinstance(node.op, ast.Invert):
        return not _evaluate_expression_node(
            node.operand,
            selected_tags=selected_tags,
        )
    raise AssertionError("validated expression contains an unexpected node")


def evaluate_tag_expression(
    expression: str,
    *,
    selected_tags: Iterable[str],
    allowed_tags: Iterable[str],
    max_source_length: int = 256,
    max_nodes: int = 64,
    max_depth: int = 16,
) -> bool:
    if not isinstance(expression, str):
        raise TypeError("expression must be text")
    source_limit = _bounded_positive_int(
        max_source_length,
        name="max_source_length",
        hard_maximum=4096,
    )
    node_limit = _bounded_positive_int(
        max_nodes,
        name="max_nodes",
        hard_maximum=1024,
    )
    depth_limit = _bounded_positive_int(
        max_depth,
        name="max_depth",
        hard_maximum=64,
    )
    if not expression or len(expression) > source_limit:
        raise ValueError("expression length is outside the accepted range")
    if any(character not in _EXPRESSION_CHARACTERS for character in expression):
        raise ValueError("expression contains a character outside the grammar")

    allowed = _validated_tags(
        allowed_tags,
        name="allowed_tags",
        allow_empty=False,
    )
    selected = _validated_tags(
        selected_tags,
        name="selected_tags",
        allow_empty=True,
    )
    if not selected <= allowed:
        raise ValueError("selected_tags must be a subset of allowed_tags")

    try:
        parsed = ast.parse(expression, mode="eval")
    except SyntaxError:
        raise ValueError("expression is not valid syntax") from None
    assert isinstance(parsed, ast.Expression)
    _validate_expression_tree(
        parsed,
        allowed_tags=allowed,
        max_nodes=node_limit,
        max_depth=depth_limit,
    )
    return _evaluate_expression_node(parsed.body, selected_tags=selected)
```

## Example

```python
allowed = {"fast", "stable", "debug"}
selected = {"fast", "stable"}
matches = evaluate_tag_expression(
    "fast & ~debug | stable",
    selected_tags=selected,
    allowed_tags=allowed,
)
parentheses_change_result = evaluate_tag_expression(
    "fast & ~(debug | stable)",
    selected_tags=selected,
    allowed_tags=allowed,
)


def is_rejected(expression: str) -> bool:
    try:
        evaluate_tag_expression(
            expression,
            selected_tags=selected,
            allowed_tags=allowed,
        )
    except ValueError:
        return True
    return False


try:
    evaluate_tag_expression(
        "fast",
        selected_tags={"fast"},
        allowed_tags={"fast", "and"},
    )
except ValueError:
    keyword_tag_rejected = True
else:
    keyword_tag_rejected = False


assert (
    matches,
    parentheses_change_result,
    is_rejected("fast and stable"),
    is_rejected("missing"),
    is_rejected("fast()"),
    is_rejected("fast.__class__"),
    keyword_tag_rejected,
) == (True, False, True, True, True, True, True)
```

## Trade-offs and Limitations

This deliberately reuses Python's precedence rules: `~` binds more tightly
than `&`, which binds more tightly than `|`. It does not normalize case,
minimize boolean formulas, explain which clause matched, or support quoted tag
names. AST parsing is still a parser and can consume resources, so callers must
keep the hard source, node, and depth caps instead of assuming that avoiding
`eval()` makes arbitrary input harmless. The implementation validates both
branches before evaluation, but evaluates eagerly rather than exposing
Python's short-circuit semantics. Add new syntax only with corresponding AST
allowlist rules and adversarial tests.

## Related Snippets

<!-- catalog:related:start -->
- [Expand Bounded Nested Brace Alternatives](expand-bounded-nested-brace-alternatives.md)
- [Dispatch Named Strategies with an Explicit Function Mapping](../python-language/dispatch-named-strategies-with-an-explicit-function-mapping.md)
<!-- catalog:related:end -->
