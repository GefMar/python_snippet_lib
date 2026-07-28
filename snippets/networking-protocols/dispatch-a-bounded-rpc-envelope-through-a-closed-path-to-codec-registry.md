---
title: "Dispatch a Bounded RPC Envelope Through a Closed Path-to-Codec Registry"
snippet_type: integration
use_cases:
  - networking
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - parse-a-bounded-three-state-json-response-envelope.md
  - ../configuration-serialization/parse-a-bounded-xml-envelope-with-closed-variant-dispatch.md
  - ../python-language/dispatch-named-strategies-with-an-explicit-function-mapping.md
---

# Dispatch a Bounded RPC Envelope Through a Closed Path-to-Codec Registry

## Idea and Problem

Dispatch one bounded RPC request by exact path, decode it with that path's trusted codec, and invoke one method from a separate closed dispatcher.

The path is never normalized and the request body never selects its own codec.
The decoded envelope must contain exactly `method` and `arguments`; the method
is a canonical ASCII name and the arguments are deeply frozen only after node,
depth, text, integer, and argument budgets pass. Every request failure becomes
one fixed, value-free code with an empty body.

## When to Use

Use this integration at an in-process protocol boundary after transport code has
already selected a request path and collected an immutable byte body. Construct
the dispatcher once from small tuples of trusted decoder/encoder pairs, trusted
handlers, and narrowly declared application exception classes.

Keep network I/O, authentication, authorization, TLS, timeouts, duplicate-key
rules inside each concrete decoder, and transport status mapping outside this
function. A schema or generated RPC framework is a better fit for an evolving
public protocol. This dispatcher does not implement XML-RPC, inspect payloads to
autodetect formats, or make untrusted Python callbacks safe.

## Implementation

```python
import math
import re
from collections.abc import Callable, Mapping
from dataclasses import dataclass
from enum import StrEnum
from types import MappingProxyType
from typing import Never

_MAX_ROUTES = 16
_MAX_METHODS = 64
_MAX_APPLICATION_ERRORS = 8
_MAX_PATH_BYTES = 128
_MAX_METHOD_BYTES = 128
_MAX_REQUEST_BYTES = 64 * 1_024
_MAX_RESPONSE_BYTES = 64 * 1_024
_MAX_DECODED_NODES = 2_048
_MAX_DECODED_DEPTH = 16
_MAX_ARGUMENTS = 32
_MAX_TEXT_BYTES = 64 * 1_024
_MAX_INTEGER_BITS = 256

_PATH = re.compile(
    r"/(?:[a-z][a-z0-9_-]{0,31})(?:/[a-z][a-z0-9_-]{0,31}){0,3}",
    re.ASCII,
)
_METHOD = re.compile(
    r"[a-z][a-z0-9_]{0,31}(?:\.[a-z][a-z0-9_]{0,31}){0,3}",
    re.ASCII,
)

type DecodedScalar = None | bool | int | float | str
type FrozenNode = DecodedScalar | tuple[FrozenNode, ...] | Mapping[str, FrozenNode]
type Decoder = Callable[[bytes], object]
type Encoder = Callable[[object], bytes]
type RpcArguments = tuple[FrozenNode, ...]
type RpcHandler = Callable[[RpcArguments], object]


class RpcCode(StrEnum):
    OK = "ok"
    ROUTE = "route"
    PARSE = "parse"
    METHOD = "method"
    APPLICATION = "application"
    HANDLER = "handler"
    ENCODING = "encoding"


@dataclass(frozen=True, slots=True)
class CodecRoute:
    path: str
    decoder: Decoder
    encoder: Encoder


@dataclass(frozen=True, slots=True)
class RpcMethod:
    name: str
    handler: RpcHandler
    application_errors: tuple[type[Exception], ...] = ()


@dataclass(frozen=True, slots=True, init=False)
class RpcDispatcher:
    routes: Mapping[str, CodecRoute]
    methods: Mapping[str, RpcMethod]

    def __init__(
        self,
        routes: tuple[CodecRoute, ...],
        methods: tuple[RpcMethod, ...],
    ) -> None:
        route_map = _validated_route_map(routes)
        method_map = _validated_method_map(methods)
        object.__setattr__(self, "routes", MappingProxyType(route_map))
        object.__setattr__(self, "methods", MappingProxyType(method_map))


@dataclass(frozen=True, slots=True)
class RpcReply:
    code: RpcCode
    body: bytes = b""


class _InvalidEnvelope(Exception):
    pass


@dataclass(slots=True)
class _DecodeBudget:
    nodes: int = 0
    text_bytes: int = 0


def _invalid_envelope() -> Never:
    raise _InvalidEnvelope


def _add_text(text: str, budget: _DecodeBudget) -> None:
    try:
        size = len(text.encode("utf-8"))
    except UnicodeEncodeError:
        _invalid_envelope()
    if size > _MAX_TEXT_BYTES - budget.text_bytes:
        _invalid_envelope()
    budget.text_bytes += size


def _freeze_node(
    value: object,
    *,
    depth: int,
    budget: _DecodeBudget,
    seen_containers: set[int],
) -> FrozenNode:
    budget.nodes += 1
    if budget.nodes > _MAX_DECODED_NODES or depth > _MAX_DECODED_DEPTH:
        _invalid_envelope()

    if value is None or type(value) is bool:
        return value
    if type(value) is int:
        if value.bit_length() > _MAX_INTEGER_BITS:
            _invalid_envelope()
        return value
    if type(value) is float:
        if not math.isfinite(value):
            _invalid_envelope()
        return value
    if type(value) is str:
        _add_text(value, budget)
        return value
    if type(value) not in (list, dict):
        _invalid_envelope()

    identity = id(value)
    if identity in seen_containers:
        _invalid_envelope()
    seen_containers.add(identity)

    if type(value) is list:
        return tuple(
            _freeze_node(
                item,
                depth=depth + 1,
                budget=budget,
                seen_containers=seen_containers,
            )
            for item in value
        )

    frozen: dict[str, FrozenNode] = {}
    for key, item in value.items():
        if type(key) is not str:
            _invalid_envelope()
        _add_text(key, budget)
        frozen[key] = _freeze_node(
            item,
            depth=depth + 1,
            budget=budget,
            seen_containers=seen_containers,
        )
    return MappingProxyType(frozen)


def _valid_path(path: object) -> bool:
    return type(path) is str and len(path) <= _MAX_PATH_BYTES and _PATH.fullmatch(path) is not None


def _valid_method_name(name: object) -> bool:
    return (
        type(name) is str and len(name) <= _MAX_METHOD_BYTES and _METHOD.fullmatch(name) is not None
    )


def _validated_route_map(routes: object) -> dict[str, CodecRoute]:
    if type(routes) is not tuple or not 1 <= len(routes) <= _MAX_ROUTES:
        raise ValueError("invalid route registry")

    route_map: dict[str, CodecRoute] = {}
    for route in routes:
        if type(route) is not CodecRoute:
            raise TypeError("invalid route definition")
        if not _valid_path(route.path):
            raise ValueError("invalid route path")
        if route.path in route_map:
            raise ValueError("duplicate route path")
        if not callable(route.decoder) or not callable(route.encoder):
            raise TypeError("route codecs must be callable")
        route_map[route.path] = route
    return route_map


def _validated_method_map(methods: object) -> dict[str, RpcMethod]:
    if type(methods) is not tuple or not 1 <= len(methods) <= _MAX_METHODS:
        raise ValueError("invalid method dispatcher")

    method_map: dict[str, RpcMethod] = {}
    for method in methods:
        if type(method) is not RpcMethod:
            raise TypeError("invalid method definition")
        if not _valid_method_name(method.name):
            raise ValueError("invalid method name")
        if method.name in method_map:
            raise ValueError("duplicate method name")
        if not callable(method.handler):
            raise TypeError("method handler must be callable")
        errors = method.application_errors
        if type(errors) is not tuple or len(errors) > _MAX_APPLICATION_ERRORS:
            raise ValueError("invalid application error declaration")
        if len(errors) != len(set(errors)):
            raise ValueError("duplicate application error declaration")
        for error_type in errors:
            if (
                type(error_type) is not type
                or not issubclass(error_type, Exception)
                or error_type is Exception
            ):
                raise TypeError("application errors must be narrow exception classes")
        method_map[method.name] = method
    return method_map


def build_rpc_dispatcher(
    routes: tuple[CodecRoute, ...],
    methods: tuple[RpcMethod, ...],
) -> RpcDispatcher:
    return RpcDispatcher(routes, methods)


def _read_envelope(decoded: object) -> tuple[object, tuple[FrozenNode, ...]]:
    if type(decoded) is not dict or set(decoded) != {"method", "arguments"}:
        _invalid_envelope()
    arguments = decoded["arguments"]
    if type(arguments) is not list or len(arguments) > _MAX_ARGUMENTS:
        _invalid_envelope()

    frozen = _freeze_node(
        decoded,
        depth=1,
        budget=_DecodeBudget(),
        seen_containers=set(),
    )
    return decoded["method"], frozen["arguments"]


def dispatch_rpc(
    dispatcher: RpcDispatcher,
    path: str,
    request: bytes,
) -> RpcReply:
    if type(dispatcher) is not RpcDispatcher:
        raise TypeError("dispatcher must be an exact RpcDispatcher")
    if not _valid_path(path):
        return RpcReply(RpcCode.ROUTE)
    route = dispatcher.routes.get(path)
    if route is None:
        return RpcReply(RpcCode.ROUTE)
    if type(request) is not bytes or not 1 <= len(request) <= _MAX_REQUEST_BYTES:
        return RpcReply(RpcCode.PARSE)

    try:
        decoded = route.decoder(request)
        method_name, arguments = _read_envelope(decoded)
    except Exception:
        return RpcReply(RpcCode.PARSE)

    if not _valid_method_name(method_name):
        return RpcReply(RpcCode.METHOD)
    method = dispatcher.methods.get(method_name)
    if method is None:
        return RpcReply(RpcCode.METHOD)

    try:
        result = method.handler(arguments)
    except Exception as error:
        code = RpcCode.APPLICATION if type(error) in method.application_errors else RpcCode.HANDLER
        return RpcReply(code)

    try:
        body = route.encoder(result)
    except Exception:
        return RpcReply(RpcCode.ENCODING)
    if type(body) is not bytes or not 1 <= len(body) <= _MAX_RESPONSE_BYTES:
        return RpcReply(RpcCode.ENCODING)
    return RpcReply(RpcCode.OK, body)
```

## Example

```python
def _unique_object(pairs: list[tuple[str, object]]) -> dict[str, object]:
    result: dict[str, object] = {}
    for key, value in pairs:
        if key in result:
            raise ValueError("duplicate object member")
        result[key] = value
    return result


def _reject_constant(_value: str) -> Never:
    raise ValueError("non-finite number")


def decode_json(payload: bytes) -> object:
    import json

    text = payload.decode("utf-8")
    return json.loads(
        text,
        object_pairs_hook=_unique_object,
        parse_constant=_reject_constant,
    )


def encode_json(value: object) -> bytes:
    import json

    return json.dumps(
        value,
        allow_nan=False,
        ensure_ascii=True,
        separators=(",", ":"),
    ).encode("ascii")


class InvalidArguments(Exception):
    pass


def add(arguments: tuple[FrozenNode, ...]) -> object:
    if len(arguments) != 2 or any(type(value) is not int for value in arguments):
        raise InvalidArguments
    return {"result": arguments[0] + arguments[1]}


def fail_unexpectedly(_arguments: tuple[FrozenNode, ...]) -> object:
    raise RuntimeError


def return_unencodable(_arguments: tuple[FrozenNode, ...]) -> object:
    return object()


dispatcher = build_rpc_dispatcher(
    (CodecRoute("/v1/json", decode_json, encode_json),),
    (
        RpcMethod("math.add", add, (InvalidArguments,)),
        RpcMethod("system.fail", fail_unexpectedly),
        RpcMethod("system.unencodable", return_unencodable),
    ),
)

try:
    RpcDispatcher(dict(dispatcher.routes), dict(dispatcher.methods))
except ValueError:
    mutable_registry_rejected = True
else:
    mutable_registry_rejected = False

requests = (
    dispatch_rpc(
        dispatcher,
        "/v1/json",
        b'{"method":"math.add","arguments":[2,3]}',
    ),
    dispatch_rpc(dispatcher, "/v1//json", b"{}"),
    dispatch_rpc(
        dispatcher,
        "/v1/json",
        b'{"method":"math.add","method":"math.add","arguments":[]}',
    ),
    dispatch_rpc(
        dispatcher,
        "/v1/json",
        b'{"method":"Math.add","arguments":[]}',
    ),
    dispatch_rpc(
        dispatcher,
        "/v1/json",
        b'{"method":"math.add","arguments":[2]}',
    ),
    dispatch_rpc(
        dispatcher,
        "/v1/json",
        b'{"method":"system.fail","arguments":[]}',
    ),
    dispatch_rpc(
        dispatcher,
        "/v1/json",
        b'{"method":"system.unencodable","arguments":[]}',
    ),
)

try:
    dispatcher.routes["/v1/other"] = dispatcher.routes["/v1/json"]
except TypeError:
    registry_is_read_only = True
else:
    registry_is_read_only = False

assert (requests, mutable_registry_rejected, registry_is_read_only) == (
    (
        RpcReply(RpcCode.OK, b'{"result":5}'),
        RpcReply(RpcCode.ROUTE),
        RpcReply(RpcCode.PARSE),
        RpcReply(RpcCode.METHOD),
        RpcReply(RpcCode.APPLICATION),
        RpcReply(RpcCode.HANDLER),
        RpcReply(RpcCode.ENCODING),
    ),
    True,
    True,
)
```

## Trade-offs and Limitations

The validating constructor accepts only exact bounded tuples and stores
read-only defensive snapshots, but their registered callables are trusted
application code. Exact exception-class matching keeps a declared application
failure from swallowing unrelated subclasses; ordinary unexpected exceptions
become `handler`, while control-flow `BaseException` instances still propagate.
All request errors discard exception text and values, which is predictable but
intentionally offers little diagnostic detail.

The frozen decoded tree rejects aliases and cycles and accepts only exact JSON-
like built-in values. Decoder-specific concerns such as duplicate members must
still be enforced before a dictionary loses that information. The limits are
process-wide constants in this example; a production composition root may need
reviewed per-protocol limits while preserving the same closed dispatch rules.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Bounded Three-State JSON Response Envelope](parse-a-bounded-three-state-json-response-envelope.md)
- [Parse a Bounded XML Envelope with Closed Variant Dispatch](../configuration-serialization/parse-a-bounded-xml-envelope-with-closed-variant-dispatch.md)
- [Dispatch Named Strategies with an Explicit Function Mapping](../python-language/dispatch-named-strategies-with-an-explicit-function-mapping.md)
<!-- catalog:related:end -->
