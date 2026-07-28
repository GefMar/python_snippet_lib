---
title: "Authorize a Handler Through Explicit Request Extractors"
snippet_type: pattern
use_cases:
  - security
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - authorize-a-principal-against-a-closed-route-policy-map.md
  - authenticate-bounded-payloads-with-versioned-hmac-keys.md
---

# Authorize a Handler Through Explicit Request Extractors

## Idea and Problem

Gate one synchronous handler with bounded credential and resource extraction from the handler's own request arguments.

A frozen specification names every possible resource location. Each extractor
runs at most once, exactly one resource must be present, and one injected
authorizer receives the bounded values. Every missing, ambiguous, malformed,
exceptional, or negative outcome becomes the same fail-closed denial.

## When to Use

Use this pattern at a small framework boundary after request parsing, when the
decorated handler and all extractors can share the same positional and keyword
arguments. It is useful when a route can address one of several resource kinds
and request-specific extraction must stay separate from the authorization
decision.

This is extraction and enforcement, not policy evaluation. The related closed
route-policy recipe starts with an established principal and route and decides
permissions from a local policy map; this decorator instead obtains bounded
request values and delegates the decision to an injected authorizer.

## Implementation

```python
import inspect
import re
from collections.abc import Callable
from dataclasses import dataclass
from functools import wraps
from typing import ParamSpec, Protocol, TypeVar

P = ParamSpec("P")
R = TypeVar("R")
Extractor = Callable[..., object]
Authorizer = Callable[[str, str, str], object]

_RESOURCE_NAME = re.compile(r"[a-z][a-z0-9_-]{0,47}", re.ASCII).fullmatch
_MAX_RESOURCE_EXTRACTORS = 8
_MAX_EXTRACTED_UTF8_BYTES = 512


class RequestAuthorizationDenied(PermissionError):
    __slots__ = ()

    def __init__(self) -> None:
        super().__init__("request_not_authorized")


class _HandlerDecorator(Protocol[P]):
    def __call__(
        self,
        handler: Callable[P, R],
        /,
    ) -> Callable[P, R]: ...


@dataclass(frozen=True, slots=True)
class _NamedResourceExtractor:
    name: str
    extract: Extractor


@dataclass(frozen=True, slots=True, init=False)
class RequestAuthorizationSpec:
    credential_extractor: Extractor
    resource_extractors: tuple[_NamedResourceExtractor, ...]
    authorizer: Authorizer

    def __init__(
        self,
        *,
        credential_extractor: Extractor,
        resource_extractors: dict[str, Extractor],
        authorizer: Authorizer,
    ) -> None:
        _require_synchronous_callable(credential_extractor, name="credential extractor")
        _require_synchronous_callable(authorizer, name="authorizer")
        if type(resource_extractors) is not dict:
            raise TypeError("resource extractors must be an exact dict")
        extractor_snapshot = resource_extractors.copy()
        if not 1 <= len(extractor_snapshot) <= _MAX_RESOURCE_EXTRACTORS:
            raise ValueError("resource extractor count is outside the supported range")

        frozen: list[_NamedResourceExtractor] = []
        for name, extractor in extractor_snapshot.items():
            if type(name) is not str or _RESOURCE_NAME(name) is None:
                raise ValueError("resource names must use the supported ASCII syntax")
            _require_synchronous_callable(extractor, name="resource extractor")
            frozen.append(_NamedResourceExtractor(name, extractor))

        object.__setattr__(self, "credential_extractor", credential_extractor)
        object.__setattr__(
            self,
            "resource_extractors",
            tuple(sorted(frozen, key=lambda item: item.name)),
        )
        object.__setattr__(self, "authorizer", authorizer)


def _require_synchronous_callable(value: object, *, name: str) -> None:
    implementation = inspect.getattr_static(value, "__call__", None)
    if (
        not callable(value)
        or inspect.iscoroutinefunction(value)
        or inspect.isasyncgenfunction(value)
        or inspect.iscoroutinefunction(implementation)
        or inspect.isasyncgenfunction(implementation)
    ):
        raise TypeError(f"{name} must be a synchronous callable")


def _extract_once(
    extractor: Extractor,
    args: tuple[object, ...],
    kwargs: dict[str, object],
) -> tuple[bool, str | None]:
    try:
        value = extractor(*args, **kwargs)
    except Exception:
        return False, None
    if value is None:
        return True, None
    if type(value) is not str or not 1 <= len(value) <= _MAX_EXTRACTED_UTF8_BYTES:
        return False, None
    try:
        encoded_size = len(value.encode("utf-8", errors="strict"))
    except UnicodeEncodeError:
        return False, None
    if encoded_size > _MAX_EXTRACTED_UTF8_BYTES:
        return False, None
    return True, value


def _is_authorized(
    spec: RequestAuthorizationSpec,
    args: tuple[object, ...],
    kwargs: dict[str, object],
) -> bool:
    credential: str | None = None
    resource_name: str | None = None
    resource_value: str | None = None
    current_value: str | None = None
    decision: object = None
    try:
        credential_valid, credential = _extract_once(
            spec.credential_extractor,
            args,
            kwargs,
        )
        if not credential_valid or credential is None:
            return False

        for resource in spec.resource_extractors:
            value_valid, current_value = _extract_once(resource.extract, args, kwargs)
            if not value_valid:
                return False
            if current_value is None:
                continue
            if resource_value is not None:
                return False
            resource_name = resource.name
            resource_value = current_value

        if resource_name is None or resource_value is None:
            return False
        try:
            decision = spec.authorizer(credential, resource_name, resource_value)
        except Exception:
            return False
        return decision is True
    finally:
        credential = None
        resource_name = None
        resource_value = None
        current_value = None
        decision = None


def authorize_handler(
    spec: RequestAuthorizationSpec,
) -> _HandlerDecorator[P]:
    if type(spec) is not RequestAuthorizationSpec:
        raise TypeError("spec must be an exact RequestAuthorizationSpec")

    def decorate(handler: Callable[P, R]) -> Callable[P, R]:
        _require_synchronous_callable(handler, name="handler")

        @wraps(handler)
        def wrapped(*args: P.args, **kwargs: P.kwargs) -> R:
            if not _is_authorized(spec, args, kwargs):
                raise RequestAuthorizationDenied
            return handler(*args, **kwargs)

        return wrapped

    return decorate
```

## Example

```python
@dataclass(frozen=True, slots=True)
class Request:
    headers: dict[str, str]
    path_values: dict[str, str]


def credential_from_header(request: Request) -> str | None:
    return request.headers.get("Authorization")


def bin_from_path(request: Request) -> str | None:
    return request.path_values.get("bin")


def parcel_from_path(request: Request) -> str | None:
    return request.path_values.get("parcel")


def may_view(credential: str, resource_kind: str, resource_id: str) -> bool:
    return (credential, resource_kind, resource_id) == (
        "sample-credential",
        "parcel",
        "box-17",
    )


spec = RequestAuthorizationSpec(
    credential_extractor=credential_from_header,
    resource_extractors={"parcel": parcel_from_path, "bin": bin_from_path},
    authorizer=may_view,
)


@authorize_handler(spec)
def show_label(request: Request) -> str:
    return f"label:{request.path_values['parcel']}"


allowed = Request(
    headers={"Authorization": "sample-credential"},
    path_values={"parcel": "box-17"},
)
ambiguous = Request(
    headers={"Authorization": "sample-credential"},
    path_values={"parcel": "box-17", "bin": "north-2"},
)
try:
    show_label(ambiguous)
except RequestAuthorizationDenied as error:
    denial_code = str(error)
else:
    denial_code = "handler_ran"


class AsyncCredential:
    async def __call__(self, request: Request) -> str | None:
        return request.headers.get("Authorization")


class AsyncHandler:
    async def __call__(self, request: Request) -> str:
        return request.path_values["parcel"]


try:
    RequestAuthorizationSpec(
        credential_extractor=AsyncCredential(),
        resource_extractors={"parcel": parcel_from_path},
        authorizer=may_view,
    )
except TypeError:
    async_extractor_rejected = True
else:
    async_extractor_rejected = False

try:
    authorize_handler(spec)(AsyncHandler())
except TypeError:
    async_handler_rejected = True
else:
    async_handler_rejected = False

assert (
    show_label(allowed),
    denial_code,
    async_extractor_rejected,
    async_handler_rejected,
) == ("label:box-17", "request_not_authorized", True, True)
```

## Trade-offs and Limitations

Every extracted value is limited to 512 UTF-8 bytes. Construction accepts only
an exact `dict`, copies its one to eight entries, and then freezes a sorted
resource list; arbitrary mapping iterators therefore cannot exceed the callback
budget. The decorator is synchronous: a blocking authorizer adds request
latency, while an async framework needs a separately designed async gate.
Declared coroutine and async-generator functions, including callable-object
implementations, are rejected during construction or decoration. A nominally
synchronous callback must still honor its documented value contract, and a
handler's return contract remains the handler author's responsibility.
Normal failures deliberately lose diagnostic detail, so emit only
non-sensitive, bounded operational counters outside this wrapper if denials
need monitoring.

The immutable specification and per-call locals let the same wrapper serve
concurrent requests without retaining credential or resource values. Injected
extractors and the authorizer can still retain arguments or perform side
effects themselves; they must follow the same no-retention rule. `Exception`
is converted to denial, while process-control exceptions such as
`KeyboardInterrupt` and `SystemExit` still propagate. Handler exceptions also
propagate because authorization has already succeeded.

## Related Snippets

<!-- catalog:related:start -->
- [Authorize a Principal Against a Closed Route Policy Map](authorize-a-principal-against-a-closed-route-policy-map.md)
- [Authenticate Bounded Payloads with Versioned HMAC Keys](authenticate-bounded-payloads-with-versioned-hmac-keys.md)
<!-- catalog:related:end -->
