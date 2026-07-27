---
title: "Resume a Bounded HTTP Byte Stream with Validated Range Responses"
snippet_type: integration
use_cases:
  - networking
  - resource-management
  - retry-recovery
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - collect-matching-cursor-pages-with-an-explicit-page-budget.md
  - read-and-write-size-capped-varint-frames.md
---

# Resume a Bounded HTTP Byte Stream with Validated Range Responses

## Idea and Problem

Resume an interrupted bounded HTTP download only after proving that the ranged response continues the same representation at the exact next byte.

The adapter records a strong ETag and total length from the initial `200`
response. After a transient read failure it sends `Range` with `If-Range`, then
requires `206`, the same ETag, and a matching `Content-Range` before appending
more bytes. A request budget and an object-size cap bound all work.

## When to Use

Use this pattern behind an HTTP client adapter when a small immutable object
must be recovered in one process without duplicating bytes. The origin must
provide a strong ETag, `Content-Length`, byte-range support, and stable object
semantics. Adapt the `open_response(headers)` callback to set timeouts and
return a response whose body iterator yields immutable byte chunks.

Use a mature download client when you need redirects, persistent partial
files, authentication refresh, compressed transfer coding, parallel ranges,
or broad HTTP compatibility. Disable transparent content decoding unless the
adapter defines offsets in the decoded representation and validates them
consistently.

## Implementation

```python
import re
from collections.abc import Iterable, Mapping
from typing import Protocol


_MAX_OBJECT_BYTES = 64 * 1024 * 1024
_MAX_REQUESTS = 8
_MAX_RESPONSE_HEADERS = 128
_MAX_DECIMAL_HEADER_LENGTH = 20
_MAX_CONTENT_RANGE_LENGTH = 80
_CONTENT_RANGE = re.compile(r"bytes ([0-9]+)-([0-9]+)/([0-9]+)")


class HTTPResumeError(RuntimeError):
    pass


class HTTPResumeLimitError(HTTPResumeError):
    pass


class ByteResponse(Protocol):
    status: int
    headers: Mapping[str, str]

    def iter_bytes(self) -> Iterable[bytes]: ...

    def close(self) -> None: ...


class ResponseOpener(Protocol):
    def __call__(self, headers: Mapping[str, str], /) -> ByteResponse: ...


def _header(headers: Mapping[str, str], name: str) -> str:
    if not isinstance(headers, Mapping):
        raise TypeError("response headers must be a mapping")
    matches: list[object] = []
    for index, (key, value) in enumerate(headers.items(), start=1):
        if index > _MAX_RESPONSE_HEADERS:
            raise HTTPResumeError("response contains too many headers")
        if isinstance(key, str) and key.lower() == name.lower():
            matches.append(value)
    if len(matches) != 1 or not isinstance(matches[0], str):
        raise HTTPResumeError(f"response must contain one {name} header")
    return matches[0]


def _decimal_header(value: str, *, name: str) -> int:
    if (
        not value
        or len(value) > _MAX_DECIMAL_HEADER_LENGTH
        or not value.isascii()
        or not value.isdecimal()
    ):
        raise HTTPResumeError(f"{name} must be an unsigned decimal integer")
    return int(value)


def _strong_etag(headers: Mapping[str, str]) -> str:
    etag = _header(headers, "ETag")
    if (
        len(etag) > 200
        or etag.startswith("W/")
        or len(etag) < 2
        or etag[0] != '"'
        or etag[-1] != '"'
        or not etag.isascii()
    ):
        raise HTTPResumeError("a bounded strong ETag is required")
    opaque = etag[1:-1]
    if any(
        ord(character) != 0x21
        and not 0x23 <= ord(character) <= 0x7E
        for character in opaque
    ):
        raise HTTPResumeError("a bounded strong ETag is required")
    return etag


def _validate_limits(max_bytes: int, max_requests: int) -> None:
    for name, value, lower, upper in (
        ("max_bytes", max_bytes, 0, _MAX_OBJECT_BYTES),
        ("max_requests", max_requests, 1, _MAX_REQUESTS),
    ):
        if isinstance(value, bool) or not isinstance(value, int):
            raise TypeError(f"{name} must be an integer")
        if not lower <= value <= upper:
            raise ValueError(f"{name} is outside the supported range")


def _validate_retry_types(
    retry_on: tuple[type[Exception], ...],
) -> None:
    if (
        not isinstance(retry_on, tuple)
        or not retry_on
        or len(retry_on) > 8
        or any(
            not isinstance(error_type, type)
            or not issubclass(error_type, Exception)
            for error_type in retry_on
        )
    ):
        raise TypeError("retry_on must contain 1 to 8 Exception classes")


def download_with_validated_ranges(
    open_response: ResponseOpener,
    *,
    max_bytes: int,
    max_requests: int = 3,
    retry_on: tuple[type[Exception], ...] = (OSError,),
) -> bytes:
    if not callable(open_response):
        raise TypeError("open_response must be callable")
    _validate_limits(max_bytes, max_requests)
    _validate_retry_types(retry_on)

    result = bytearray()
    etag: str | None = None
    total: int | None = None

    for _request_number in range(max_requests):
        offset = len(result)
        request_headers: dict[str, str] = {}
        if offset:
            if etag is None or total is None:
                raise AssertionError("resume metadata must already be known")
            request_headers = {
                "Range": f"bytes={offset}-",
                "If-Range": etag,
            }

        response: ByteResponse | None = None
        try:
            response = open_response(request_headers)
            status = response.status
            if isinstance(status, bool) or not isinstance(status, int):
                raise HTTPResumeError("response status must be an integer")

            response_etag = _strong_etag(response.headers)
            if offset == 0:
                if status != 200:
                    raise HTTPResumeError("initial response must have status 200")
                etag = response_etag
                total = _decimal_header(
                    _header(response.headers, "Content-Length"),
                    name="Content-Length",
                )
                if total > max_bytes:
                    raise HTTPResumeLimitError("object exceeds max_bytes")
                segment_end = total - 1
            else:
                if status != 206:
                    raise HTTPResumeError("resumed response must have status 206")
                if response_etag != etag:
                    raise HTTPResumeError("the response representation changed")
                content_range = _header(response.headers, "Content-Range")
                if len(content_range) > _MAX_CONTENT_RANGE_LENGTH:
                    raise HTTPResumeError("Content-Range is too long")
                match = _CONTENT_RANGE.fullmatch(content_range)
                if match is None:
                    raise HTTPResumeError("Content-Range is not canonical")
                range_start, segment_end, response_total = map(
                    int,
                    match.groups(),
                )
                if (
                    range_start != offset
                    or total is None
                    or response_total != total
                    or segment_end < range_start
                    or segment_end >= total
                ):
                    raise HTTPResumeError("Content-Range does not continue the object")
                content_length = _decimal_header(
                    _header(response.headers, "Content-Length"),
                    name="Content-Length",
                )
                if content_length != segment_end - range_start + 1:
                    raise HTTPResumeError("Content-Length disagrees with Content-Range")

            expected_end = segment_end + 1
            for chunk in response.iter_bytes():
                if not isinstance(chunk, bytes):
                    raise TypeError("response chunks must be immutable bytes")
                if not chunk:
                    raise HTTPResumeError("response produced an empty chunk")
                if len(result) + len(chunk) > expected_end:
                    raise HTTPResumeError("response exceeded its declared byte range")
                result.extend(chunk)

            if len(result) != expected_end:
                raise HTTPResumeError("response ended before its declared byte range")
            if total is not None and len(result) == total:
                return bytes(result)
        except (HTTPResumeError, TypeError, AttributeError):
            raise
        except retry_on:
            pass
        finally:
            if response is not None:
                response.close()

    raise HTTPResumeLimitError("max_requests was exhausted before completion")
```

## Example

```python
class FakeResponse:
    def __init__(
        self,
        status: int,
        headers: Mapping[str, str],
        chunks: tuple[bytes, ...],
        *,
        fail_after_chunks: bool = False,
    ) -> None:
        self.status = status
        self.headers = headers
        self._chunks = chunks
        self._fail_after_chunks = fail_after_chunks
        self.closed = False

    def iter_bytes(self) -> Iterable[bytes]:
        yield from self._chunks
        if self._fail_after_chunks:
            raise OSError("temporary read failure")

    def close(self) -> None:
        self.closed = True


responses = [
    FakeResponse(
        200,
        {"ETag": '"sample-v1"', "Content-Length": "11"},
        (b"hello ",),
        fail_after_chunks=True,
    ),
    FakeResponse(
        206,
        {
            "ETag": '"sample-v1"',
            "Content-Length": "5",
            "Content-Range": "bytes 6-10/11",
        },
        (b"world",),
    ),
]
requests: list[dict[str, str]] = []


def open_fake(headers: Mapping[str, str]) -> FakeResponse:
    requests.append(dict(headers))
    return responses[len(requests) - 1]


body = download_with_validated_ranges(
    open_fake,
    max_bytes=32,
    max_requests=2,
)

assert (body, requests, [response.closed for response in responses]) == (
    b"hello world",
    [
        {},
        {"Range": "bytes=6-", "If-Range": '"sample-v1"'},
    ],
    [True, True],
)
```

## Trade-offs and Limitations

The complete object is retained in memory, bounded by `max_bytes`. A transient
failure can replay the current request, while bytes already accepted remain in
memory; this is in-process recovery, not a crash-safe downloader. The adapter
must set transport timeouts because a request or body iteration can otherwise
block despite the request count.

The parser intentionally accepts one conservative header form. It does not
handle multipart ranges, unknown totals, weak validators, redirects, transfer
decoding, or servers that ignore `Range`; accepting a `200` during resume would
silently duplicate the prefix. The server still controls the truthfulness of
its validator. Hitting any structural inconsistency fails the whole operation
rather than restarting from zero against a possibly different object.

## Related Snippets

<!-- catalog:related:start -->
- [Collect Matching Cursor Pages with an Explicit Page Budget](collect-matching-cursor-pages-with-an-explicit-page-budget.md)
- [Read and Write Size-Capped Varint Frames](read-and-write-size-capped-varint-frames.md)
<!-- catalog:related:end -->
