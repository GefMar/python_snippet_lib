---
title: "Normalize a Bounded JSON Copy Before Standard Schema Validation"
snippet_type: integration
use_cases:
  - configuration
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies:
  - name: jsonschema
    version: "4.25.1"
related:
  - normalize-bounded-named-options-with-explicit-default-semantics.md
  - migrate-one-bounded-json-record-to-a-current-version.md
  - get-nested-values-with-a-validated-dot-path.md
---

# Normalize a Bounded JSON Copy Before Standard Schema Validation

## Idea and Problem

Apply trusted adapters to explicit paths in a defensive JSON copy, then validate the normalized document with an unchanged standard schema.

Normalization and schema validation are different boundaries. A closed adapter
set performs deliberate conversions without embedding Python callables in JSON
Schema or depending on validator keyword order. The result reports bounded,
stable issue codes and paths without retaining rejected values or library error
messages.

## When to Use

Use this integration when a small JSON-compatible document accepts a few
well-defined textual representations but downstream code needs one canonical
shape. It fits trusted application adapters such as trimming a label or parsing
one strict decimal field before applying a Draft 2020-12 schema.

Keep the adapter paths versioned with the schema. Use a dedicated decoding
model when aliases, defaults, unions, cross-field transformations, or many
coercions dominate the boundary. Reject remote schemas and references when
validation must remain deterministic and offline.

## Implementation

```python
import math
import re
from collections.abc import Callable, Mapping
from dataclasses import dataclass
from itertools import combinations
from types import MappingProxyType
from typing import TypeAlias

from jsonschema import Draft202012Validator
from jsonschema.exceptions import SchemaError

JSONValue: TypeAlias = None | bool | int | float | str | list["JSONValue"] | dict[str, "JSONValue"]
FrozenJSONValue: TypeAlias = (
    None
    | bool
    | int
    | float
    | str
    | tuple["FrozenJSONValue", ...]
    | Mapping[str, "FrozenJSONValue"]
)
PathSegment: TypeAlias = str | int

_DRAFT_URI = "https://json-schema.org/draft/2020-12/schema"
_PATH_NAME = re.compile(r"[a-z][a-z0-9_-]{0,63}", re.ASCII).fullmatch
_ISSUE_CODE = re.compile(r"[a-z][a-z0-9_]{0,31}", re.ASCII).fullmatch
_KEYWORD = re.compile(r"[A-Za-z][A-Za-z0-9_-]{0,63}", re.ASCII).fullmatch
_POINTER_INDEX = re.compile(r"0|[1-9][0-9]{0,5}", re.ASCII).fullmatch
_MAX_NODES = 512
_MAX_DEPTH = 10
_MAX_CONTAINER_ITEMS = 64
_MAX_TEXT_BYTES = 4 * 1_024
_MAX_TOTAL_TEXT_BYTES = 64 * 1_024
_MAX_ABSOLUTE_INTEGER = 1_000_000_000_000_000
_MAX_ADAPTERS = 16
_MAX_PATH_DEPTH = 8
_MAX_ISSUES = 32


class NormalizationConfigurationError(ValueError):
    pass


class AdapterRejection(ValueError):
    def __init__(self, code: str) -> None:
        if type(code) is not str or _ISSUE_CODE(code) is None:
            raise ValueError("adapter issue code must be a conservative ASCII identifier")
        super().__init__(code)
        self.code = code


@dataclass(frozen=True, slots=True)
class PathAdapter:
    path: tuple[PathSegment, ...]
    transform: Callable[[JSONValue], JSONValue]


@dataclass(frozen=True, slots=True)
class ValidationIssue:
    path: tuple[PathSegment, ...]
    code: str


@dataclass(frozen=True, slots=True)
class NormalizationResult:
    document: FrozenJSONValue | None
    issues: tuple[ValidationIssue, ...]

    @property
    def accepted(self) -> bool:
        return not self.issues


def _copy_bounded_json(value: object, *, name: str) -> JSONValue:
    nodes = 0
    text_bytes = 0

    def copy_text(text: str) -> str:
        nonlocal text_bytes
        try:
            size = len(text.encode("utf-8"))
        except UnicodeEncodeError as error:
            raise ValueError(f"{name} contains invalid Unicode text") from error
        if size > _MAX_TEXT_BYTES:
            raise ValueError(f"{name} contains text above the per-value byte limit")
        text_bytes += size
        if text_bytes > _MAX_TOTAL_TEXT_BYTES:
            raise ValueError(f"{name} exceeds the aggregate text byte limit")
        return text

    def copy_node(node: object, *, depth: int) -> JSONValue:
        nonlocal nodes
        nodes += 1
        if nodes > _MAX_NODES:
            raise ValueError(f"{name} exceeds the node limit")
        if depth > _MAX_DEPTH:
            raise ValueError(f"{name} exceeds the depth limit")

        if node is None or type(node) is bool:
            return node
        if type(node) is int:
            if abs(node) > _MAX_ABSOLUTE_INTEGER:
                raise ValueError(f"{name} contains an integer outside the supported range")
            return node
        if type(node) is float:
            if not math.isfinite(node):
                raise ValueError(f"{name} contains a non-finite number")
            return node
        if type(node) is str:
            return copy_text(node)
        if type(node) is list:
            if len(node) > _MAX_CONTAINER_ITEMS:
                raise ValueError(f"{name} contains an oversized array")
            return [copy_node(item, depth=depth + 1) for item in node]
        if type(node) is dict:
            if len(node) > _MAX_CONTAINER_ITEMS:
                raise ValueError(f"{name} contains an oversized object")
            copied: dict[str, JSONValue] = {}
            for key, item in node.items():
                if type(key) is not str:
                    raise TypeError(f"{name} object keys must be exact strings")
                copied[copy_text(key)] = copy_node(item, depth=depth + 1)
            return copied
        raise TypeError(f"{name} must contain only exact JSON-compatible values")

    return copy_node(value, depth=0)


def _path_key(path: tuple[PathSegment, ...]) -> tuple[tuple[int, str | int], ...]:
    return tuple((0, segment) if type(segment) is str else (1, segment) for segment in path)


def _validated_adapters(value: object) -> tuple[PathAdapter, ...]:
    if type(value) is not tuple:
        raise TypeError("adapters must be an exact tuple")
    if len(value) > _MAX_ADAPTERS:
        raise ValueError("adapter count exceeds the supported limit")

    adapters: list[PathAdapter] = []
    paths: set[tuple[PathSegment, ...]] = set()
    for adapter in value:
        if type(adapter) is not PathAdapter:
            raise TypeError("adapters must contain exact PathAdapter values")
        if type(adapter.path) is not tuple or not 1 <= len(adapter.path) <= _MAX_PATH_DEPTH:
            raise ValueError("adapter paths must be non-empty bounded tuples")
        validated_path: list[PathSegment] = []
        for segment in adapter.path:
            if type(segment) is str:
                if _PATH_NAME(segment) is None:
                    raise ValueError("text path segments must be conservative ASCII names")
            elif type(segment) is int:
                if not 0 <= segment < _MAX_CONTAINER_ITEMS:
                    raise ValueError("integer path segments are outside the supported range")
            else:
                raise TypeError("path segments must be exact strings or integers")
            validated_path.append(segment)
        path = tuple(validated_path)
        if path in paths:
            raise ValueError("adapter paths must be unique")
        if not callable(adapter.transform):
            raise TypeError("adapter transforms must be callable")
        paths.add(path)
        adapters.append(PathAdapter(path, adapter.transform))

    for left, right in combinations(paths, 2):
        shortest = min(len(left), len(right))
        if left[:shortest] == right[:shortest]:
            raise ValueError("adapter paths must not overlap")
    return tuple(sorted(adapters, key=lambda adapter: _path_key(adapter.path)))


def _decode_pointer_token(token: str) -> str:
    if "%" in token or re.search(r"~(?![01])", token):
        raise NormalizationConfigurationError("local JSON pointers must use plain ~0/~1 escaping")
    return token.replace("~1", "/").replace("~0", "~")


def _require_local_pointer(root: JSONValue, reference: object) -> None:
    if type(reference) is not str or not reference.startswith("#"):
        raise NormalizationConfigurationError("remote schema references are forbidden")
    if reference == "#":
        return
    if not reference.startswith("#/"):
        raise NormalizationConfigurationError("only local JSON Pointer references are supported")

    current: JSONValue = root
    for raw_token in reference[2:].split("/"):
        token = _decode_pointer_token(raw_token)
        if type(current) is dict and token in current:
            current = current[token]
            continue
        if type(current) is list and _POINTER_INDEX(token) is not None:
            index = int(token)
            if index < len(current):
                current = current[index]
                continue
        raise NormalizationConfigurationError("a local schema reference does not resolve")


def _validate_schema_policy(schema: dict[str, JSONValue]) -> None:
    declared_draft = schema.get("$schema")
    if declared_draft is not None and declared_draft != _DRAFT_URI:
        raise NormalizationConfigurationError("schema must use Draft 2020-12")

    stack: list[JSONValue] = [schema]
    references: list[object] = []
    while stack:
        node = stack.pop()
        if type(node) is dict:
            if any(key in node for key in ("$id", "$anchor", "$dynamicAnchor", "$dynamicRef")):
                raise NormalizationConfigurationError(
                    "schema identifiers and dynamic references are outside this contract"
                )
            if "$ref" in node:
                references.append(node["$ref"])
            stack.extend(node.values())
        elif type(node) is list:
            stack.extend(node)
    for reference in references:
        _require_local_pointer(schema, reference)


def _target(
    document: JSONValue,
    path: tuple[PathSegment, ...],
) -> tuple[dict[str, JSONValue] | list[JSONValue], PathSegment] | None:
    current = document
    for segment in path[:-1]:
        if type(current) is dict and type(segment) is str and segment in current:
            current = current[segment]
        elif type(current) is list and type(segment) is int and segment < len(current):
            current = current[segment]
        else:
            return None

    leaf = path[-1]
    if type(current) is dict and type(leaf) is str and leaf in current:
        return current, leaf
    if type(current) is list and type(leaf) is int and leaf < len(current):
        return current, leaf
    return None


def _issue_sort_key(issue: ValidationIssue) -> tuple[object, ...]:
    return (*_path_key(issue.path), (2, issue.code))


def _freeze_json(value: JSONValue) -> FrozenJSONValue:
    if type(value) is list:
        return tuple(_freeze_json(item) for item in value)
    if type(value) is dict:
        return MappingProxyType({key: _freeze_json(item) for key, item in value.items()})
    return value


def normalize_json_copy(
    schema: dict[str, JSONValue],
    document: JSONValue,
    adapters: tuple[PathAdapter, ...],
) -> NormalizationResult:
    copied_schema = _copy_bounded_json(schema, name="schema")
    if type(copied_schema) is not dict:
        raise TypeError("schema must be an exact JSON object")
    _validate_schema_policy(copied_schema)
    try:
        Draft202012Validator.check_schema(copied_schema)
    except SchemaError:
        raise NormalizationConfigurationError("schema is not valid Draft 2020-12") from None

    normalized = _copy_bounded_json(document, name="document")
    validated_adapters = _validated_adapters(adapters)
    issues: list[ValidationIssue] = []
    for adapter in validated_adapters:
        located = _target(normalized, adapter.path)
        if located is None:
            issues.append(ValidationIssue(adapter.path, "adapter.path_missing"))
            continue
        parent, leaf = located
        original = parent[leaf]
        try:
            converted = adapter.transform(_copy_bounded_json(original, name="adapter input"))
        except AdapterRejection as error:
            issues.append(ValidationIssue(adapter.path, f"adapter.{error.code}"))
            continue
        parent[leaf] = _copy_bounded_json(converted, name="adapter output")

    normalized = _copy_bounded_json(normalized, name="normalized document")
    validator = Draft202012Validator(copied_schema)
    issue_limit_reached = False
    for error in validator.iter_errors(normalized):
        if len(issues) >= _MAX_ISSUES:
            issue_limit_reached = True
            break
        path = tuple(error.absolute_path)
        if any(type(segment) not in (str, int) for segment in path):
            path = ()
        keyword = error.validator
        if type(keyword) is not str or _KEYWORD(keyword) is None:
            keyword = "invalid"
        issues.append(ValidationIssue(path, f"schema.{keyword.lower()}"))

    if issue_limit_reached:
        issues = issues[: _MAX_ISSUES - 1]
        issues.append(ValidationIssue((), "schema.issue_limit"))
    stable_issues = tuple(sorted(set(issues), key=_issue_sort_key))
    accepted_document = None if stable_issues else _freeze_json(normalized)
    return NormalizationResult(accepted_document, stable_issues)
```

## Example

```python
def strip_label(value: JSONValue) -> JSONValue:
    if type(value) is not str:
        raise AdapterRejection("text_required")
    stripped = value.strip()
    if not stripped:
        raise AdapterRejection("blank")
    return stripped


def parse_port(value: JSONValue) -> JSONValue:
    if type(value) is not str or re.fullmatch(r"[1-9][0-9]{0,4}", value) is None:
        raise AdapterRejection("decimal_required")
    return int(value)


schema = {
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "type": "object",
    "additionalProperties": False,
    "required": ["label", "port"],
    "properties": {
        "label": {"type": "string", "minLength": 1},
        "port": {"type": "integer", "minimum": 1, "maximum": 65_535},
    },
}
adapters = (
    PathAdapter(("label",), strip_label),
    PathAdapter(("port",), parse_port),
)
raw = {"label": " primary ", "port": "8080"}

accepted = normalize_json_copy(schema, raw, adapters)
rejected = normalize_json_copy(
    schema,
    {"label": " ", "port": "99999"},
    adapters,
)

try:
    normalize_json_copy({"$ref": "https://invalid.example/schema"}, {}, ())
except NormalizationConfigurationError:
    remote_reference_rejected = True
else:
    remote_reference_rejected = False

assert (
    dict(accepted.document) if isinstance(accepted.document, Mapping) else None,
    accepted.accepted,
    raw,
    rejected.document,
    rejected.issues,
    remote_reference_rejected,
) == (
    {"label": "primary", "port": 8080},
    True,
    {"label": " primary ", "port": "8080"},
    None,
    (
        ValidationIssue(("label",), "adapter.blank"),
        ValidationIssue(("port",), "schema.maximum"),
    ),
    True,
)
```

## Trade-offs and Limitations

The function copies and materializes the schema, input, adapter values, and
normalized result under fixed node, depth, container, and text budgets. An
accepted result recursively freezes objects as read-only mappings and arrays
as tuples; any issue suppresses the normalized document entirely. Adapter paths
are unique, non-overlapping, and applied in deterministic path order. Adapters
are trusted application code: expected value rejection becomes a stable issue,
while an unexpected adapter exception or invalid return value remains a
programming error.

Only plain local JSON Pointer references are accepted; remote references,
dynamic references, identifiers, anchors, and remote retrieval are excluded.
The standard validator does not enforce `format` annotations by default. Issue
codes intentionally retain only a bounded instance path and validator keyword,
so they omit rejected values, schema values, and version-specific library
messages that may be useful during private debugging.

## Related Snippets

<!-- catalog:related:start -->
- [Normalize Bounded Named Options with Explicit Default Semantics](normalize-bounded-named-options-with-explicit-default-semantics.md)
- [Migrate One Bounded JSON Record to a Current Version](migrate-one-bounded-json-record-to-a-current-version.md)
- [Get Nested Values with a Validated Dot Path](get-nested-values-with-a-validated-dot-path.md)
<!-- catalog:related:end -->
