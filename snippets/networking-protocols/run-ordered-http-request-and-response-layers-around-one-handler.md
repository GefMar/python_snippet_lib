---
title: "Run Ordered HTTP Request and Response Layers Around One Handler"
snippet_type: pattern
use_cases:
  - data-transformation
  - networking
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - release-a-pooled-response-connection-only-after-clean-eof.md
  - ../python-language/dispatch-named-strategies-with-an-explicit-function-mapping.md
  - ../reliability-resilience/compensate-completed-workflow-steps-in-reverse-order.md
---

# Run Ordered HTTP Request and Response Layers Around One Handler

## Idea and Problem

Compose bounded synchronous HTTP transformations by running request callbacks forward, one terminal handler once, and response callbacks in reverse.

Frozen request, response, and layer records make the data flow explicit. The
runner validates the initial request and the entire layer structure before any
callback, then validates every transformed request and response before passing
it to the next stage.

## When to Use

Use this pattern for a small in-memory HTTP processing chain whose ordering and
failure boundaries must be obvious. Each layer has one request transformation
and one response transformation, and the terminal callable alone produces the
first response. Build the layer tuple from trusted callables before handling a
request.

The deliberately narrow value grammar suits adapters and protocol exercises
that have already parsed and bounded their inputs. Put guaranteed cleanup in a
separate context manager around the whole call; response callbacks run only
after the terminal has returned a valid response.

## Implementation

```python
from collections.abc import Callable
from dataclasses import dataclass
from typing import TypeAlias


_MAX_LAYERS = 16
_MAX_METHOD_LENGTH = 16
_MAX_TARGET_LENGTH = 256
_MAX_HEADERS = 32
_MAX_HEADER_NAME_LENGTH = 64
_MAX_HEADER_VALUE_LENGTH = 1_024
_MAX_BODY_LENGTH = 64 * 1_024
_TOKEN_CHARACTERS = frozenset(
    "!#$%&'*+-.^_`|~0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"
)

HeaderBlock: TypeAlias = tuple[tuple[str, str], ...]


class HttpValueError(ValueError):
    pass


@dataclass(frozen=True, slots=True)
class HttpRequest:
    method: str
    target: str
    headers: HeaderBlock
    body: bytes


@dataclass(frozen=True, slots=True)
class HttpResponse:
    status: int
    headers: HeaderBlock
    body: bytes


RequestTransform: TypeAlias = Callable[[HttpRequest], HttpRequest]
ResponseTransform: TypeAlias = Callable[[HttpResponse], HttpResponse]
TerminalHandler: TypeAlias = Callable[[HttpRequest], HttpResponse]


@dataclass(frozen=True, slots=True)
class HttpLayer:
    name: str
    before: RequestTransform
    after: ResponseTransform


def _require_token(value: object, *, field: str, maximum: int) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be text")
    if not 1 <= len(value) <= maximum:
        raise HttpValueError(f"{field} length is outside the supported range")
    if any(character not in _TOKEN_CHARACTERS for character in value):
        raise HttpValueError(f"{field} must contain only ASCII token characters")
    return value


def _validate_headers(headers: object, *, owner: str) -> None:
    if type(headers) is not tuple:
        raise TypeError(f"{owner} headers must be an exact tuple")
    if len(headers) > _MAX_HEADERS:
        raise HttpValueError(f"{owner} has too many headers")

    seen_names: set[str] = set()
    for index, pair in enumerate(headers):
        if type(pair) is not tuple:
            raise TypeError(f"{owner} header {index} must be an exact tuple")
        if len(pair) != 2:
            raise HttpValueError(f"{owner} header {index} must contain two items")
        name = _require_token(
            pair[0],
            field=f"{owner} header {index} name",
            maximum=_MAX_HEADER_NAME_LENGTH,
        )
        value = pair[1]
        if type(value) is not str:
            raise TypeError(f"{owner} header {index} value must be text")
        if len(value) > _MAX_HEADER_VALUE_LENGTH:
            raise HttpValueError(f"{owner} header {index} value is too long")
        if any(not 0x20 <= ord(character) <= 0x7E for character in value):
            raise HttpValueError(
                f"{owner} header {index} value must contain only printable ASCII"
            )
        normalized_name = name.lower()
        if normalized_name in seen_names:
            raise HttpValueError(f"{owner} header names must be unique ignoring case")
        seen_names.add(normalized_name)


def _validate_request(value: object) -> HttpRequest:
    if type(value) is not HttpRequest:
        raise TypeError("request callbacks must return an HttpRequest")
    method = _require_token(
        value.method,
        field="request method",
        maximum=_MAX_METHOD_LENGTH,
    )
    if method != method.upper():
        raise HttpValueError("request method must use canonical uppercase ASCII")
    if type(value.target) is not str:
        raise TypeError("request target must be text")
    if not 1 <= len(value.target) <= _MAX_TARGET_LENGTH:
        raise HttpValueError("request target length is outside the supported range")
    if any(not 0x21 <= ord(character) <= 0x7E for character in value.target):
        raise HttpValueError("request target must contain only visible ASCII")
    _validate_headers(value.headers, owner="request")
    if type(value.body) is not bytes:
        raise TypeError("request body must be exact bytes")
    if len(value.body) > _MAX_BODY_LENGTH:
        raise HttpValueError("request body is too large")
    return value


def _validate_response(value: object) -> HttpResponse:
    if type(value) is not HttpResponse:
        raise TypeError("terminal and response callbacks must return an HttpResponse")
    if type(value.status) is not int:
        raise TypeError("response status must be an integer")
    if not 100 <= value.status <= 599:
        raise HttpValueError("response status is outside the supported range")
    _validate_headers(value.headers, owner="response")
    if type(value.body) is not bytes:
        raise TypeError("response body must be exact bytes")
    if len(value.body) > _MAX_BODY_LENGTH:
        raise HttpValueError("response body is too large")
    return value


def _validate_layers(layers: object, terminal: object) -> None:
    if type(layers) is not tuple:
        raise TypeError("layers must be an exact tuple")
    if len(layers) > _MAX_LAYERS:
        raise HttpValueError("layer count exceeds the supported limit")
    if not callable(terminal):
        raise TypeError("terminal handler must be callable")

    names: set[str] = set()
    for index, layer in enumerate(layers):
        if type(layer) is not HttpLayer:
            raise TypeError(f"layer {index} must be an HttpLayer")
        name = _require_token(layer.name, field=f"layer {index} name", maximum=64)
        if name in names:
            raise HttpValueError("layer names must be unique")
        if not callable(layer.before):
            raise TypeError(f"layer {index} before callback must be callable")
        if not callable(layer.after):
            raise TypeError(f"layer {index} after callback must be callable")
        names.add(name)


def run_http_layers(
    request: HttpRequest,
    layers: tuple[HttpLayer, ...],
    terminal: TerminalHandler,
) -> HttpResponse:
    current_request = _validate_request(request)
    _validate_layers(layers, terminal)

    for layer in layers:
        current_request = _validate_request(layer.before(current_request))

    current_response = _validate_response(terminal(current_request))
    for layer in reversed(layers):
        current_response = _validate_response(layer.after(current_response))
    return current_response
```

## Example

```python
from dataclasses import replace


trace = []
terminal_requests = []


def outer_before(request: HttpRequest) -> HttpRequest:
    trace.append("outer:before")
    return replace(
        request,
        headers=request.headers + (("X-Flow", "request"),),
    )


def inner_before(request: HttpRequest) -> HttpRequest:
    trace.append("inner:before")
    return replace(request, target=request.target + "?view=compact")


def handle_request(request: HttpRequest) -> HttpResponse:
    trace.append("terminal")
    terminal_requests.append(request)
    return HttpResponse(200, (("Content-Type", "text/plain"),), b"core")


def inner_after(response: HttpResponse) -> HttpResponse:
    trace.append("inner:after")
    return replace(response, body=response.body + b"|inner")


def outer_after(response: HttpResponse) -> HttpResponse:
    trace.append("outer:after")
    return replace(
        response,
        headers=response.headers + (("X-Flow", "response"),),
    )


request = HttpRequest(
    "POST",
    "/resource",
    (("Accept", "text/plain"),),
    b"input",
)
layers = (
    HttpLayer("outer", outer_before, outer_after),
    HttpLayer("inner", inner_before, inner_after),
)
response = run_http_layers(request, layers, handle_request)

direct_trace = []


def direct_handler(current: HttpRequest) -> HttpResponse:
    direct_trace.append(current.target)
    return HttpResponse(204, (), b"")


direct_response = run_http_layers(request, (), direct_handler)


def unchanged_response(current: HttpResponse) -> HttpResponse:
    return current


def quiet_handler(current: HttpRequest) -> HttpResponse:
    return HttpResponse(200, (), b"ok")


def invalid_request_output(current: HttpRequest) -> HttpRequest:
    return replace(current, method="post")


def invalid_response_output(current: HttpResponse) -> HttpResponse:
    return replace(current, body=b"x" * (64 * 1_024 + 1))


invalid_output_calls = (
    lambda: run_http_layers(
        request,
        (HttpLayer("bad-request", invalid_request_output, unchanged_response),),
        quiet_handler,
    ),
    lambda: run_http_layers(request, (), lambda current: HttpResponse(99, (), b"")),
    lambda: run_http_layers(
        request,
        (HttpLayer("bad-response", lambda current: current, invalid_response_output),),
        quiet_handler,
    ),
)
invalid_output_rejections = 0
for invoke in invalid_output_calls:
    try:
        invoke()
    except HttpValueError:
        invalid_output_rejections += 1

structure_trace = []


def structure_before(current: HttpRequest) -> HttpRequest:
    structure_trace.append("called")
    return current


structure_cases = (
    (
        HttpLayer("first", structure_before, unchanged_response),
        HttpLayer("second", object(), unchanged_response),
    ),
    (
        HttpLayer("same", structure_before, unchanged_response),
        HttpLayer("same", structure_before, unchanged_response),
    ),
)
structure_rejections = 0
for candidate_layers in structure_cases:
    try:
        run_http_layers(request, candidate_layers, quiet_handler)
    except (TypeError, HttpValueError):
        structure_rejections += 1


def tracing_before(events: list[str], label: str) -> RequestTransform:
    def transform(current: HttpRequest) -> HttpRequest:
        events.append(f"{label}:before")
        return current

    return transform


def tracing_after(events: list[str], label: str) -> ResponseTransform:
    def transform(current: HttpResponse) -> HttpResponse:
        events.append(f"{label}:after")
        return current

    return transform


before_events = []
before_problem = RuntimeError("before stopped")


def fail_before(current: HttpRequest) -> HttpRequest:
    before_events.append("second:before")
    raise before_problem


def unexpected_terminal(current: HttpRequest) -> HttpResponse:
    before_events.append("terminal")
    return HttpResponse(200, (), b"")


try:
    run_http_layers(
        request,
        (
            HttpLayer(
                "first",
                tracing_before(before_events, "first"),
                tracing_after(before_events, "first"),
            ),
            HttpLayer("second", fail_before, tracing_after(before_events, "second")),
        ),
        unexpected_terminal,
    )
except RuntimeError as error:
    before_identity_preserved = error is before_problem

terminal_events = []
terminal_problem = LookupError("terminal stopped")
terminal_layers = (
    HttpLayer(
        "outer",
        tracing_before(terminal_events, "outer"),
        tracing_after(terminal_events, "outer"),
    ),
    HttpLayer(
        "inner",
        tracing_before(terminal_events, "inner"),
        tracing_after(terminal_events, "inner"),
    ),
)


def fail_terminal(current: HttpRequest) -> HttpResponse:
    terminal_events.append("terminal")
    raise terminal_problem


try:
    run_http_layers(request, terminal_layers, fail_terminal)
except LookupError as error:
    terminal_identity_preserved = error is terminal_problem

after_events = []
after_problem = OSError("response stopped")


def fail_middle_after(current: HttpResponse) -> HttpResponse:
    after_events.append("middle:after")
    raise after_problem


after_layers = (
    HttpLayer(
        "outer",
        tracing_before(after_events, "outer"),
        tracing_after(after_events, "outer"),
    ),
    HttpLayer(
        "middle",
        tracing_before(after_events, "middle"),
        fail_middle_after,
    ),
    HttpLayer(
        "inner",
        tracing_before(after_events, "inner"),
        tracing_after(after_events, "inner"),
    ),
)


def unwind_terminal(current: HttpRequest) -> HttpResponse:
    after_events.append("terminal")
    return HttpResponse(200, (), b"")


try:
    run_http_layers(request, after_layers, unwind_terminal)
except OSError as error:
    after_identity_preserved = error is after_problem

assert (
    trace,
    terminal_requests,
    response,
    direct_trace,
    direct_response,
    invalid_output_rejections,
    structure_rejections,
    structure_trace,
    before_events,
    before_identity_preserved,
    terminal_events,
    terminal_identity_preserved,
    after_events,
    after_identity_preserved,
) == (
    ["outer:before", "inner:before", "terminal", "inner:after", "outer:after"],
    [
        HttpRequest(
            "POST",
            "/resource?view=compact",
            (("Accept", "text/plain"), ("X-Flow", "request")),
            b"input",
        )
    ],
    HttpResponse(
        200,
        (("Content-Type", "text/plain"), ("X-Flow", "response")),
        b"core|inner",
    ),
    ["/resource"],
    HttpResponse(204, (), b""),
    3,
    2,
    [],
    ["first:before", "second:before"],
    True,
    ["outer:before", "inner:before", "terminal"],
    True,
    [
        "outer:before",
        "middle:before",
        "inner:before",
        "terminal",
        "inner:after",
        "middle:after",
    ],
    True,
)
```

## Trade-offs and Limitations

The strict value grammar intentionally rejects non-ASCII targets and header
values, duplicate header names, non-uppercase methods, large bodies, and large
layer chains. Each transformation commonly allocates a new frozen record and
may copy a body or header tuple, so this pattern is intended for bounded
in-memory messages rather than bulk transfer.

Callback exceptions propagate without wrapping. A failed request callback or
terminal call skips every response callback; a failed response callback stops
the reverse traversal. There is no implicit `finally` phase, rollback, or close
hook, so layers must not assume that reverse traversal owns resource cleanup.

The runner supplies no sockets or HTTP parser, asynchronous or streaming
execution, retry or redirect policy, cookies, caching, authentication, timeout,
cancellation, rollback, close operation, or resource ownership. It also does
not inspect callable signatures or recursively freeze state captured by trusted
callbacks.

## Related Snippets

<!-- catalog:related:start -->
- [Release a Pooled Response Connection Only After Clean EOF](release-a-pooled-response-connection-only-after-clean-eof.md)
- [Dispatch Named Strategies with an Explicit Function Mapping](../python-language/dispatch-named-strategies-with-an-explicit-function-mapping.md)
- [Compensate Completed Workflow Steps in Reverse Order](../reliability-resilience/compensate-completed-workflow-steps-in-reverse-order.md)
<!-- catalog:related:end -->
