---
title: "Parse a Bounded Three-State JSON Response Envelope"
snippet_type: integration
use_cases:
  - networking
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - parse-a-bounded-ascii-media-type-value.md
  - ../configuration-serialization/fingerprint-a-set-like-json-array-deterministically.md
  - ../security-privacy/redact-explicit-paths-in-bounded-json-like-data.md
---

# Parse a Bounded Three-State JSON Response Envelope

## Idea and Problem

Turn one strictly shaped success, fail, or error JSON mapping into a typed immutable result after validating and freezing its complete value tree.

The accepted key matrices are exact: `success` and `fail` contain only
`status` and `data`; `error` contains `status` and `message`, with optional
`code` and `data`. The parser accepts only passive built-in JSON value types,
rejects aliases and cycles, and applies fixed depth, node, text, and integer
budgets before exposing any nested data.

## When to Use

Use this integration at the boundary after a trusted JSON decoder has produced
one small response mapping and all peers have agreed on these three envelope
shapes. Catch `EnvelopeProtocolError` as a malformed peer response, then apply
application-specific handling to the returned outcome type.

Keep duplicate-key rejection and source-byte limits in the JSON decoder; a
Python dictionary can no longer reveal duplicate object members. Keep HTTP
status interpretation, authentication, retry decisions, connection cleanup,
and transport failures outside this value parser.

## Implementation

```python
import math
from collections.abc import Mapping
from dataclasses import dataclass
from types import MappingProxyType
from typing import TypeAlias


_MAX_NODES = 4_096
_MAX_DEPTH = 32
_MAX_TEXT_BYTES = 256_000
_MAX_INTEGER_BITS = 256
_MAX_MESSAGE_BYTES = 4_096
_MIN_CODE = -(2**63)
_MAX_CODE = 2**63 - 1

JsonScalar: TypeAlias = bool | int | float | str | None
FrozenJson: TypeAlias = (
    JsonScalar
    | tuple["FrozenJson", ...]
    | Mapping[str, "FrozenJson"]
)


class EnvelopeProtocolError(ValueError):
    pass


@dataclass(frozen=True, slots=True)
class SuccessResponse:
    data: FrozenJson


@dataclass(frozen=True, slots=True)
class FailResponse:
    data: Mapping[str, FrozenJson]


@dataclass(frozen=True, slots=True)
class ErrorResponse:
    message: str
    code: int | None
    data_present: bool
    data: FrozenJson


ResponseEnvelope: TypeAlias = SuccessResponse | FailResponse | ErrorResponse


@dataclass(slots=True)
class _Budget:
    nodes: int = 0
    text_bytes: int = 0


def _invalid() -> None:
    raise EnvelopeProtocolError("invalid response envelope")


def _add_text(text: str, budget: _Budget) -> None:
    remaining = _MAX_TEXT_BYTES - budget.text_bytes
    if len(text) > remaining:
        _invalid()
    try:
        size = len(text.encode("utf-8"))
    except UnicodeEncodeError:
        _invalid()
    budget.text_bytes += size
    if budget.text_bytes > _MAX_TEXT_BYTES:
        _invalid()


def _freeze_json(
    value: object,
    *,
    container_depth: int,
    budget: _Budget,
    seen_containers: set[int],
) -> FrozenJson:
    budget.nodes += 1
    if budget.nodes > _MAX_NODES:
        _invalid()

    if value is None or type(value) is bool:
        return value
    if type(value) is int:
        if value.bit_length() > _MAX_INTEGER_BITS:
            _invalid()
        return value
    if type(value) is float:
        if not math.isfinite(value):
            _invalid()
        return value
    if type(value) is str:
        _add_text(value, budget)
        return value
    if type(value) not in (list, dict):
        _invalid()

    next_depth = container_depth + 1
    if next_depth > _MAX_DEPTH:
        _invalid()
    identity = id(value)
    if identity in seen_containers:
        _invalid()
    seen_containers.add(identity)

    if type(value) is list:
        return tuple(
            _freeze_json(
                item,
                container_depth=next_depth,
                budget=budget,
                seen_containers=seen_containers,
            )
            for item in value
        )

    frozen: dict[str, FrozenJson] = {}
    for key, item in value.items():
        if type(key) is not str:
            _invalid()
        _add_text(key, budget)
        frozen[key] = _freeze_json(
            item,
            container_depth=next_depth,
            budget=budget,
            seen_containers=seen_containers,
        )
    return MappingProxyType(frozen)


def parse_response_envelope(envelope: object) -> ResponseEnvelope:
    if type(envelope) is not dict:
        _invalid()

    frozen_root = _freeze_json(
        envelope,
        container_depth=0,
        budget=_Budget(),
        seen_containers=set(),
    )
    keys = frozenset(envelope)
    status = envelope.get("status")
    if type(status) is not str:
        _invalid()

    if status == "success":
        if keys != {"status", "data"}:
            _invalid()
        return SuccessResponse(frozen_root["data"])

    if status == "fail":
        if keys != {"status", "data"} or type(envelope["data"]) is not dict:
            _invalid()
        return FailResponse(frozen_root["data"])

    if status != "error":
        _invalid()
    required = {"status", "message"}
    allowed = required | {"code", "data"}
    if not required <= keys or not keys <= allowed:
        _invalid()

    message = envelope["message"]
    if type(message) is not str or not message:
        _invalid()
    if len(message.encode("utf-8")) > _MAX_MESSAGE_BYTES:
        _invalid()

    code: int | None = None
    if "code" in envelope:
        candidate = envelope["code"]
        if type(candidate) is not int or not _MIN_CODE <= candidate <= _MAX_CODE:
            _invalid()
        code = candidate

    data_present = "data" in envelope
    data = frozen_root["data"] if data_present else None
    return ErrorResponse(message, code, data_present, data)
```

## Example

```python
source = {
    "status": "success",
    "data": {"items": [{"id": 7}], "complete": True},
}
success = parse_response_envelope(source)
failure = parse_response_envelope(
    {"status": "fail", "data": {"query": "is required"}},
)
error = parse_response_envelope(
    {
        "status": "error",
        "message": "temporarily unavailable",
        "code": 503,
        "data": None,
    },
)
error_without_data = parse_response_envelope(
    {"status": "error", "message": "unavailable"},
)

source["data"]["items"][0]["id"] = 99
try:
    success.data["new"] = "value"
except TypeError:
    frozen = True
else:
    frozen = False

invalid_envelopes = (
    {"status": "success", "data": None, "extra": True},
    {"status": "fail", "data": ["not", "an", "object"]},
    {"status": "error", "message": "", "code": 1},
    {"status": "error", "message": "bad", "code": True},
    {"status": "pending", "data": None},
)
rejected = 0
for invalid in invalid_envelopes:
    try:
        parse_response_envelope(invalid)
    except EnvelopeProtocolError:
        rejected += 1

assert (
    isinstance(success, SuccessResponse),
    success.data["items"][0]["id"],
    isinstance(success.data["items"], tuple),
    dict(failure.data),
    error,
    error_without_data.data_present,
    frozen,
    rejected,
) == (
    True,
    7,
    True,
    {"query": "is required"},
    ErrorResponse("temporarily unavailable", 503, True, None),
    False,
    True,
    5,
)
```

## Trade-offs and Limitations

Validation and freezing are linear in the accepted tree and allocate new
tuples and dictionaries for every container. Mapping proxies and tuples prevent
mutation through the result, but they are not ordinary JSON containers and may
need conversion before serialization. Shared container identities are rejected
even when acyclic because a decoded JSON value is expected to be a tree.

The fixed budgets suit small envelopes, not bulk response bodies. The parser
does not consume JSON text, detect duplicate object members, validate an
application schema inside `data`, or attach meaning to error codes. Its key
matrices are intentionally closed, so protocol extensions require an explicit
versioned change rather than silently passing through extra members.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Bounded ASCII Media Type Value](parse-a-bounded-ascii-media-type-value.md)
- [Fingerprint a Set-Like JSON Array Deterministically](../configuration-serialization/fingerprint-a-set-like-json-array-deterministically.md)
- [Redact Explicit Paths in Bounded JSON-Like Data](../security-privacy/redact-explicit-paths-in-bounded-json-like-data.md)
<!-- catalog:related:end -->
