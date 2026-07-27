---
title: "Choose Buffered or Streaming Multipart Encoding from Bounded Parts"
snippet_type: integration
use_cases:
  - networking
  - resource-management
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - encode-a-bounded-http-1-1-chunked-body-for-protocol-tests.md
  - parse-a-bounded-ascii-media-type-value.md
  - ../storage-databases/split-a-binary-stream-into-exclusively-created-numbered-parts.md
---

# Choose Buffered or Streaming Multipart Encoding from Bounded Parts

## Idea and Problem

Buffer a small multipart/form-data body only when every part is sized, otherwise expose one bounded, closeable stream without consuming one-shot sources during planning.

Static byte parts have an exact encoded length and can be joined below a memory
threshold. A streaming part makes the length unknown, while a large all-bytes
body can still be emitted incrementally with its known length. Conservative
ASCII metadata avoids header escaping and CRLF injection inside this deliberately
limited encoder.

## When to Use

Use this integration boundary when one caller must handle small byte fields and
large one-shot byte iterators through the same multipart/form-data shape. Every
part, chunk, encoded byte, and metadata field is bounded, and streaming callers
must keep the returned body in a context manager so source cleanup runs on
success, failure, or early exit.

Use a maintained HTTP client's multipart implementation for broad filename,
Unicode, nested multipart, retry, or transport interoperability. A random
boundary makes accidental collision unlikely but does not prove that hostile
stream content cannot contain a delimiter.

## Implementation

```python
import re
import secrets
from collections.abc import Callable, Iterable, Iterator
from dataclasses import dataclass, field
from itertools import islice


_MAX_PARTS = 32
_MAX_STREAM_CHUNKS = 8_192
_MAX_CHUNK_BYTES = 1024 * 1024
_MAX_BUFFER_BYTES = 4 * 1024 * 1024
_MAX_ENCODED_BYTES = 64 * 1024 * 1024
_BOUNDARY = re.compile(r"[A-Za-z0-9._-]{16,64}", re.ASCII)
_PARAMETER = re.compile(r"[A-Za-z0-9][A-Za-z0-9._-]{0,63}", re.ASCII)
_MEDIA_TYPE = re.compile(
    r"[a-z0-9][a-z0-9!#$&^_.+-]{0,63}/[a-z0-9][a-z0-9!#$&^_.+-]{0,63}",
    re.ASCII,
)


@dataclass(frozen=True, slots=True)
class BytesPart:
    name: str
    content: bytes = field(repr=False)
    filename: str | None = None
    content_type: str = "application/octet-stream"


@dataclass(frozen=True, slots=True)
class StreamPart:
    name: str
    chunks: Iterator[bytes] = field(repr=False)
    close_source: Callable[[], None] | None = field(default=None, repr=False)
    filename: str | None = None
    content_type: str = "application/octet-stream"


Part = BytesPart | StreamPart


@dataclass(frozen=True, slots=True)
class MultipartEncoding:
    content_type: str
    content_length: int | None
    body: bytes | "MultipartStream" = field(repr=False)


def _validate_positive_limit(value: int, *, name: str, upper: int) -> int:
    if isinstance(value, bool) or not isinstance(value, int):
        raise TypeError(f"{name} must be an integer")
    if not 1 <= value <= upper:
        raise ValueError(f"{name} is outside the supported range")
    return value


def _validate_metadata(part: Part) -> None:
    if _PARAMETER.fullmatch(part.name) is None:
        raise ValueError("part name is outside the conservative ASCII format")
    if part.filename is not None and _PARAMETER.fullmatch(part.filename) is None:
        raise ValueError("filename is outside the conservative ASCII format")
    if _MEDIA_TYPE.fullmatch(part.content_type) is None:
        raise ValueError("content_type must be a canonical type/subtype")


def _part_prefix(part: Part, boundary: bytes) -> bytes:
    disposition = f'Content-Disposition: form-data; name="{part.name}"'
    if part.filename is not None:
        disposition += f'; filename="{part.filename}"'
    return (
        b"--"
        + boundary
        + b"\r\n"
        + disposition.encode("ascii")
        + b"\r\nContent-Type: "
        + part.content_type.encode("ascii")
        + b"\r\n\r\n"
    )


def _closing_boundary(boundary: bytes) -> bytes:
    return b"--" + boundary + b"--\r\n"


def _static_length(parts: tuple[Part, ...], boundary: bytes) -> int | None:
    if any(isinstance(part, StreamPart) for part in parts):
        return None
    return sum(
        len(_part_prefix(part, boundary)) + len(part.content) + 2
        for part in parts
        if isinstance(part, BytesPart)
    ) + len(_closing_boundary(boundary))


class MultipartStream(Iterator[bytes]):
    def __init__(
        self,
        parts: tuple[Part, ...],
        boundary: bytes,
        *,
        max_encoded_bytes: int,
    ) -> None:
        self._parts = parts
        self._boundary = boundary
        self._max_encoded_bytes = max_encoded_bytes
        self._iterator = self._encode()
        self._closed = False

    def __iter__(self) -> "MultipartStream":
        return self

    def __next__(self) -> bytes:
        if self._closed:
            raise StopIteration
        try:
            return next(self._iterator)
        except StopIteration:
            self.close()
            raise
        except BaseException as error:
            try:
                self.close()
            except BaseException as close_error:
                raise BaseExceptionGroup(
                    "multipart iteration and cleanup failed",
                    (error, close_error),
                ) from None
            raise

    def __enter__(self) -> "MultipartStream":
        if self._closed:
            raise RuntimeError("multipart stream is already closed")
        return self

    def __exit__(
        self,
        _exception_type: object,
        exception: BaseException | None,
        _traceback: object,
    ) -> None:
        try:
            self.close()
        except BaseException as close_error:
            if exception is None:
                raise
            raise BaseExceptionGroup(
                "multipart body and cleanup failed",
                (exception, close_error),
            ) from None

    @property
    def closed(self) -> bool:
        return self._closed

    def _encode(self) -> Iterator[bytes]:
        emitted = 0
        stream_chunks = 0

        def bounded(fragment: bytes) -> bytes:
            nonlocal emitted
            emitted += len(fragment)
            if emitted > self._max_encoded_bytes:
                raise ValueError("multipart body exceeds max_encoded_bytes")
            return fragment

        for part in self._parts:
            yield bounded(_part_prefix(part, self._boundary))
            if isinstance(part, BytesPart):
                if part.content:
                    yield bounded(part.content)
            else:
                for chunk in part.chunks:
                    stream_chunks += 1
                    if stream_chunks > _MAX_STREAM_CHUNKS:
                        raise ValueError("multipart stream has too many chunks")
                    if not isinstance(chunk, bytes):
                        raise TypeError("stream chunks must be immutable bytes")
                    if not chunk:
                        raise ValueError("stream chunks must not be empty")
                    if len(chunk) > _MAX_CHUNK_BYTES:
                        raise ValueError("a stream chunk exceeds the supported size")
                    yield bounded(chunk)
            yield bounded(b"\r\n")
        yield bounded(_closing_boundary(self._boundary))

    def close(self) -> None:
        if self._closed:
            return
        self._closed = True
        errors: list[BaseException] = []
        try:
            self._iterator.close()
        except BaseException as error:
            errors.append(error)
        for part in self._parts:
            if isinstance(part, StreamPart):
                try:
                    close_iterator = getattr(part.chunks, "close", None)
                    if callable(close_iterator):
                        close_iterator()
                except BaseException as error:
                    errors.append(error)
                if part.close_source is not None:
                    try:
                        part.close_source()
                    except BaseException as error:
                        errors.append(error)
        if errors:
            raise BaseExceptionGroup("multipart source cleanup failed", errors)


def choose_multipart_encoding(
    parts: Iterable[Part],
    *,
    boundary: str | None = None,
    buffer_limit: int = 256 * 1024,
    max_encoded_bytes: int = 8 * 1024 * 1024,
) -> MultipartEncoding:
    buffer_limit = _validate_positive_limit(
        buffer_limit,
        name="buffer_limit",
        upper=_MAX_BUFFER_BYTES,
    )
    max_encoded_bytes = _validate_positive_limit(
        max_encoded_bytes,
        name="max_encoded_bytes",
        upper=_MAX_ENCODED_BYTES,
    )
    if buffer_limit > max_encoded_bytes:
        raise ValueError("buffer_limit must not exceed max_encoded_bytes")

    if isinstance(parts, (str, bytes)):
        raise TypeError("parts must be a non-text iterable")
    snapshot = tuple(islice(parts, _MAX_PARTS + 1))
    if not 1 <= len(snapshot) <= _MAX_PARTS:
        raise ValueError("part count is outside the supported range")

    stream_sources: set[int] = set()
    for part in snapshot:
        if not isinstance(part, (BytesPart, StreamPart)):
            raise TypeError("parts must contain BytesPart or StreamPart values")
        _validate_metadata(part)
        if isinstance(part, BytesPart):
            if not isinstance(part.content, bytes):
                raise TypeError("byte-part content must be immutable bytes")
        else:
            if not isinstance(part.chunks, Iterator):
                raise TypeError("stream-part chunks must be a one-shot iterator")
            if part.close_source is not None and not callable(part.close_source):
                raise TypeError("close_source must be callable or None")
            source_identity = id(part.chunks)
            if source_identity in stream_sources:
                raise ValueError("a stream iterator cannot back multiple parts")
            stream_sources.add(source_identity)

    selected_boundary = secrets.token_hex(24) if boundary is None else boundary
    if not isinstance(selected_boundary, str) or _BOUNDARY.fullmatch(
        selected_boundary
    ) is None:
        raise ValueError("boundary is outside the conservative ASCII format")
    boundary_bytes = selected_boundary.encode("ascii")

    for part in snapshot:
        if isinstance(part, BytesPart):
            delimiter = b"--" + boundary_bytes
            if part.content.startswith(delimiter) or b"\r\n" + delimiter in part.content:
                raise ValueError("a byte part contains the selected boundary delimiter")

    exact_length = _static_length(snapshot, boundary_bytes)
    if exact_length is not None and exact_length > max_encoded_bytes:
        raise ValueError("multipart body exceeds max_encoded_bytes")
    content_type = f"multipart/form-data; boundary={selected_boundary}"

    if exact_length is not None and exact_length <= buffer_limit:
        stream = MultipartStream(
            snapshot,
            boundary_bytes,
            max_encoded_bytes=max_encoded_bytes,
        )
        with stream:
            body = b"".join(stream)
        if len(body) != exact_length:
            raise AssertionError("multipart length calculation is inconsistent")
        return MultipartEncoding(content_type, exact_length, body)

    return MultipartEncoding(
        content_type=content_type,
        content_length=exact_length,
        body=MultipartStream(
            snapshot,
            boundary_bytes,
            max_encoded_bytes=max_encoded_bytes,
        ),
    )
```

## Example

```python
buffered = choose_multipart_encoding(
    (
        BytesPart(
            name="note",
            content=b"hello",
            content_type="text/plain",
        ),
    ),
    boundary="ExampleBoundary0123456789",
    buffer_limit=1_024,
)

closed: list[str] = []
streamed = choose_multipart_encoding(
    (
        BytesPart(name="meta", content=b"v1", content_type="text/plain"),
        StreamPart(
            name="payload",
            chunks=iter((b"alpha", b"beta")),
            close_source=lambda: closed.append("payload"),
            filename="sample.bin",
        ),
    ),
    boundary="AnotherBoundary0123456789",
    buffer_limit=64,
    max_encoded_bytes=2_048,
)
assert isinstance(streamed.body, MultipartStream)
with streamed.body as body_stream:
    streamed_bytes = b"".join(body_stream)

assert (
    isinstance(buffered.body, bytes),
    len(buffered.body) == buffered.content_length,
    buffered.body.endswith(b"--ExampleBoundary0123456789--\r\n"),
    streamed.content_length,
    streamed_bytes.endswith(b"--AnotherBoundary0123456789--\r\n"),
    closed,
) == (True, True, True, None, True, ["payload"])
```

## Trade-offs and Limitations

The conservative metadata grammar rejects valid international form names,
filename encodings, media-type parameters, and several interoperable boundary
characters. That deliberate restriction avoids inventing a general MIME header
encoder. Byte parts are checked for a delimiter at line start, but streamed
content cannot be scanned safely in advance; use a fresh random boundary and a
maintained library for adversarial sources.

A streamed validation or size failure can occur after earlier fragments were
sent, so a transport may have a partial request. Cleanup callbacks should be
idempotent and non-raising; closing a stream first calls its iterator's public
`close()` method when present, then its optional callback, and reports multiple
failures as an exception group. Ownership of stream sources transfers only
when this function returns successfully; after an argument-validation failure,
the caller must still close its sources. `content_length=None` means the encoder
cannot calculate a length—it does not itself select `Transfer-Encoding:
chunked`, retry a one-shot body, or close an HTTP request or response.

## Related Snippets

<!-- catalog:related:start -->
- [Encode a Bounded HTTP/1.1 Chunked Body for Protocol Tests](encode-a-bounded-http-1-1-chunked-body-for-protocol-tests.md)
- [Parse a Bounded ASCII Media Type Value](parse-a-bounded-ascii-media-type-value.md)
- [Split a Binary Stream into Exclusively Created Numbered Parts](../storage-databases/split-a-binary-stream-into-exclusively-created-numbered-parts.md)
<!-- catalog:related:end -->
