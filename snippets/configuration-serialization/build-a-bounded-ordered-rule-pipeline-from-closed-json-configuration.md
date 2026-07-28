---
title: "Build a Bounded Ordered Rule Pipeline from Closed JSON Configuration"
snippet_type: pattern
use_cases:
  - configuration
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../python-language/dispatch-named-strategies-with-an-explicit-function-mapping.md
  - ../python-language/load-bounded-trusted-extension-factories-by-one-entry-point.md
  - migrate-one-bounded-json-record-to-a-current-version.md
---

# Build a Bounded Ordered Rule Pipeline from Closed JSON Configuration

## Idea and Problem

Compile duplicate-free bounded JSON into an immutable ordered pipeline whose versioned rules and scalar parameters come from one explicit trusted registry.

A configuration document can select several transformations without becoming a
plugin-discovery mechanism. Rejecting duplicate object keys during decoding,
validating the entire document before returning a pipeline, and storing only
immutable scalar parameters makes ordering and failure boundaries explicit.
Execution can then validate each immutable intermediate state and identify a
failed step without exposing state values.

## When to Use

Use this pattern when an application owns a small closed tuple of trusted,
pure, resource-free transformations. Each versioned rule identifier declares
an exact scalar parameter schema, and every configured step has its own stable
identifier. The state is a bounded tuple of unique key/value pairs containing
only immutable JSON scalars.

Do not use this as a sandbox for untrusted Python. Callbacks are ordinary
trusted callables and can still perform side effects if they violate the
contract. Resource-owning extensions need a separate lifecycle design.

## Implementation

```python
import json
import math
import re
from collections.abc import Callable
from dataclasses import dataclass


type JsonScalar = None | bool | int | float | str
type FrozenValues = tuple[tuple[str, JsonScalar], ...]
type RuleTransform = Callable[[FrozenValues, FrozenValues], FrozenValues]

_MAX_JSON_BYTES = 16 * 1_024
_MAX_DEPTH = 8
_MAX_NODES = 512
_MAX_STEPS = 32
_MAX_RULES = 64
_MAX_PARAMETERS = 16
_MAX_STATE_FIELDS = 64
_MAX_TEXT_BYTES = 256
_MAX_INTEGER = (1 << 63) - 1
_TOKEN = re.compile(r"[a-z][a-z0-9_-]{0,63}", re.ASCII)
_RULE_ID = re.compile(r"[a-z][a-z0-9_-]{0,31}@v[1-9][0-9]{0,2}", re.ASCII)
_KINDS = frozenset({"boolean", "integer", "number", "text"})


@dataclass(frozen=True, slots=True)
class RuleParameter:
    name: str
    kind: str


@dataclass(frozen=True, slots=True)
class RuleDefinition:
    rule_id: str
    parameters: tuple[RuleParameter, ...]
    transform: RuleTransform


@dataclass(frozen=True, slots=True)
class CompiledRuleStep:
    step_id: str
    rule_id: str
    parameters: FrozenValues
    transform: RuleTransform


@dataclass(frozen=True, slots=True)
class RulePipeline:
    steps: tuple[CompiledRuleStep, ...]


class RuleStepError(RuntimeError):
    def __init__(self, index: int, step_id: str, error_type: str) -> None:
        self.index = index
        self.step_id = step_id
        self.error_type = error_type
        super().__init__(f"step {index} ({step_id}) failed with {error_type}")


def _reject_constant(value: str) -> None:
    raise ValueError(f"non-finite JSON number is forbidden: {value}")


def _parse_integer(value: str) -> int:
    if len(value.lstrip("-")) > 19:
        raise ValueError("JSON integer is too large")
    result = int(value)
    if not -_MAX_INTEGER - 1 <= result <= _MAX_INTEGER:
        raise ValueError("JSON integer is outside the signed 64-bit range")
    return result


def _parse_number(value: str) -> float:
    if len(value) > 64:
        raise ValueError("JSON number is too large")
    result = float(value)
    if not math.isfinite(result):
        raise ValueError("JSON number must be finite")
    return result


def _unique_object(pairs: list[tuple[str, object]]) -> dict[str, object]:
    result: dict[str, object] = {}
    for key, value in pairs:
        if key in result:
            raise ValueError("duplicate JSON object key")
        result[key] = value
    return result


def _measure_json(value: object, depth: int = 1) -> int:
    if depth > _MAX_DEPTH:
        raise ValueError("JSON depth limit exceeded")
    nodes = 1
    if type(value) is dict:
        for key, child in value.items():
            if len(key.encode("utf-8")) > _MAX_TEXT_BYTES or "\x00" in key:
                raise ValueError("JSON key is too large or contains NUL")
            nodes += _measure_json(child, depth + 1)
    elif type(value) is list:
        for child in value:
            nodes += _measure_json(child, depth + 1)
    elif type(value) is str:
        if len(value.encode("utf-8")) > _MAX_TEXT_BYTES or "\x00" in value:
            raise ValueError("JSON string is too large or contains NUL")
    elif value is not None and type(value) not in (bool, int, float):
        raise TypeError("unsupported decoded JSON value")
    if nodes > _MAX_NODES:
        raise ValueError("JSON node limit exceeded")
    return nodes


def _token(value: object, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if _TOKEN.fullmatch(value) is None:
        raise ValueError(f"{field} is invalid")
    return value


def _scalar(value: object, kind: str, field: str) -> JsonScalar:
    if kind == "boolean" and type(value) is bool:
        return value
    if kind == "integer" and type(value) is int:
        if -_MAX_INTEGER - 1 <= value <= _MAX_INTEGER:
            return value
    elif kind == "number" and type(value) in (int, float):
        if math.isfinite(value):
            return value
    elif kind == "text" and type(value) is str:
        if len(value.encode("utf-8")) <= _MAX_TEXT_BYTES and "\x00" not in value:
            return value
    raise TypeError(f"{field} does not match parameter kind {kind}")


def _registry(
    value: object,
) -> dict[str, tuple[tuple[RuleParameter, ...], RuleTransform]]:
    if type(value) is not tuple:
        raise TypeError("registry must be an exact tuple")
    if not 1 <= len(value) <= _MAX_RULES:
        raise ValueError(f"registry must contain 1..{_MAX_RULES} definitions")

    result: dict[str, tuple[tuple[RuleParameter, ...], RuleTransform]] = {}
    for definition in value:
        if type(definition) is not RuleDefinition:
            raise TypeError("registry must contain exact RuleDefinition records")
        if type(definition.rule_id) is not str or _RULE_ID.fullmatch(definition.rule_id) is None:
            raise ValueError("rule_id must be an explicit versioned identifier")
        if definition.rule_id in result:
            raise ValueError("duplicate rule_id")
        if type(definition.parameters) is not tuple:
            raise TypeError("rule parameters must be an exact tuple")
        if len(definition.parameters) > _MAX_PARAMETERS:
            raise ValueError("parameter-schema limit exceeded")
        if not callable(definition.transform):
            raise TypeError("rule transform must be callable")

        names: set[str] = set()
        for parameter in definition.parameters:
            if type(parameter) is not RuleParameter:
                raise TypeError("parameter schema contains an invalid record")
            name = _token(parameter.name, "parameter name")
            if name in names:
                raise ValueError("duplicate parameter name")
            names.add(name)
            if type(parameter.kind) is not str or parameter.kind not in _KINDS:
                raise ValueError("unsupported parameter kind")
        result[definition.rule_id] = (definition.parameters, definition.transform)
    return result


def build_rule_pipeline(
    text: str,
    registry: tuple[RuleDefinition, ...],
) -> RulePipeline:
    if type(text) is not str:
        raise TypeError("text must be an exact string")
    if len(text.encode("utf-8")) > _MAX_JSON_BYTES:
        raise ValueError("JSON byte limit exceeded")
    try:
        document = json.loads(
            text,
            object_pairs_hook=_unique_object,
            parse_constant=_reject_constant,
            parse_int=_parse_integer,
            parse_float=_parse_number,
        )
    except (json.JSONDecodeError, RecursionError) as error:
        raise ValueError("invalid bounded JSON document") from error
    _measure_json(document)
    definitions = _registry(registry)

    if type(document) is not dict or set(document) != {"version", "steps"}:
        raise ValueError("document must contain exactly version and steps")
    if type(document["version"]) is not int or document["version"] != 1:
        raise ValueError("unsupported configuration version")
    steps = document["steps"]
    if type(steps) is not list or not 1 <= len(steps) <= _MAX_STEPS:
        raise ValueError(f"steps must contain 1..{_MAX_STEPS} entries")

    seen_steps: set[str] = set()
    compiled: list[CompiledRuleStep] = []
    for raw_step in steps:
        if type(raw_step) is not dict or set(raw_step) != {"id", "rule", "params"}:
            raise ValueError("each step must contain exactly id, rule, and params")
        step_id = _token(raw_step["id"], "step id")
        if step_id in seen_steps:
            raise ValueError("duplicate step id")
        seen_steps.add(step_id)

        rule_id = raw_step["rule"]
        if type(rule_id) is not str or _RULE_ID.fullmatch(rule_id) is None:
            raise ValueError("configured rule identifier is invalid")
        try:
            schema, transform = definitions[rule_id]
        except KeyError:
            raise ValueError("configured rule is not in the closed registry") from None
        params = raw_step["params"]
        if type(params) is not dict or len(params) > _MAX_PARAMETERS:
            raise ValueError("params must be a bounded object")
        if set(params) != {parameter.name for parameter in schema}:
            raise ValueError("configured parameters do not match the closed schema")

        normalized = tuple(
            (
                parameter.name,
                _scalar(params[parameter.name], parameter.kind, "parameter"),
            )
            for parameter in schema
        )
        compiled.append(CompiledRuleStep(step_id, rule_id, normalized, transform))
    return RulePipeline(tuple(compiled))


def _state(value: object) -> FrozenValues:
    if type(value) is not tuple:
        raise TypeError("state must be an exact tuple")
    if len(value) > _MAX_STATE_FIELDS:
        raise ValueError("state field limit exceeded")
    result: list[tuple[str, JsonScalar]] = []
    names: set[str] = set()
    for pair in value:
        if type(pair) is not tuple or len(pair) != 2:
            raise TypeError("state entries must be exact name/value tuples")
        name = _token(pair[0], "state name")
        if name in names:
            raise ValueError("duplicate state name")
        names.add(name)
        raw = pair[1]
        if raw is None or type(raw) is bool:
            scalar = raw
        elif type(raw) is int:
            scalar = _scalar(raw, "integer", "state value")
        elif type(raw) is float:
            scalar = _scalar(raw, "number", "state value")
        elif type(raw) is str:
            scalar = _scalar(raw, "text", "state value")
        else:
            raise TypeError("state contains a non-scalar value")
        result.append((name, scalar))
    return tuple(result)


def _pipeline_steps(value: object) -> tuple[CompiledRuleStep, ...]:
    if type(value) is not RulePipeline or type(value.steps) is not tuple:
        raise TypeError("pipeline must contain an exact step tuple")
    if not 1 <= len(value.steps) <= _MAX_STEPS:
        raise ValueError(f"pipeline must contain 1..{_MAX_STEPS} steps")

    seen: set[str] = set()
    for step in value.steps:
        if type(step) is not CompiledRuleStep:
            raise TypeError("pipeline contains an invalid step record")
        step_id = _token(step.step_id, "step id")
        if step_id in seen:
            raise ValueError("pipeline contains a duplicate step id")
        seen.add(step_id)
        if type(step.rule_id) is not str or _RULE_ID.fullmatch(step.rule_id) is None:
            raise ValueError("pipeline contains an invalid rule identifier")
        if len(_state(step.parameters)) > _MAX_PARAMETERS:
            raise ValueError("pipeline parameter limit exceeded")
        if not callable(step.transform):
            raise TypeError("pipeline transform must be callable")
    return value.steps


def run_rule_pipeline(pipeline: RulePipeline, state: FrozenValues) -> FrozenValues:
    steps = _pipeline_steps(pipeline)
    current = _state(state)
    for index, step in enumerate(steps):
        try:
            candidate = step.transform(current, step.parameters)
            current = _state(candidate)
        except Exception as error:
            raise RuleStepError(
                index,
                step.step_id,
                type(error).__name__[:64],
            ) from None
    return current
```

## Example

```python
def add_offset(state: FrozenValues, params: FrozenValues) -> FrozenValues:
    values = dict(state)
    values["score"] += dict(params)["amount"]
    return tuple(sorted(values.items()))


def apply_ceiling(state: FrozenValues, params: FrozenValues) -> FrozenValues:
    values = dict(state)
    values["score"] = min(values["score"], dict(params)["maximum"])
    return tuple(sorted(values.items()))


registry = (
    RuleDefinition(
        "offset@v1",
        (RuleParameter("amount", "integer"),),
        add_offset,
    ),
    RuleDefinition(
        "ceiling@v1",
        (RuleParameter("maximum", "integer"),),
        apply_ceiling,
    ),
)
configuration = """
{
  "version": 1,
  "steps": [
    {"id": "increase", "rule": "offset@v1", "params": {"amount": 3}},
    {"id": "cap", "rule": "ceiling@v1", "params": {"maximum": 5}}
  ]
}
"""
original = (("score", 4),)

pipeline = build_rule_pipeline(configuration, registry)
result = run_rule_pipeline(pipeline, original)

assert result == (("score", 5),) and original == (("score", 4),)
```

## Trade-offs and Limitations

Compilation is bounded by 16 KiB, depth 8, 512 decoded nodes, 32 steps, and 64
registry definitions. Registry lookup and validation are linear at these small
limits; execution is `O(s)` plus trusted transform work and validation of at
most 64 state fields after every step. Duplicate object keys fail during JSON
decoding, and no transform runs until the complete document and registry pass.

The immutable tuple contract prevents transforms from mutating input state or
parameters, but it cannot prove that a trusted callback is pure, deterministic,
fast, or side-effect free. Failure stops at the first step and exposes only its
bounded index, identifier, and exception type; there is no rollback. The code
performs no imports, discovery, global registration, `eval`, factory calls,
installation, resource management, or execution of configuration-supplied code.

## Related Snippets

<!-- catalog:related:start -->
- [Dispatch Named Strategies with an Explicit Function Mapping](../python-language/dispatch-named-strategies-with-an-explicit-function-mapping.md)
- [Load Bounded Trusted Extension Factories by One Entry Point](../python-language/load-bounded-trusted-extension-factories-by-one-entry-point.md)
- [Migrate One Bounded JSON Record to a Current Version](migrate-one-bounded-json-record-to-a-current-version.md)
<!-- catalog:related:end -->
