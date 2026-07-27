---
title: "Verify Ordered HTTP Client Expectations with Bounded Mismatch Reports"
snippet_type: testing-technique
use_cases:
  - networking
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../configuration-serialization/render-a-stable-unified-diff-for-nested-json-values.md
  - ../networking-protocols/build-a-canonical-http-origin-key.md
  - wait-for-named-queue-conditions-under-one-monotonic-deadline.md
---

# Verify Ordered HTTP Client Expectations with Bounded Mismatch Reports

## Idea and Problem

Test an HTTP client through a finite in-memory transport that consumes only exact ordered expectations and reports mismatches without echoing request values.

A transport-boundary fake can verify request construction without binding a
socket or depending on a server framework. Immutable request signatures retain
only a safe route plus digests of selected header values and the bounded body.
Mismatch details name differing fields and positions, so diagnostics remain
useful without copying credentials or payloads into an exception message.

## When to Use

Use this technique for one single-threaded unit test when the code under test
already accepts an injected send callable. Normalize the method, safe route,
selected headers, and body through the same helper on both sides. Declare a
finite FIFO of calls and predefined responses, then require final verification
so missing calls cannot pass silently.

Use `unittest.mock` call assertions when simple call recording is enough. Use a
real test server or protocol-specific transport when redirects, streaming,
timeouts, connection reuse, header combination rules, encoding, or network
failures are part of the behavior under test.

## Implementation

```python
import hashlib
import re
from collections.abc import Sequence
from dataclasses import dataclass


_MAX_EXPECTATIONS = 32
_MAX_TOTAL_CALLS = 64
_MAX_BODY_BYTES = 64 * 1024
_MAX_RESPONSE_BYTES = 64 * 1024
_MAX_TARGET_CHARACTERS = 256
_MAX_HEADERS = 16
_MAX_HEADER_VALUE_CHARACTERS = 1_024
_METHOD = re.compile(r"[A-Z][A-Z0-9-]{0,15}", re.ASCII)
_TARGET = re.compile(r"/[A-Za-z0-9._~!$&'()*+,;=:@%/-]{0,255}", re.ASCII)
_HEADER_NAME = re.compile(r"[a-z0-9!#$%&'*+.^_`|~-]{1,64}", re.ASCII)
_SHA256 = re.compile(r"[0-9a-f]{64}", re.ASCII)


@dataclass(frozen=True, slots=True)
class RequestSignature:
    method: str
    target: str
    header_digests: tuple[tuple[str, str], ...]
    body_sha256: str


@dataclass(frozen=True, slots=True)
class StubResponse:
    status: int
    body: bytes


@dataclass(frozen=True, slots=True)
class ExpectedCall:
    request: RequestSignature
    response: StubResponse
    times: int = 1


@dataclass(frozen=True, slots=True)
class MismatchDetail:
    kind: str
    call_number: int
    differing_fields: tuple[str, ...] = ()
    matching_call_number: int | None = None
    remaining_calls: int = 0

    def render(self) -> str:
        parts = [f"{self.kind} at call {self.call_number}"]
        if self.differing_fields:
            parts.append("fields=" + ",".join(self.differing_fields))
        if self.matching_call_number is not None:
            parts.append(f"matches call {self.matching_call_number}")
        if self.remaining_calls:
            parts.append(f"remaining={self.remaining_calls}")
        return "; ".join(parts)


class TransportExpectationError(AssertionError):
    def __init__(self, detail: MismatchDetail) -> None:
        self.detail = detail
        super().__init__(detail.render())


def _sha256(data: bytes) -> str:
    return hashlib.sha256(data).hexdigest()


def make_request_signature(
    method: str,
    target: str,
    *,
    headers: Sequence[tuple[str, str]] = (),
    body: bytes = b"",
) -> RequestSignature:
    if not isinstance(method, str) or _METHOD.fullmatch(method) is None:
        raise ValueError("method is not canonical uppercase ASCII")
    if (
        not isinstance(target, str)
        or len(target) > _MAX_TARGET_CHARACTERS
        or _TARGET.fullmatch(target) is None
    ):
        raise ValueError("target must be a bounded normalized path without a query")
    if not isinstance(headers, Sequence) or isinstance(headers, (str, bytes)):
        raise TypeError("headers must be a sequence")
    if len(headers) > _MAX_HEADERS:
        raise ValueError("header count exceeds the supported limit")
    if not isinstance(body, bytes):
        raise TypeError("body must be immutable bytes")
    if len(body) > _MAX_BODY_BYTES:
        raise ValueError("body exceeds the supported limit")

    digests: list[tuple[str, str]] = []
    names: set[str] = set()
    for header in headers:
        if not isinstance(header, tuple) or len(header) != 2:
            raise TypeError("each header must be a name-value tuple")
        name, value = header
        if not isinstance(name, str) or _HEADER_NAME.fullmatch(name) is None:
            raise ValueError("header names must be normalized lowercase ASCII")
        if name in names:
            raise ValueError("header names must be unique")
        if (
            not isinstance(value, str)
            or not 0 <= len(value) <= _MAX_HEADER_VALUE_CHARACTERS
            or not value.isascii()
            or not value.isprintable()
        ):
            raise ValueError("a header value is outside the supported format")
        names.add(name)
        digests.append((name, _sha256(value.encode("ascii"))))

    return RequestSignature(
        method=method,
        target=target,
        header_digests=tuple(sorted(digests)),
        body_sha256=_sha256(body),
    )


def _validate_signature(value: object) -> RequestSignature:
    if not isinstance(value, RequestSignature):
        raise TypeError("request must be a RequestSignature")
    if _METHOD.fullmatch(value.method) is None:
        raise ValueError("signature method is not canonical")
    if (
        len(value.target) > _MAX_TARGET_CHARACTERS
        or _TARGET.fullmatch(value.target) is None
    ):
        raise ValueError("signature target is not canonical")
    if not isinstance(value.header_digests, tuple):
        raise TypeError("signature header_digests must be a tuple")
    if not 0 <= len(value.header_digests) <= _MAX_HEADERS:
        raise ValueError("signature header count is outside the supported range")
    previous_name: str | None = None
    for pair in value.header_digests:
        if not isinstance(pair, tuple) or len(pair) != 2:
            raise TypeError("signature headers must contain pairs")
        name, digest = pair
        if (
            not isinstance(name, str)
            or _HEADER_NAME.fullmatch(name) is None
            or previous_name is not None
            and name <= previous_name
        ):
            raise ValueError("signature headers must be unique and sorted")
        if not isinstance(digest, str) or _SHA256.fullmatch(digest) is None:
            raise ValueError("signature contains an invalid header digest")
        previous_name = name
    if (
        not isinstance(value.body_sha256, str)
        or _SHA256.fullmatch(value.body_sha256) is None
    ):
        raise ValueError("signature contains an invalid body digest")
    return value


def _validate_response(value: object) -> StubResponse:
    if not isinstance(value, StubResponse):
        raise TypeError("response must be a StubResponse")
    if isinstance(value.status, bool) or not isinstance(value.status, int):
        raise TypeError("response status must be an integer")
    if not 100 <= value.status <= 599:
        raise ValueError("response status is outside the supported range")
    if not isinstance(value.body, bytes):
        raise TypeError("response body must be immutable bytes")
    if len(value.body) > _MAX_RESPONSE_BYTES:
        raise ValueError("response body exceeds the supported limit")
    return value


def _differing_fields(
    expected: RequestSignature,
    actual: RequestSignature,
) -> tuple[str, ...]:
    return tuple(
        field
        for field in ("method", "target", "header_digests", "body_sha256")
        if getattr(expected, field) != getattr(actual, field)
    )


class ExpectedHttpTransport:
    def __init__(self, expectations: Sequence[ExpectedCall]) -> None:
        if not isinstance(expectations, Sequence) or isinstance(
            expectations, (str, bytes)
        ):
            raise TypeError("expectations must be a sequence")
        if not 1 <= len(expectations) <= _MAX_EXPECTATIONS:
            raise ValueError("expectation count is outside the supported range")

        calls: list[tuple[RequestSignature, StubResponse]] = []
        for expectation in expectations:
            if not isinstance(expectation, ExpectedCall):
                raise TypeError("expectations must contain ExpectedCall values")
            request = _validate_signature(expectation.request)
            response = _validate_response(expectation.response)
            if isinstance(expectation.times, bool) or not isinstance(
                expectation.times, int
            ):
                raise TypeError("times must be an integer")
            if not 1 <= expectation.times <= _MAX_TOTAL_CALLS:
                raise ValueError("times is outside the supported range")
            if len(calls) + expectation.times > _MAX_TOTAL_CALLS:
                raise ValueError("expanded calls exceed the supported limit")
            calls.extend((request, response) for _ in range(expectation.times))

        self._calls = tuple(calls)
        self._position = 0

    @property
    def remaining(self) -> int:
        return len(self._calls) - self._position

    def send(self, request: RequestSignature) -> StubResponse:
        actual = _validate_signature(request)
        call_number = self._position + 1
        if self._position >= len(self._calls):
            raise TransportExpectationError(
                MismatchDetail(kind="extra-call", call_number=call_number)
            )

        expected, response = self._calls[self._position]
        if actual != expected:
            later_position = next(
                (
                    position
                    for position in range(self._position + 1, len(self._calls))
                    if self._calls[position][0] == actual
                ),
                None,
            )
            raise TransportExpectationError(
                MismatchDetail(
                    kind="out-of-order" if later_position is not None else "mismatch",
                    call_number=call_number,
                    differing_fields=_differing_fields(expected, actual),
                    matching_call_number=(
                        later_position + 1 if later_position is not None else None
                    ),
                )
            )

        self._position += 1
        return response

    def verify_complete(self) -> None:
        if self.remaining:
            raise TransportExpectationError(
                MismatchDetail(
                    kind="missing-call",
                    call_number=self._position + 1,
                    remaining_calls=self.remaining,
                )
            )
```

## Example

```python
expected_request = make_request_signature(
    "POST",
    "/items",
    headers=(("content-type", "application/json"),),
    body=b'{"name":"sample"}',
)
transport = ExpectedHttpTransport(
    (
        ExpectedCall(
            request=expected_request,
            response=StubResponse(status=202, body=b"queued"),
        ),
    )
)

wrong_request = make_request_signature("GET", "/items")
try:
    transport.send(wrong_request)
except TransportExpectationError as error:
    mismatch = error.detail

response = transport.send(expected_request)
transport.verify_complete()

assert (mismatch.kind, mismatch.differing_fields, response, transport.remaining) == (
    "mismatch",
    ("method", "header_digests", "body_sha256"),
    StubResponse(status=202, body=b"queued"),
    0,
)
```

## Trade-offs and Limitations

Strict FIFO expectations make ordering bugs visible but can make a unit test
brittle when call order is intentionally irrelevant. Repeated expectations are
expanded into a bounded immutable tuple for simple matching. A failed match
does not advance the cursor, and a later exact signature is reported as
out-of-order, but diagnostics deliberately omit all field values.

The fake is mutable and intentionally not thread-safe; give each single-threaded
test its own instance. Header and body digests avoid retaining those raw request
values, but equality still reveals whether two values match, and the safe
normalized target remains in memory. This is not an HTTP server and cannot
model wire semantics or demonstrate interoperability with a real client stack.

## Related Snippets

<!-- catalog:related:start -->
- [Render a Stable Unified Diff for Nested JSON Values](../configuration-serialization/render-a-stable-unified-diff-for-nested-json-values.md)
- [Build a Canonical HTTP Origin Key](../networking-protocols/build-a-canonical-http-origin-key.md)
- [Wait for Named Queue Conditions Under One Monotonic Deadline](wait-for-named-queue-conditions-under-one-monotonic-deadline.md)
<!-- catalog:related:end -->
