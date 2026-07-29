---
title: "Extract a Bounded Static Python Annotation Index without Evaluation"
snippet_type: testing-technique
use_cases:
  - parsing
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../data-processing/count-static-imports-across-bounded-python-notebook-cells.md
  - ../configuration-serialization/evaluate-a-bounded-boolean-tag-expression-with-an-ast-allowlist.md
  - parse-a-bounded-debugger-function-listing-into-canonical-source-locations.md
---

# Extract a Bounded Static Python Annotation Index without Evaluation

## Idea and Problem

Index selected Python annotations from source text while preserving their exact spelling and never evaluating the expressions they contain.

Annotations can contain unresolved names, calls, subscriptions, and arbitrary
expressions. Importing a module, calling `typing.get_type_hints`, or compiling
and evaluating those expressions crosses an execution boundary. An AST keeps
the operation syntactic, while explicit source, tree, depth, result, and
retained-text limits make the accepted workload clear.

## When to Use

Use this helper in a bounded documentation check, migration audit, or test tool
that needs the annotations declared directly by a module and its top-level
classes. It records simple annotated variables plus every annotated parameter
and return value of direct synchronous and asynchronous functions.

The returned source positions describe the annotation expression itself.
Variable subjects are the declared names, parameter subjects are argument
names, and a return site's subject is its callable name. Owner paths identify
the containing function, class, or method. Use a full static-analysis framework
when imports, aliases, nested scopes, inferred types, or cross-file name
resolution matter.

## Implementation

```python
import ast
import re
from dataclasses import dataclass
from enum import StrEnum

_MAX_SOURCE_CHARACTERS = 65_536
_MAX_SOURCE_BYTES = 131_072
_MAX_AST_NODES = 5_000
_MAX_AST_DEPTH = 64
_MAX_SITES = 512
_MAX_RETAINED_CHARACTERS = 32_768
_PARSER_LINE = re.compile(r"(.*?(?:\r\n|\n|\r|$))")


class AnnotationRole(StrEnum):
    VARIABLE = "variable"
    PARAMETER = "parameter"
    RETURN = "return"


@dataclass(frozen=True, slots=True)
class AnnotationSite:
    owner: tuple[str, ...]
    subject: str
    role: AnnotationRole
    annotation: str
    line: int
    column: int
    end_line: int
    end_column: int


def _check_tree_limits(tree: ast.AST) -> None:
    stack = [(tree, 0)]
    node_count = 0
    while stack:
        node, depth = stack.pop()
        node_count += 1
        if node_count > _MAX_AST_NODES:
            raise ValueError("AST exceeds the supported node count")
        if depth > _MAX_AST_DEPTH:
            raise ValueError("AST exceeds the supported depth")
        stack.extend((child, depth + 1) for child in ast.iter_child_nodes(node))


def _encode_parser_lines(source: str) -> tuple[bytes, ...]:
    # Match the parser's CRLF/LF/CR line boundaries without treating form feed
    # as a line break. Encoding once also makes AST byte-column slicing cheap.
    return tuple(
        match[0].encode("utf-8")
        for match in _PARSER_LINE.finditer(source)
    )


def _source_segment(lines: tuple[bytes, ...], node: ast.expr) -> str:
    if node.end_lineno is None or node.end_col_offset is None:
        raise ValueError("annotation has no complete source span")

    first_line = node.lineno - 1
    last_line = node.end_lineno - 1
    if first_line == last_line:
        encoded = lines[first_line][node.col_offset : node.end_col_offset]
    else:
        encoded = b"".join(
            (
                lines[first_line][node.col_offset :],
                *lines[first_line + 1 : last_line],
                lines[last_line][: node.end_col_offset],
            )
        )
    return encoded.decode("utf-8")


def extract_static_annotation_index(source: str) -> tuple[AnnotationSite, ...]:
    if type(source) is not str:
        raise TypeError("source must be an exact string")
    if len(source) > _MAX_SOURCE_CHARACTERS:
        raise ValueError("source exceeds the supported character count")
    try:
        encoded_source = source.encode("utf-8")
    except UnicodeEncodeError as error:
        raise ValueError("source must not contain Unicode surrogates") from error
    if len(encoded_source) > _MAX_SOURCE_BYTES:
        raise ValueError("source exceeds the supported UTF-8 byte count")

    tree = ast.parse(source, mode="exec", type_comments=False)
    _check_tree_limits(tree)
    lines = _encode_parser_lines(source)
    sites: list[AnnotationSite] = []
    retained_characters = 0

    def add_site(
        owner: tuple[str, ...],
        subject: str,
        role: AnnotationRole,
        annotation_node: ast.expr,
    ) -> None:
        nonlocal retained_characters
        if len(sites) >= _MAX_SITES:
            raise ValueError("annotation index exceeds the supported site count")
        annotation = _source_segment(lines, annotation_node)
        retained_characters += len(annotation)
        if retained_characters > _MAX_RETAINED_CHARACTERS:
            raise ValueError("retained annotation text exceeds the supported count")
        if (
            annotation_node.end_lineno is None
            or annotation_node.end_col_offset is None
        ):
            raise ValueError("annotation has no complete source span")
        sites.append(
            AnnotationSite(
                owner=owner,
                subject=subject,
                role=role,
                annotation=annotation,
                line=annotation_node.lineno,
                column=annotation_node.col_offset,
                end_line=annotation_node.end_lineno,
                end_column=annotation_node.end_col_offset,
            )
        )

    def add_assignment(node: ast.AnnAssign, owner: tuple[str, ...]) -> None:
        if not isinstance(node.target, ast.Name) or node.simple != 1:
            raise ValueError(
                "admitted annotated assignments must target one simple name"
            )
        add_site(
            owner,
            node.target.id,
            AnnotationRole.VARIABLE,
            node.annotation,
        )

    def add_signature(
        node: ast.FunctionDef | ast.AsyncFunctionDef,
        owner: tuple[str, ...],
    ) -> None:
        arguments = node.args
        positional = (*arguments.posonlyargs, *arguments.args)
        for argument in positional:
            if argument.annotation is not None:
                add_site(
                    owner,
                    argument.arg,
                    AnnotationRole.PARAMETER,
                    argument.annotation,
                )
        if arguments.vararg is not None and arguments.vararg.annotation is not None:
            add_site(
                owner,
                arguments.vararg.arg,
                AnnotationRole.PARAMETER,
                arguments.vararg.annotation,
            )
        for argument in arguments.kwonlyargs:
            if argument.annotation is not None:
                add_site(
                    owner,
                    argument.arg,
                    AnnotationRole.PARAMETER,
                    argument.annotation,
                )
        if arguments.kwarg is not None and arguments.kwarg.annotation is not None:
            add_site(
                owner,
                arguments.kwarg.arg,
                AnnotationRole.PARAMETER,
                arguments.kwarg.annotation,
            )
        if node.returns is not None:
            add_site(owner, node.name, AnnotationRole.RETURN, node.returns)

    function_nodes = (ast.FunctionDef, ast.AsyncFunctionDef)
    for statement in tree.body:
        if isinstance(statement, ast.AnnAssign):
            add_assignment(statement, ())
        elif isinstance(statement, function_nodes):
            add_signature(statement, (statement.name,))
        elif isinstance(statement, ast.ClassDef):
            for class_statement in statement.body:
                if isinstance(class_statement, ast.AnnAssign):
                    add_assignment(class_statement, (statement.name,))
                elif isinstance(class_statement, function_nodes):
                    add_signature(
                        class_statement,
                        (statement.name, class_statement.name),
                    )

    role_order = {
        AnnotationRole.VARIABLE: 0,
        AnnotationRole.PARAMETER: 1,
        AnnotationRole.RETURN: 2,
    }
    sites.sort(
        key=lambda site: (
            site.line,
            site.column,
            site.end_line,
            site.end_column,
            role_order[site.role],
            site.owner,
            site.subject,
        )
    )
    return tuple(sites)
```

## Example

```python
source = """\
threshold: Limit
threshold: list[int]

async def collect(
    prefix: Namespace,
    /,
    item: explode(),
    *extras: tuple[int, ...],
    ready: bool = False,
    **options: str,
) -> Result[
    bytes
]:
    raise AssertionError("function bodies are not executed")

class Worker:
    state: State

    def convert(self, value: Input) -> Output:
        raise AssertionError("function bodies are not executed")
"""

sites = extract_static_annotation_index(source)
threshold_sites = tuple(
    site.annotation
    for site in sites
    if site.subject == "threshold"
)
item_site = next(site for site in sites if site.subject == "item")
collect_return = next(
    site
    for site in sites
    if site.owner == ("collect",) and site.role is AnnotationRole.RETURN
)
method_parameter = next(
    site
    for site in sites
    if site.owner == ("Worker", "convert") and site.subject == "value"
)

assert len(sites) == 11
assert threshold_sites == ("Limit", "list[int]")
assert item_site.annotation == "explode()"
assert collect_return.annotation == "Result[\n    bytes\n]"
assert method_parameter.annotation == "Input"
```

## Trade-offs and Limitations

`ast.parse` must still build the tree before the post-parse node and depth
limits can reject it. Keep the source byte limit small enough for the caller's
resource budget, and use process isolation for hostile workloads that require
hard CPU or memory containment. The cached UTF-8 line list avoids repeatedly
splitting the complete source for each annotation; extraction is linear in the
source, tree, and retained text, plus result sorting.

This deliberately ignores imports, type comments, type aliases, type-parameter
bounds, decorators, nested classes, local variables, nested functions, and
annotations hidden inside conditional or exception-handler bodies. It neither
resolves names nor proves that an annotation is valid for a type checker.
Duplicate declarations are retained as separate evidence rather than merged.

## Related Snippets

<!-- catalog:related:start -->
- [Count Static Imports Across Bounded Python Notebook Cells](../data-processing/count-static-imports-across-bounded-python-notebook-cells.md)
- [Evaluate a Bounded Boolean Tag Expression with an AST Allowlist](../configuration-serialization/evaluate-a-bounded-boolean-tag-expression-with-an-ast-allowlist.md)
- [Parse a Bounded Debugger Function Listing into Canonical Source Locations](parse-a-bounded-debugger-function-listing-into-canonical-source-locations.md)
<!-- catalog:related:end -->
