---
title: "Parse Bounded UTF-8 JSON Without Duplicate Object Names or Non-Finite Numbers"
snippet_type: recipe
use_cases:
  - parsing
  - serialization
  - validation
  - security
tested_python:
  - "3.14"
dependencies: []
related:
  - normalize-a-bounded-json-copy-before-standard-schema-validation.md
  - migrate-one-bounded-json-record-to-a-current-version.md
  - ../networking-protocols/parse-a-bounded-three-state-json-response-envelope.md
---

# Parse Bounded UTF-8 JSON Without Duplicate Object Names or Non-Finite Numbers

## Idea and Problem

Own the bytes-to-value boundary so duplicate object names, unsafe numbers, malformed text, and oversized trees are rejected before application validation begins.

Ordinary `json.loads()` keeps only the last occurrence of a repeated object
name and accepts JavaScript-style non-finite constants by default. Once either
choice reaches a dictionary or application model, the original ambiguity is
gone. A strict boundary can retain object pairs long enough to detect duplicate
decoded names, sanitize parser failures, and return only an immutable passive
tree.

## When to Use

Use this recipe for small untrusted JSON documents whose exact parse semantics
matter: configuration payloads, signed metadata, policy inputs, or protocol
documents that will receive separate schema validation. The fixed input cap is
the pre-parse resource boundary; depth, node, container, and text limits are
additional acceptance checks after the standard-library parser materializes
the value.

Do not use it when duplicate names have application-defined merge semantics,
numbers require lossless decimal or lexical round trips, or inputs can exceed
the in-memory byte limit. Those cases need a purpose-built streaming parser and
an explicit numeric representation.

## Implementation

```python
import json
import math
from collections.abc import Mapping
from dataclasses import dataclass
from enum import StrEnum
from types import MappingProxyType

_MAX_INPUT_BYTES = 256 * 1024
_MAX_CONTAINER_DEPTH = 32
_MAX_NODES = 8_192
_MAX_CONTAINER_ITEMS = 1_024
_MAX_TEXT_ITEM_BYTES = 32 * 1024
_MAX_TEXT_BYTES = 192 * 1024
_MAX_INTEGER_DIGITS = 128
_MAX_FLOAT_TOKEN_CHARACTERS = 128
_UTF8_BOM = b"\xef\xbb\xbf"


type FrozenJson = (
    None | bool | int | float | str | tuple[FrozenJson, ...] | Mapping[str, FrozenJson]
)


class JsonBoundaryProblem(StrEnum):
    EMPTY_INPUT = "empty_input"
    INPUT_TOO_LARGE = "input_too_large"
    UTF8_BOM = "utf8_bom"
    INVALID_UTF8 = "invalid_utf8"
    INVALID_SYNTAX = "invalid_syntax"
    DUPLICATE_NAME = "duplicate_name"
    NON_FINITE_NUMBER = "non_finite_number"
    NUMBER_TOKEN_TOO_LONG = "number_token_too_long"
    SURROGATE_TEXT = "surrogate_text"
    CONTAINER_TOO_DEEP = "container_too_deep"
    TOO_MANY_NODES = "too_many_nodes"
    TOO_MANY_CONTAINER_ITEMS = "too_many_container_items"
    TEXT_ITEM_TOO_LARGE = "text_item_too_large"
    TEXT_BUDGET_EXCEEDED = "text_budget_exceeded"
    PARSER_RECURSION = "parser_recursion"


class BoundedJsonError(ValueError):
    def __init__(
        self,
        problem: JsonBoundaryProblem,
        *,
        byte_offset: int | None = None,
    ) -> None:
        self.problem = problem
        self.byte_offset = byte_offset
        message = problem.value
        if byte_offset is not None:
            message = f"{message} at byte {byte_offset}"
        super().__init__(message)


class _HookProblem(Exception):
    def __init__(self, problem: JsonBoundaryProblem) -> None:
        self.problem = problem


@dataclass(slots=True)
class _TreeBudget:
    nodes: int = 0
    text_bytes: int = 0


def _bounded_integer(token: str) -> int:
    digits = token[1:] if token.startswith("-") else token
    if len(digits) > _MAX_INTEGER_DIGITS:
        raise _HookProblem(JsonBoundaryProblem.NUMBER_TOKEN_TOO_LONG)
    return int(token)


def _bounded_float(token: str) -> float:
    if len(token) > _MAX_FLOAT_TOKEN_CHARACTERS:
        raise _HookProblem(JsonBoundaryProblem.NUMBER_TOKEN_TOO_LONG)
    value = float(token)
    if not math.isfinite(value):
        raise _HookProblem(JsonBoundaryProblem.NON_FINITE_NUMBER)
    return value


def _reject_constant(token: str) -> object:
    del token
    raise _HookProblem(JsonBoundaryProblem.NON_FINITE_NUMBER)


def _unique_object(pairs: list[tuple[str, object]]) -> dict[str, object]:
    if len(pairs) > _MAX_CONTAINER_ITEMS:
        raise _HookProblem(JsonBoundaryProblem.TOO_MANY_CONTAINER_ITEMS)
    result: dict[str, object] = {}
    for name, value in pairs:
        if name in result:
            raise _HookProblem(JsonBoundaryProblem.DUPLICATE_NAME)
        result[name] = value
    return result


def _count_text(value: str, budget: _TreeBudget) -> None:
    try:
        size = len(value.encode("utf-8", errors="strict"))
    except UnicodeEncodeError:
        raise _HookProblem(JsonBoundaryProblem.SURROGATE_TEXT) from None
    if size > _MAX_TEXT_ITEM_BYTES:
        raise _HookProblem(JsonBoundaryProblem.TEXT_ITEM_TOO_LARGE)
    budget.text_bytes += size
    if budget.text_bytes > _MAX_TEXT_BYTES:
        raise _HookProblem(JsonBoundaryProblem.TEXT_BUDGET_EXCEEDED)


def _freeze(
    value: object,
    budget: _TreeBudget,
    *,
    container_depth: int,
) -> FrozenJson:
    budget.nodes += 1
    if budget.nodes > _MAX_NODES:
        raise _HookProblem(JsonBoundaryProblem.TOO_MANY_NODES)

    if value is None or type(value) in (bool, int):
        return value
    if type(value) is float:
        if not math.isfinite(value):
            raise _HookProblem(JsonBoundaryProblem.NON_FINITE_NUMBER)
        return value
    if type(value) is str:
        _count_text(value, budget)
        return value

    if type(value) is list:
        depth = container_depth + 1
        if depth > _MAX_CONTAINER_DEPTH:
            raise _HookProblem(JsonBoundaryProblem.CONTAINER_TOO_DEEP)
        if len(value) > _MAX_CONTAINER_ITEMS:
            raise _HookProblem(JsonBoundaryProblem.TOO_MANY_CONTAINER_ITEMS)
        return tuple(_freeze(item, budget, container_depth=depth) for item in value)

    if type(value) is dict:
        depth = container_depth + 1
        if depth > _MAX_CONTAINER_DEPTH:
            raise _HookProblem(JsonBoundaryProblem.CONTAINER_TOO_DEEP)
        if len(value) > _MAX_CONTAINER_ITEMS:
            raise _HookProblem(JsonBoundaryProblem.TOO_MANY_CONTAINER_ITEMS)
        frozen: dict[str, FrozenJson] = {}
        for name, item in value.items():
            _count_text(name, budget)
            frozen[name] = _freeze(item, budget, container_depth=depth)
        return MappingProxyType(frozen)

    raise AssertionError("the JSON decoder returned an unsupported value")


def parse_bounded_json(data: bytes) -> FrozenJson:
    if type(data) is not bytes:
        raise TypeError("data must be exact bytes")
    if not data:
        raise BoundedJsonError(JsonBoundaryProblem.EMPTY_INPUT)
    if len(data) > _MAX_INPUT_BYTES:
        raise BoundedJsonError(JsonBoundaryProblem.INPUT_TOO_LARGE)
    if data.startswith(_UTF8_BOM):
        raise BoundedJsonError(JsonBoundaryProblem.UTF8_BOM, byte_offset=0)

    try:
        document = data.decode("utf-8", errors="strict")
    except UnicodeDecodeError as error:
        offset = error.start
        del error
        raise BoundedJsonError(
            JsonBoundaryProblem.INVALID_UTF8,
            byte_offset=offset,
        ) from None

    try:
        parsed = json.loads(
            document,
            object_pairs_hook=_unique_object,
            parse_int=_bounded_integer,
            parse_float=_bounded_float,
            parse_constant=_reject_constant,
        )
        return _freeze(parsed, _TreeBudget(), container_depth=0)
    except _HookProblem as error:
        problem = error.problem
        del error
        raise BoundedJsonError(problem) from None
    except json.JSONDecodeError as error:
        offset = len(document[: error.pos].encode("utf-8"))
        del error
        raise BoundedJsonError(
            JsonBoundaryProblem.INVALID_SYNTAX,
            byte_offset=offset,
        ) from None
    except RecursionError:
        raise BoundedJsonError(JsonBoundaryProblem.PARSER_RECURSION) from None
```

## Example

```python
document = parse_bounded_json(b'{"enabled": false, "limits": [0, 1.5], "note": null}')

try:
    parse_bounded_json(b'{"name": 1, "\\u006eame": 2}')
except BoundedJsonError as error:
    duplicate_problem = error.problem
else:
    duplicate_problem = None

try:
    parse_bounded_json(b"1e400")
except BoundedJsonError as error:
    overflow_problem = error.problem
else:
    overflow_problem = None

assert (
    document["enabled"],
    document["limits"],
    document["note"],
    duplicate_problem,
    overflow_problem,
) == (
    False,
    (0, 1.5),
    None,
    JsonBoundaryProblem.DUPLICATE_NAME,
    JsonBoundaryProblem.NON_FINITE_NUMBER,
)
```

## Trade-offs and Limitations

Duplicate comparison uses exact decoded Python strings. Escape-equivalent names
therefore collide, while canonically equivalent Unicode spellings remain
different because the parser deliberately performs no normalization or case
folding. Object insertion order is preserved. Any JSON root value is accepted,
including a scalar.

The byte cap is the only bound enforced before `json.loads()` allocates the
tree. Post-parse depth, node, container, and text budgets reject unsuitable
values but cannot undo that temporary allocation. Every value, including the
root, counts as one node; object names do not count as nodes, but every name
occurrence and string value contributes its strict UTF-8 byte length to the
text budget. Container depth counts the root array or object as one.

Raised errors contain only a closed problem code and, when available, a byte
offset. Debuggers and traceback collectors that capture frame locals can still
retain the decoded document, so disable local-variable capture at this boundary
when input disclosure is a concern.

Float tokens use Python binary floating point. Finite rounding and underflow
retain standard `float` behavior; use a decimal representation when exact
decimal round trips matter. The immutable output prevents structural mutation
through the returned tree, but it is not a schema model and does not prove any
application-level meaning.

## Related Snippets

<!-- catalog:related:start -->
- [Normalize a Bounded JSON Copy Before Standard Schema Validation](normalize-a-bounded-json-copy-before-standard-schema-validation.md)
- [Migrate One Bounded JSON Record to a Current Version](migrate-one-bounded-json-record-to-a-current-version.md)
- [Parse a Bounded Three-State JSON Response Envelope](../networking-protocols/parse-a-bounded-three-state-json-response-envelope.md)
<!-- catalog:related:end -->
