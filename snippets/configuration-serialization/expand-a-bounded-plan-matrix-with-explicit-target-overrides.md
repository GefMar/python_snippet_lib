---
title: "Expand a Bounded Plan Matrix with Explicit Target Overrides"
snippet_type: algorithm
use_cases:
  - configuration
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - parse-a-bounded-flat-placeholder-template.md
  - expand-bounded-nested-brace-alternatives.md
  - merge-nested-mappings-without-mutating-inputs.md
---

# Expand a Bounded Plan Matrix with Explicit Target Overrides

## Idea and Problem

Expand closed immutable templates into ordered plans while applying an exact target override only after base options and axis bindings.

Each target pattern uses a deliberately flat `{axis_name}` grammar. Repeated
placeholders select the same axis value and add that axis to the Cartesian
product only once, at its first appearance. A complete preflight validates the
closed option names, computes the product count, renders collision-free target
identifiers, and charges a logical output-byte budget before any `Plan` objects
are returned.

## When to Use

Use this algorithm when a trusted configuration layer defines a small matrix of
deployment, build, or test plans. Axis values must already be exact canonical
ASCII strings; rejecting integers, booleans, and other coercible values prevents
several distinct inputs from silently rendering as the same text.

The precedence is explicit: base options are copied first, axis bindings replace
same-named base options next, and the selected target override replaces them
last. An omitted override key inherits the earlier value, while an explicit
`None` replaces it with null. Override keys cannot introduce undeclared options,
and override targets must name a generated target exactly.

## Implementation

```python
import itertools
import math
import re
from collections.abc import Mapping
from dataclasses import dataclass, field
from enum import StrEnum
from types import MappingProxyType
from typing import Never

_MAX_TEMPLATES = 32
_MAX_AXES = 8
_MAX_AXIS_VALUES = 32
_MAX_OPTIONS = 64
_MAX_OVERRIDES = 512
_MAX_PATTERN_BYTES = 256
_MAX_TARGET_BYTES = 192
_MAX_INPUT_NODES = 4_096
_MAX_VALUE_DEPTH = 16
_MAX_VALUE_TEXT_BYTES = 128 * 1_024
_MAX_INTEGER_BITS = 256
_MAX_PLANS = 4_096
_MAX_AGGREGATE_OUTPUT_BYTES = 4 * 1_024 * 1_024

_NAME = re.compile(r"[a-z][a-z0-9_]{0,31}", re.ASCII)
_AXIS_VALUE = re.compile(r"[a-z0-9][a-z0-9_-]{0,31}", re.ASCII)
_TARGET = re.compile(r"[a-z][a-z0-9._-]{0,191}", re.ASCII)
_LITERAL = re.compile(r"[a-z0-9._-]*", re.ASCII)


class MatrixErrorCode(StrEnum):
    TEMPLATE = "template"
    AXIS = "axis"
    OPTION = "option"
    OVERRIDE = "override"
    LIMIT = "limit"
    COLLISION = "collision"


class PlanMatrixError(ValueError):
    def __init__(self, code: MatrixErrorCode) -> None:
        self.code = code
        super().__init__(code.value)


def _fail(code: MatrixErrorCode) -> Never:
    raise PlanMatrixError(code)


@dataclass(frozen=True, slots=True)
class FrozenObject:
    items: tuple[tuple[str, FrozenValue], ...]


type FrozenScalar = None | bool | int | float | str
type FrozenValue = FrozenScalar | tuple[FrozenValue, ...] | FrozenObject


@dataclass(frozen=True, slots=True)
class LiteralToken:
    text: str


@dataclass(frozen=True, slots=True)
class AxisToken:
    name: str


type PatternToken = LiteralToken | AxisToken


@dataclass(frozen=True, slots=True)
class Axis:
    name: str
    values: tuple[str, ...]


@dataclass(frozen=True, slots=True)
class Plan:
    template_id: str
    target_id: str
    options: FrozenObject


@dataclass(slots=True)
class _ValueBudget:
    nodes: int = 0
    text_bytes: int = 0
    seen_containers: set[int] = field(default_factory=set)


def _add_text(text: str, budget: _ValueBudget, code: MatrixErrorCode) -> None:
    try:
        size = len(text.encode("utf-8"))
    except UnicodeEncodeError:
        _fail(code)
    if size > _MAX_VALUE_TEXT_BYTES - budget.text_bytes:
        _fail(code)
    budget.text_bytes += size


def _freeze_value(
    value: object,
    *,
    depth: int,
    budget: _ValueBudget,
    code: MatrixErrorCode,
) -> FrozenValue:
    budget.nodes += 1
    if budget.nodes > _MAX_INPUT_NODES or depth > _MAX_VALUE_DEPTH:
        _fail(code)

    if value is None or type(value) is bool:
        return value
    if type(value) is int:
        if value.bit_length() > _MAX_INTEGER_BITS:
            _fail(code)
        return value
    if type(value) is float:
        if not math.isfinite(value):
            _fail(code)
        return value
    if type(value) is str:
        _add_text(value, budget, code)
        return value
    if type(value) not in (list, dict):
        _fail(code)

    identity = id(value)
    if identity in budget.seen_containers:
        _fail(code)
    budget.seen_containers.add(identity)

    if type(value) is list:
        return tuple(
            _freeze_value(
                item,
                depth=depth + 1,
                budget=budget,
                code=code,
            )
            for item in value
        )

    if len(value) > _MAX_OPTIONS:
        _fail(code)
    items: list[tuple[str, FrozenValue]] = []
    for key, item in value.items():
        if type(key) is not str or _NAME.fullmatch(key) is None:
            _fail(code)
        _add_text(key, budget, code)
        items.append(
            (
                key,
                _freeze_value(
                    item,
                    depth=depth + 1,
                    budget=budget,
                    code=code,
                ),
            )
        )
    return FrozenObject(tuple(items))


def _freeze_options(
    options: object,
    *,
    budget: _ValueBudget,
    code: MatrixErrorCode,
) -> FrozenObject:
    if type(options) is not dict or len(options) > _MAX_OPTIONS:
        _fail(code)
    frozen = _freeze_value(options, depth=1, budget=budget, code=code)
    assert isinstance(frozen, FrozenObject)
    return frozen


def _parse_pattern(pattern: object) -> tuple[tuple[PatternToken, ...], tuple[str, ...]]:
    if type(pattern) is not str:
        _fail(MatrixErrorCode.TEMPLATE)
    try:
        encoded = pattern.encode("ascii")
    except UnicodeEncodeError:
        _fail(MatrixErrorCode.TEMPLATE)
    if not 1 <= len(encoded) <= _MAX_PATTERN_BYTES:
        _fail(MatrixErrorCode.TEMPLATE)

    tokens: list[PatternToken] = []
    axis_order: list[str] = []
    seen_axes: set[str] = set()
    literal_start = 0
    index = 0
    while index < len(pattern):
        character = pattern[index]
        if character == "}":
            _fail(MatrixErrorCode.TEMPLATE)
        if character != "{":
            index += 1
            continue

        literal = pattern[literal_start:index]
        if _LITERAL.fullmatch(literal) is None:
            _fail(MatrixErrorCode.TEMPLATE)
        if literal:
            tokens.append(LiteralToken(literal))

        close = pattern.find("}", index + 1)
        if close == -1:
            _fail(MatrixErrorCode.TEMPLATE)
        name = pattern[index + 1 : close]
        if _NAME.fullmatch(name) is None:
            _fail(MatrixErrorCode.TEMPLATE)
        tokens.append(AxisToken(name))
        if name not in seen_axes:
            seen_axes.add(name)
            axis_order.append(name)
        index = close + 1
        literal_start = index

    literal = pattern[literal_start:]
    if _LITERAL.fullmatch(literal) is None:
        _fail(MatrixErrorCode.TEMPLATE)
    if literal:
        tokens.append(LiteralToken(literal))
    if not axis_order:
        _fail(MatrixErrorCode.TEMPLATE)
    return tuple(tokens), tuple(axis_order)


@dataclass(frozen=True, slots=True, init=False)
class PlanTemplate:
    template_id: str
    target_pattern: str
    axes: Mapping[str, tuple[str, ...]]
    base_options: FrozenObject
    target_overrides: Mapping[str, FrozenObject]
    _tokens: tuple[PatternToken, ...]
    _axis_order: tuple[str, ...]

    def __init__(
        self,
        *,
        template_id: str,
        target_pattern: str,
        axes: tuple[Axis, ...],
        base_options: dict[str, object],
        target_overrides: dict[str, dict[str, object]],
    ) -> None:
        if type(template_id) is not str or _NAME.fullmatch(template_id) is None:
            _fail(MatrixErrorCode.TEMPLATE)
        tokens, axis_order = _parse_pattern(target_pattern)

        if type(axes) is not tuple or not 1 <= len(axes) <= _MAX_AXES:
            _fail(MatrixErrorCode.AXIS)
        axis_map: dict[str, tuple[str, ...]] = {}
        for axis in axes:
            if type(axis) is not Axis:
                _fail(MatrixErrorCode.AXIS)
            if type(axis.name) is not str or _NAME.fullmatch(axis.name) is None:
                _fail(MatrixErrorCode.AXIS)
            if axis.name in axis_map or type(axis.values) is not tuple:
                _fail(MatrixErrorCode.AXIS)
            if not 1 <= len(axis.values) <= _MAX_AXIS_VALUES:
                _fail(MatrixErrorCode.AXIS)
            if any(
                type(value) is not str or _AXIS_VALUE.fullmatch(value) is None
                for value in axis.values
            ):
                _fail(MatrixErrorCode.AXIS)
            if len(axis.values) != len(set(axis.values)):
                _fail(MatrixErrorCode.AXIS)
            axis_map[axis.name] = tuple(axis.values)
        if set(axis_map) != set(axis_order):
            _fail(MatrixErrorCode.AXIS)

        budget = _ValueBudget()
        frozen_base = _freeze_options(
            base_options,
            budget=budget,
            code=MatrixErrorCode.OPTION,
        )
        allowed_options = {name for name, _value in frozen_base.items}
        allowed_options.update(axis_order)

        if type(target_overrides) is not dict or len(target_overrides) > _MAX_OVERRIDES:
            _fail(MatrixErrorCode.OVERRIDE)
        frozen_overrides: dict[str, FrozenObject] = {}
        for target_id, options in target_overrides.items():
            if type(target_id) is not str or _TARGET.fullmatch(target_id) is None:
                _fail(MatrixErrorCode.OVERRIDE)
            frozen_options = _freeze_options(
                options,
                budget=budget,
                code=MatrixErrorCode.OVERRIDE,
            )
            if any(name not in allowed_options for name, _value in frozen_options.items):
                _fail(MatrixErrorCode.OVERRIDE)
            frozen_overrides[target_id] = frozen_options

        object.__setattr__(self, "template_id", template_id)
        object.__setattr__(self, "target_pattern", target_pattern)
        object.__setattr__(self, "axes", MappingProxyType(axis_map))
        object.__setattr__(self, "base_options", frozen_base)
        object.__setattr__(
            self,
            "target_overrides",
            MappingProxyType(frozen_overrides),
        )
        object.__setattr__(self, "_tokens", tokens)
        object.__setattr__(self, "_axis_order", axis_order)


def _render_target(template: PlanTemplate, values: Mapping[str, str]) -> str:
    target_id = "".join(
        token.text if isinstance(token, LiteralToken) else values[token.name]
        for token in template._tokens
    )
    if len(target_id) > _MAX_TARGET_BYTES or _TARGET.fullmatch(target_id) is None:
        _fail(MatrixErrorCode.TEMPLATE)
    return target_id


def _merge_options(
    template: PlanTemplate,
    values: Mapping[str, str],
    override: FrozenObject | None,
) -> FrozenObject:
    merged = dict(template.base_options.items)
    for name in template._axis_order:
        merged[name] = values[name]
    if override is not None:
        for name, value in override.items:
            merged[name] = value
    return FrozenObject(tuple(merged.items()))


def _logical_size(value: FrozenValue) -> int:
    if value is None:
        return 4
    if type(value) is bool:
        return 4 if value else 5
    if type(value) is int:
        return len(str(value))
    if type(value) is float:
        return len(repr(value))
    if type(value) is str:
        return len(value.encode("utf-8"))
    if type(value) is tuple:
        return 2 + sum(1 + _logical_size(item) for item in value)
    return 2 + sum(
        len(name.encode("ascii")) + 2 + _logical_size(item) for name, item in value.items
    )


def expand_plan_matrix(templates: tuple[PlanTemplate, ...]) -> tuple[Plan, ...]:
    if type(templates) is not tuple or not 1 <= len(templates) <= _MAX_TEMPLATES:
        _fail(MatrixErrorCode.TEMPLATE)

    template_ids: set[str] = set()
    plan_count = 0
    for template in templates:
        if type(template) is not PlanTemplate or template.template_id in template_ids:
            _fail(MatrixErrorCode.TEMPLATE)
        template_ids.add(template.template_id)
        count = math.prod(len(template.axes[name]) for name in template._axis_order)
        if count > _MAX_PLANS - plan_count:
            _fail(MatrixErrorCode.LIMIT)
        plan_count += count

    prepared: list[tuple[str, str, FrozenObject]] = []
    all_targets: set[str] = set()
    aggregate_size = 0
    for template in templates:
        local_targets: set[str] = set()
        value_rows = (template.axes[name] for name in template._axis_order)
        for combination in itertools.product(*value_rows):
            values = dict(zip(template._axis_order, combination, strict=True))
            target_id = _render_target(template, values)
            if target_id in all_targets:
                _fail(MatrixErrorCode.COLLISION)
            local_targets.add(target_id)
            all_targets.add(target_id)

            options = _merge_options(
                template,
                values,
                template.target_overrides.get(target_id),
            )
            aggregate_size += (
                len(template.template_id.encode("ascii"))
                + len(target_id.encode("ascii"))
                + _logical_size(options)
            )
            if aggregate_size > _MAX_AGGREGATE_OUTPUT_BYTES:
                _fail(MatrixErrorCode.LIMIT)
            prepared.append((template.template_id, target_id, options))

        if any(target not in local_targets for target in template.target_overrides):
            _fail(MatrixErrorCode.OVERRIDE)

    return tuple(Plan(*row) for row in prepared)
```

## Example

```python
base_options = {
    "retries": 2,
    "metadata": {"owner": "operations"},
    "region": "fallback",
}
target_overrides = {
    "deploy-eu-small-eu": {
        "retries": None,
        "metadata": {"owner": "edge"},
    },
}

template = PlanTemplate(
    template_id="deploy",
    target_pattern="deploy-{region}-{size}-{region}",
    axes=(
        Axis("size", ("small", "large")),
        Axis("region", ("eu", "us")),
    ),
    base_options=base_options,
    target_overrides=target_overrides,
)

base_options["metadata"]["owner"] = "changed"
target_overrides["deploy-eu-small-eu"]["metadata"]["owner"] = "changed"
plans = expand_plan_matrix((template,))

first = dict(plans[0].options.items)
second = dict(plans[1].options.items)

collision_template = PlanTemplate(
    template_id="collision",
    target_pattern="x-{left}{right}",
    axes=(
        Axis("left", ("a", "ab")),
        Axis("right", ("bc", "c")),
    ),
    base_options={},
    target_overrides={},
)
try:
    expand_plan_matrix((collision_template,))
except PlanMatrixError as error:
    collision_code = error.code
else:
    collision_code = None

assert (
    tuple(plan.target_id for plan in plans),
    tuple(name for name, _value in plans[0].options.items),
    first["retries"],
    first["region"],
    first["metadata"],
    second["retries"],
    second["metadata"],
    collision_code,
) == (
    (
        "deploy-eu-small-eu",
        "deploy-eu-large-eu",
        "deploy-us-small-us",
        "deploy-us-large-us",
    ),
    ("retries", "metadata", "region", "size"),
    None,
    "eu",
    FrozenObject((("owner", "edge"),)),
    2,
    FrozenObject((("owner", "operations"),)),
    MatrixErrorCode.COLLISION,
)
```

## Trade-offs and Limitations

The two-pass expansion allocates bounded preflight rows before allocating the
returned tuple. Its aggregate budget measures the UTF-8 or ASCII content of
every repeated plan field plus deterministic scalar and container framing; it
is a logical in-memory limit, not a promise about any later wire encoding.
Large or sparse matrices are better represented by a lazy iterator or a domain-
specific planner, but those designs need a different way to prove global target
uniqueness and validate every override.

Patterns support only lowercase ASCII literals and flat `{name}` placeholders:
there is no escaping, formatting, nesting, evaluation, environment lookup, or
shell expansion. Option values are bounded JSON-like trees, and all containers
are copied into frozen records or tuples without mutating the inputs. Empty axes,
unknown declarations or overrides, duplicate or non-string axis values, cycles,
aliases, non-finite numbers, and rendered target collisions are rejected with
fixed codes that never contain supplied names or values.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Bounded Flat Placeholder Template](parse-a-bounded-flat-placeholder-template.md)
- [Expand Bounded Nested Brace Alternatives](expand-bounded-nested-brace-alternatives.md)
- [Merge Nested Mappings Without Mutating Inputs](merge-nested-mappings-without-mutating-inputs.md)
<!-- catalog:related:end -->
