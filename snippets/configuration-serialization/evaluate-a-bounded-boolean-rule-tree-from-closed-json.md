---
title: "Evaluate a Bounded Boolean Rule Tree from Closed JSON"
snippet_type: pattern
use_cases:
  - configuration
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - evaluate-a-bounded-boolean-tag-expression-with-an-ast-allowlist.md
  - build-a-bounded-ordered-rule-pipeline-from-closed-json-configuration.md
  - parse-a-bounded-component-options-expression.md
---

# Evaluate a Bounded Boolean Rule Tree from Closed JSON

## Idea and Problem

Turn a small JSON tree into immutable Boolean nodes only after validating every operator and predicate argument against a closed trusted registry.

Configuration can choose how trusted predicates are combined without choosing
code to execute. Exactly four node forms provide conjunction, disjunction,
negation, and one named predicate. Parse duplicate-rejecting bounded JSON,
validate the complete tree into frozen nodes, and only then let an evaluator
call explicitly supplied pure predicates.

## When to Use

Use this pattern when a trusted application owns a small predicate vocabulary
but operators need to be arranged as data. It fits feature selection, report
filtering, and other in-memory decisions where the context is already a closed
mapping of bounded scalar values.

Use ordinary Python for rules that do not need data-driven composition. Use a
dedicated policy engine for authorization, obligations, explanations, changing
schemas, untrusted extensions, or cross-service policy distribution. This tree
is deliberately not a formula language and never interprets Python syntax.

## Implementation

```python
import json
import re
from collections.abc import Callable, Mapping
from dataclasses import dataclass
from types import MappingProxyType
from typing import TypeAlias

Scalar: TypeAlias = bool | int | str | None
_VALUE_TYPES = (bool, int, str)
_NAME = re.compile(r"[a-z][a-z0-9_]{0,31}", re.ASCII).fullmatch
_MAX_JSON_BYTES = 16 * 1_024
_MAX_NODES = 128
_MAX_DEPTH = 8
_MAX_FAN_OUT = 16
_MAX_PREDICATES = 32
_MAX_PARAMETERS = 8
_MAX_CONTEXT_VALUES = 32
_MAX_TEXT_LENGTH = 64
_MAX_ABSOLUTE_INTEGER = 1_000_000_000


class RuleConfigurationError(ValueError):
    pass


@dataclass(frozen=True, slots=True)
class ParameterSpec:
    name: str
    value_type: type
    nullable: bool = False


@dataclass(frozen=True, slots=True)
class PredicateDefinition:
    name: str
    parameters: tuple[ParameterSpec, ...]
    evaluate: Callable[[Mapping[str, Scalar], Mapping[str, Scalar]], bool]


@dataclass(frozen=True, slots=True)
class AllNode:
    children: tuple["RuleNode", ...]


@dataclass(frozen=True, slots=True)
class AnyNode:
    children: tuple["RuleNode", ...]


@dataclass(frozen=True, slots=True)
class NotNode:
    child: "RuleNode"


@dataclass(frozen=True, slots=True)
class PredicateNode:
    name: str
    arguments: tuple[tuple[str, Scalar], ...]


RuleNode: TypeAlias = AllNode | AnyNode | NotNode | PredicateNode


def _name(value: object, *, field: str) -> str:
    if type(value) is not str or _NAME(value) is None:
        raise RuleConfigurationError(f"{field} must be a conservative ASCII identifier")
    return value


def _validated_scalar(
    value: object,
    *,
    value_type: type,
    nullable: bool,
    field: str,
) -> Scalar:
    if value is None:
        if not nullable:
            raise RuleConfigurationError(f"{field} does not allow null")
        return None
    if type(value) is not value_type:
        raise RuleConfigurationError(f"{field} has the wrong exact type")
    if type(value) is int and abs(value) > _MAX_ABSOLUTE_INTEGER:
        raise RuleConfigurationError(f"{field} integer exceeds the supported range")
    if type(value) is str and not 1 <= len(value) <= _MAX_TEXT_LENGTH:
        raise RuleConfigurationError(f"{field} text length is outside its limit")
    return value


def _validated_registry(
    value: object,
) -> dict[str, PredicateDefinition]:
    if type(value) is not tuple:
        raise TypeError("registry must be an exact tuple")
    if not 1 <= len(value) <= _MAX_PREDICATES:
        raise RuleConfigurationError("predicate count is outside the supported range")

    registry: dict[str, PredicateDefinition] = {}
    for definition in value:
        if type(definition) is not PredicateDefinition:
            raise TypeError("registry must contain exact PredicateDefinition values")
        name = _name(definition.name, field="predicate name")
        if name in registry:
            raise RuleConfigurationError("predicate names must be unique")
        if type(definition.parameters) is not tuple:
            raise TypeError("predicate parameters must be an exact tuple")
        if len(definition.parameters) > _MAX_PARAMETERS:
            raise RuleConfigurationError("predicate parameter count exceeds its limit")
        parameter_names: set[str] = set()
        for parameter in definition.parameters:
            if type(parameter) is not ParameterSpec:
                raise TypeError("parameters must contain exact ParameterSpec values")
            parameter_name = _name(parameter.name, field="parameter name")
            if parameter_name in parameter_names:
                raise RuleConfigurationError("parameter names must be unique")
            parameter_names.add(parameter_name)
            if not any(parameter.value_type is kind for kind in _VALUE_TYPES):
                raise TypeError("parameter value_type must be bool, int, or str")
            if type(parameter.nullable) is not bool:
                raise TypeError("parameter nullable must be an exact boolean")
        if not callable(definition.evaluate):
            raise TypeError("predicate evaluate must be callable")
        registry[name] = definition
    return registry


def _decode_json(document: object) -> object:
    if type(document) is not str:
        raise TypeError("document must be an exact string")
    try:
        encoded = document.encode("utf-8")
    except UnicodeEncodeError as error:
        raise RuleConfigurationError("document must be valid Unicode text") from error
    if len(encoded) > _MAX_JSON_BYTES:
        raise RuleConfigurationError("JSON document exceeds the byte limit")

    def object_without_duplicates(
        pairs: list[tuple[str, object]],
    ) -> dict[str, object]:
        result: dict[str, object] = {}
        for key, value in pairs:
            if key in result:
                raise RuleConfigurationError("JSON object keys must be unique")
            result[key] = value
        return result

    def reject_constant(_value: str) -> object:
        raise RuleConfigurationError("non-finite JSON numbers are forbidden")

    try:
        return json.loads(
            document,
            object_pairs_hook=object_without_duplicates,
            parse_constant=reject_constant,
        )
    except RuleConfigurationError:
        raise
    except (UnicodeError, ValueError) as error:
        raise RuleConfigurationError("document is not valid JSON") from error


def load_boolean_rule_tree(
    document: str,
    registry: tuple[PredicateDefinition, ...],
) -> tuple[RuleNode, Mapping[str, PredicateDefinition]]:
    definitions = _validated_registry(registry)
    decoded = _decode_json(document)
    node_count = 0

    def parse_node(value: object, *, depth: int) -> RuleNode:
        nonlocal node_count
        node_count += 1
        if node_count > _MAX_NODES:
            raise RuleConfigurationError("rule tree exceeds the node limit")
        if depth > _MAX_DEPTH:
            raise RuleConfigurationError("rule tree exceeds the depth limit")
        if type(value) is not dict:
            raise RuleConfigurationError("every rule node must be an object")

        keys = set(value)
        if keys in ({"all"}, {"any"}):
            operator = next(iter(keys))
            children = value[operator]
            if type(children) is not list:
                raise RuleConfigurationError(f"{operator} must contain an array")
            if not 1 <= len(children) <= _MAX_FAN_OUT:
                raise RuleConfigurationError(f"{operator} fan-out is outside its limit")
            parsed = tuple(parse_node(child, depth=depth + 1) for child in children)
            return AllNode(parsed) if operator == "all" else AnyNode(parsed)
        if keys == {"not"}:
            return NotNode(parse_node(value["not"], depth=depth + 1))
        if keys != {"predicate", "arguments"}:
            raise RuleConfigurationError("rule node has an unsupported shape")

        predicate_name = _name(value["predicate"], field="predicate reference")
        definition = definitions.get(predicate_name)
        if definition is None:
            raise RuleConfigurationError("rule tree references an unknown predicate")
        raw_arguments = value["arguments"]
        if type(raw_arguments) is not dict:
            raise RuleConfigurationError("predicate arguments must be an object")
        expected_names = tuple(parameter.name for parameter in definition.parameters)
        if set(raw_arguments) != set(expected_names):
            raise RuleConfigurationError("predicate arguments do not match its schema")
        arguments = tuple(
            (
                parameter.name,
                _validated_scalar(
                    raw_arguments[parameter.name],
                    value_type=parameter.value_type,
                    nullable=parameter.nullable,
                    field=f"argument {parameter.name}",
                ),
            )
            for parameter in definition.parameters
        )
        return PredicateNode(predicate_name, arguments)

    root = parse_node(decoded, depth=1)
    return root, MappingProxyType(definitions)


def evaluate_boolean_rule_tree(
    root: RuleNode,
    registry: Mapping[str, PredicateDefinition],
    context: Mapping[str, Scalar],
) -> bool:
    if not isinstance(context, Mapping) or len(context) > _MAX_CONTEXT_VALUES:
        raise RuleConfigurationError("context must be a bounded mapping")
    context_snapshot: dict[str, Scalar] = {}
    for key, value in context.items():
        name = _name(key, field="context key")
        if value is not None and not any(type(value) is kind for kind in _VALUE_TYPES):
            raise RuleConfigurationError(f"context value {name} has an unsupported type")
        context_snapshot[name] = _validated_scalar(
            value,
            value_type=type(value) if value is not None else str,
            nullable=value is None,
            field=f"context value {name}",
        )
    frozen_context = MappingProxyType(context_snapshot)

    def evaluate(node: RuleNode) -> bool:
        if type(node) is AllNode:
            return all(evaluate(child) for child in node.children)
        if type(node) is AnyNode:
            return any(evaluate(child) for child in node.children)
        if type(node) is NotNode:
            return not evaluate(node.child)
        if type(node) is not PredicateNode:
            raise TypeError("root must contain validated rule nodes")
        definition = registry.get(node.name)
        if definition is None:
            raise RuleConfigurationError("validated predicate is absent from registry")
        result = definition.evaluate(
            frozen_context,
            MappingProxyType(dict(node.arguments)),
        )
        if type(result) is not bool:
            raise TypeError("predicate must return an exact boolean")
        return result

    return evaluate(root)
```

## Example

```python
def number_at_least(
    context: Mapping[str, Scalar],
    arguments: Mapping[str, Scalar],
) -> bool:
    value = context.get(arguments["field"])
    return type(value) is int and value >= arguments["minimum"]


def text_equals(
    context: Mapping[str, Scalar],
    arguments: Mapping[str, Scalar],
) -> bool:
    return context.get(arguments["field"]) == arguments["expected"]


registry = (
    PredicateDefinition(
        "number_at_least",
        (ParameterSpec("field", str), ParameterSpec("minimum", int)),
        number_at_least,
    ),
    PredicateDefinition(
        "text_equals",
        (ParameterSpec("field", str), ParameterSpec("expected", str)),
        text_equals,
    ),
)
document = """
{
  "all": [
    {"predicate": "number_at_least", "arguments": {"field": "score", "minimum": 7}},
    {"not": {"predicate": "text_equals", "arguments": {"field": "state", "expected": "paused"}}}
  ]
}
"""
tree, definitions = load_boolean_rule_tree(document, registry)
accepted = evaluate_boolean_rule_tree(
    tree,
    definitions,
    {"score": 9, "state": "active"},
)
rejected = evaluate_boolean_rule_tree(
    tree,
    definitions,
    {"score": 9, "state": "paused"},
)

try:
    load_boolean_rule_tree(
        '{"all": [], "all": []}',
        registry,
    )
except RuleConfigurationError:
    duplicate_key_rejected = True
else:
    duplicate_key_rejected = False

assert (accepted, rejected, duplicate_key_rejected) == (True, False, True)
```

## Trade-offs and Limitations

Loading takes linear work in at most 128 nodes and produces immutable tuples,
but evaluation may short-circuit and therefore not call every predicate. The
registry callbacks are trusted application code; closed registration prevents
JSON from selecting arbitrary callables but does not make a slow, stateful, or
incorrect callback safe. Keep callbacks pure and cheap.

The scalar context and exact parameter types intentionally exclude nested
objects and numeric coercion. This evaluator provides no authorization model,
explanation graph, caching, asynchronous predicates, side-effect rollback, or
policy version negotiation. Changing a predicate's meaning still requires an
application-level configuration migration.

## Related Snippets

<!-- catalog:related:start -->
- [Evaluate a Bounded Boolean Tag Expression with an AST Allowlist](evaluate-a-bounded-boolean-tag-expression-with-an-ast-allowlist.md)
- [Build a Bounded Ordered Rule Pipeline from Closed JSON Configuration](build-a-bounded-ordered-rule-pipeline-from-closed-json-configuration.md)
- [Parse a Bounded Component Options Expression](parse-a-bounded-component-options-expression.md)
<!-- catalog:related:end -->
