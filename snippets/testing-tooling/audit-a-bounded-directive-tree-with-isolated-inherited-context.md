---
title: "Audit a Bounded Directive Tree with Isolated Inherited Context"
snippet_type: pattern
use_cases:
  - configuration
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - parse-a-bounded-space-indented-test-outline-into-leaf-paths.md
  - ../configuration-serialization/evaluate-a-bounded-boolean-rule-tree-from-closed-json.md
  - ../python-language/walk-a-tree-recursively-with-yield-from.md
---

# Audit a Bounded Directive Tree with Isolated Inherited Context

## Idea and Problem

Audit an immutable directive tree in source order while isolating context changes at each nested scope.

Rules see inherited bindings plus bindings introduced by earlier siblings, but
not the current directive's new bindings. Those new bindings become visible to
the directive's children and later siblings only after every rule has audited
the directive. Each child list gets a local context copy, so changes made below
one directive cannot leak to that directive's later siblings.

## When to Use

Use this pattern after a parser has produced a small immutable tree whose root
and child tuples are already in source order. It fits linters and policy checks
for declarative test specifications, build descriptions, and similar nested
inputs where bindings have lexical, forward-only visibility.

Supply a closed ordered tuple of trusted rules. Their order determines issue
order at each directive. Zero-based tuple paths identify roots and descendants
without depending on directive names or line-number uniqueness.

## Implementation

```python
import re
from collections.abc import Callable, Mapping
from dataclasses import dataclass
from types import MappingProxyType

_MAX_ROOTS = 16
_MAX_DEPTH = 8
_MAX_NODES = 128
_MAX_CHILDREN = 16
_MAX_ARGUMENTS = 8
_MAX_BINDINGS = 8
_MAX_RULES = 16
_MAX_MESSAGES_PER_CALLBACK = 4
_MAX_ISSUES = 256
_MAX_IDENTIFIER_BYTES = 48
_MAX_ARGUMENT_BYTES = 128
_MAX_BINDING_VALUE_BYTES = 256
_MAX_TREE_TEXT_BYTES = 32 * 1024
_MAX_MESSAGE_BYTES = 256
_MAX_CALLBACK_TEXT_BYTES = 32 * 1024
_MAX_SOURCE_LINE = (1 << 31) - 1
_IDENTIFIER = re.compile(r"[a-z][a-z0-9_.-]{0,47}", re.ASCII)


@dataclass(frozen=True, slots=True)
class ContextBinding:
    name: str
    value: str


@dataclass(frozen=True, slots=True)
class Directive:
    name: str
    arguments: tuple[str, ...]
    source_line: int
    bindings: tuple[ContextBinding, ...] = ()
    children: tuple[Directive, ...] = ()


@dataclass(frozen=True, slots=True)
class DirectiveView:
    name: str
    arguments: tuple[str, ...]
    source_line: int
    bindings: tuple[ContextBinding, ...]


@dataclass(frozen=True, slots=True)
class RuleMessage:
    code: str
    text: str


@dataclass(frozen=True, slots=True)
class AuditRule:
    rule_id: str
    check: Callable[
        [DirectiveView, Mapping[str, str], tuple[int, ...]],
        tuple[RuleMessage, ...],
    ]


@dataclass(frozen=True, slots=True)
class AuditIssue:
    rule_id: str
    code: str
    text: str
    directive_name: str
    source_line: int
    scope_path: tuple[int, ...]


def _validated_identifier(value: object, *, field: str) -> tuple[str, int]:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if _IDENTIFIER.fullmatch(value) is None:
        raise ValueError(f"{field} must be a bounded conservative ASCII identifier")
    return value, len(value)


def _validated_text(
    value: object,
    *,
    field: str,
    byte_limit: int,
    allow_empty: bool,
) -> tuple[str, int]:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    try:
        encoded = value.encode("utf-8")
    except UnicodeEncodeError as error:
        raise ValueError(f"{field} must be valid UTF-8 text") from error
    if (not allow_empty and not encoded) or len(encoded) > byte_limit:
        raise ValueError(f"{field} length is outside the supported range")
    if any(not character.isprintable() for character in value):
        raise ValueError(f"{field} must not contain control characters")
    return value, len(encoded)


def _validated_source_line(value: object, *, field: str) -> int:
    if type(value) is not int:
        raise TypeError(f"{field} must be an exact integer")
    if not 1 <= value <= _MAX_SOURCE_LINE:
        raise ValueError(f"{field} is outside the supported range")
    return value


def _validated_rules(value: object) -> tuple[AuditRule, ...]:
    if type(value) is not tuple:
        raise TypeError("rules must be an exact tuple")
    if len(value) > _MAX_RULES:
        raise ValueError("rules exceeds the supported count")

    checked: list[AuditRule] = []
    seen_ids: set[str] = set()
    for index, raw_rule in enumerate(value):
        field = f"rules[{index}]"
        if type(raw_rule) is not AuditRule:
            raise TypeError(f"{field} must be an exact AuditRule")
        rule_id, _ = _validated_identifier(
            raw_rule.rule_id,
            field=f"{field}.rule_id",
        )
        if rule_id in seen_ids:
            raise ValueError("rule IDs must be unique")
        if not callable(raw_rule.check):
            raise TypeError(f"{field}.check must be callable")
        seen_ids.add(rule_id)
        checked.append(AuditRule(rule_id, raw_rule.check))
    return tuple(checked)


def _preflight_tree(value: object) -> tuple[Directive, ...]:
    if type(value) is not tuple:
        raise TypeError("roots must be an exact tuple")
    if len(value) > _MAX_ROOTS:
        raise ValueError("roots exceeds the supported count")

    seen_nodes: set[int] = set()
    active_nodes: set[int] = set()
    node_count = 0
    total_text_bytes = 0

    def add_identifier(raw: object, *, field: str) -> str:
        nonlocal total_text_bytes
        checked, size = _validated_identifier(raw, field=field)
        total_text_bytes += size
        if total_text_bytes > _MAX_TREE_TEXT_BYTES:
            raise ValueError("tree text exceeds the aggregate UTF-8 byte limit")
        return checked

    def add_text(
        raw: object,
        *,
        field: str,
        byte_limit: int,
    ) -> str:
        nonlocal total_text_bytes
        checked, size = _validated_text(
            raw,
            field=field,
            byte_limit=byte_limit,
            allow_empty=True,
        )
        total_text_bytes += size
        if total_text_bytes > _MAX_TREE_TEXT_BYTES:
            raise ValueError("tree text exceeds the aggregate UTF-8 byte limit")
        return checked

    def visit(raw_node: object, *, depth: int, field: str) -> Directive:
        nonlocal node_count
        if type(raw_node) is not Directive:
            raise TypeError(f"{field} must be an exact Directive")
        node_identity = id(raw_node)
        if node_identity in active_nodes:
            raise ValueError("directive tree contains a cycle")
        if node_identity in seen_nodes:
            raise ValueError("directive tree contains a shared node")
        if depth > _MAX_DEPTH:
            raise ValueError("directive tree exceeds the supported depth")

        seen_nodes.add(node_identity)
        active_nodes.add(node_identity)
        node_count += 1
        try:
            if node_count > _MAX_NODES:
                raise ValueError("directive tree exceeds the supported node count")
            name = add_identifier(raw_node.name, field=f"{field}.name")
            source_line = _validated_source_line(
                raw_node.source_line,
                field=f"{field}.source_line",
            )

            if type(raw_node.arguments) is not tuple:
                raise TypeError(f"{field}.arguments must be an exact tuple")
            if len(raw_node.arguments) > _MAX_ARGUMENTS:
                raise ValueError(f"{field}.arguments exceeds the supported count")
            arguments = tuple(
                add_text(
                    argument,
                    field=f"{field}.arguments[{index}]",
                    byte_limit=_MAX_ARGUMENT_BYTES,
                )
                for index, argument in enumerate(raw_node.arguments)
            )

            if type(raw_node.bindings) is not tuple:
                raise TypeError(f"{field}.bindings must be an exact tuple")
            if len(raw_node.bindings) > _MAX_BINDINGS:
                raise ValueError(f"{field}.bindings exceeds the supported count")
            bindings: list[ContextBinding] = []
            binding_names: set[str] = set()
            for index, raw_binding in enumerate(raw_node.bindings):
                binding_field = f"{field}.bindings[{index}]"
                if type(raw_binding) is not ContextBinding:
                    raise TypeError(f"{binding_field} must be an exact ContextBinding")
                binding_name = add_identifier(
                    raw_binding.name,
                    field=f"{binding_field}.name",
                )
                if binding_name in binding_names:
                    raise ValueError(f"{field}.bindings contains a duplicate name")
                binding_names.add(binding_name)
                binding_value = add_text(
                    raw_binding.value,
                    field=f"{binding_field}.value",
                    byte_limit=_MAX_BINDING_VALUE_BYTES,
                )
                bindings.append(ContextBinding(binding_name, binding_value))

            if type(raw_node.children) is not tuple:
                raise TypeError(f"{field}.children must be an exact tuple")
            if len(raw_node.children) > _MAX_CHILDREN:
                raise ValueError(f"{field}.children exceeds the supported count")
            children = tuple(
                visit(
                    child,
                    depth=depth + 1,
                    field=f"{field}.children[{index}]",
                )
                for index, child in enumerate(raw_node.children)
            )
            return Directive(
                name=name,
                arguments=arguments,
                source_line=source_line,
                bindings=tuple(bindings),
                children=children,
            )
        finally:
            active_nodes.remove(node_identity)

    return tuple(visit(root, depth=1, field=f"roots[{index}]") for index, root in enumerate(value))


def _validated_messages(value: object) -> tuple[tuple[RuleMessage, ...], int]:
    if type(value) is not tuple:
        raise TypeError("a rule callback must return an exact tuple")
    if len(value) > _MAX_MESSAGES_PER_CALLBACK:
        raise ValueError("a rule callback returned too many messages")

    checked: list[RuleMessage] = []
    text_bytes = 0
    for index, raw_message in enumerate(value):
        field = f"callback result[{index}]"
        if type(raw_message) is not RuleMessage:
            raise TypeError(f"{field} must be an exact RuleMessage")
        code, code_bytes = _validated_identifier(
            raw_message.code,
            field=f"{field}.code",
        )
        message, message_bytes = _validated_text(
            raw_message.text,
            field=f"{field}.text",
            byte_limit=_MAX_MESSAGE_BYTES,
            allow_empty=False,
        )
        text_bytes += code_bytes + message_bytes
        checked.append(RuleMessage(code, message))
    return tuple(checked), text_bytes


def audit_directive_tree(
    roots: tuple[Directive, ...],
    *,
    rules: tuple[AuditRule, ...],
) -> tuple[AuditIssue, ...]:
    checked_rules = _validated_rules(rules)
    checked_roots = _preflight_tree(roots)
    issues: list[AuditIssue] = []
    callback_text_bytes = 0

    def audit_siblings(
        siblings: tuple[Directive, ...],
        *,
        inherited: Mapping[str, str],
        parent_path: tuple[int, ...],
    ) -> None:
        nonlocal callback_text_bytes
        visible_context = dict(inherited)
        for index, directive in enumerate(siblings):
            scope_path = (*parent_path, index)
            context_snapshot = MappingProxyType(dict(visible_context))
            view = DirectiveView(
                directive.name,
                directive.arguments,
                directive.source_line,
                directive.bindings,
            )
            for rule in checked_rules:
                raw_messages = rule.check(view, context_snapshot, scope_path)
                messages, text_bytes = _validated_messages(raw_messages)
                if callback_text_bytes + text_bytes > _MAX_CALLBACK_TEXT_BYTES:
                    raise ValueError("callback messages exceed the aggregate text limit")
                if len(issues) + len(messages) > _MAX_ISSUES:
                    raise ValueError("audit issues exceed the supported count")
                callback_text_bytes += text_bytes
                issues.extend(
                    AuditIssue(
                        rule_id=rule.rule_id,
                        code=message.code,
                        text=message.text,
                        directive_name=directive.name,
                        source_line=directive.source_line,
                        scope_path=scope_path,
                    )
                    for message in messages
                )

            for binding in directive.bindings:
                visible_context[binding.name] = binding.value
            audit_siblings(
                directive.children,
                inherited=visible_context,
                parent_path=scope_path,
            )

    audit_siblings(checked_roots, inherited=MappingProxyType({}), parent_path=())
    return tuple(issues)
```

## Example

```python


def expect_binding(
    directive: DirectiveView,
    context: Mapping[str, str],
    scope_path: tuple[int, ...],
) -> tuple[RuleMessage, ...]:
    if directive.name != "expect":
        return ()
    binding_name, expected_value = directive.arguments
    observed_value = context.get(binding_name, "<missing>")
    if observed_value == expected_value:
        return ()
    return (
        RuleMessage(
            "binding-mismatch",
            f"{binding_name} expected {expected_value}; observed {observed_value}",
        ),
    )


tree = (
    Directive(
        name="section",
        arguments=(),
        source_line=1,
        bindings=(ContextBinding("mode", "base"),),
        children=(
            Directive(
                name="branch",
                arguments=(),
                source_line=2,
                bindings=(ContextBinding("mode", "child"),),
                children=(
                    Directive(
                        name="expect",
                        arguments=("mode", "base"),
                        source_line=3,
                    ),
                ),
            ),
        ),
    ),
    Directive(
        name="expect",
        arguments=("mode", "child"),
        source_line=4,
    ),
)

issues = audit_directive_tree(
    tree,
    rules=(AuditRule("expected-binding", expect_binding),),
)

assert issues == (
    AuditIssue(
        rule_id="expected-binding",
        code="binding-mismatch",
        text="mode expected base; observed child",
        directive_name="expect",
        source_line=3,
        scope_path=(0, 0, 0),
    ),
    AuditIssue(
        rule_id="expected-binding",
        code="binding-mismatch",
        text="mode expected child; observed base",
        directive_name="expect",
        source_line=4,
        scope_path=(1,),
    ),
)
```

## Trade-offs and Limitations

Preflight takes linear time and memory within limits of 128 nodes, depth eight,
16 roots or children per list, and eight arguments or bindings per directive.
It validates all text and structure before the first callback, including node
identity, so malformed later nodes, cycles, and shared subtrees cannot produce
partial audit activity. Rule, callback-message, aggregate-text, and final-issue
limits bound retained state and callback output.

Traversal is deterministic for deterministic callbacks: preorder source paths,
rule order, and callback tuple order define issue order. Every callback receives
a fresh read-only context snapshot whose backing dictionary is never updated.
Bindings shadow by name, but recursive child updates stay inside that child
list's copied context and never affect a parent sibling.

This helper does no parsing, import or plugin discovery, global registration,
I/O, or shared mutation. Read-only arguments do not sandbox callbacks or make
them safe: rules are trusted terminating, deterministic, side-effect-free
policy functions, and their exceptions propagate to the caller.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Bounded Space-Indented Test Outline into Leaf Paths](parse-a-bounded-space-indented-test-outline-into-leaf-paths.md)
- [Evaluate a Bounded Boolean Rule Tree from Closed JSON](../configuration-serialization/evaluate-a-bounded-boolean-rule-tree-from-closed-json.md)
- [Walk a Tree Recursively with yield from](../python-language/walk-a-tree-recursively-with-yield-from.md)
<!-- catalog:related:end -->
