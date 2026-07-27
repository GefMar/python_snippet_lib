---
title: "Release a Pooled Response Connection Only After Clean EOF"
snippet_type: pattern
use_cases:
  - lifecycle-management
  - networking
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - resume-a-bounded-http-byte-stream-with-validated-range-responses.md
  - read-and-write-size-capped-varint-frames.md
  - ../concurrency-lifecycle/drain-bounded-deferred-writes-outside-the-queue-lock.md
---

# Release a Pooled Response Connection Only After Clean EOF

## Idea and Problem

Return a borrowed blocking connection to its pool only after a bounded consumer observes protocol-complete EOF and the adapter proves that the next cycle is reusable.

The iterator owns one response lifecycle. It releases the connection after a
positive-size read returns `b""`, protocol preparation succeeds, reuse is
allowed, and the response does not require closure. Every other terminal path
discards the connection exactly once.

## When to Use

Use this pattern inside a synchronous HTTP/1.x-like adapter whose body reader
has blocking semantics and knows when the framed response is complete. The
pool callbacks must perform synchronous, non-raising ownership transfer. Set
conservative byte and chunk limits, and consume the iterator to exhaustion
when reuse matters.

Use the networking library's native response context manager when it already
enforces equivalent pool rules. Do not wrap close-delimited responses, async
streams, pipelined traffic, or multiplexed protocols with this abstraction.
Their EOF and connection-ownership contracts differ.

## Implementation

```python
from collections.abc import Callable, Iterator
from enum import Enum, auto
from types import TracebackType
from typing import Protocol, Self


_MAX_CHUNK_SIZE = 1024 * 1024
_MAX_BODY_BYTES = 64 * 1024 * 1024
_MAX_BODY_CHUNKS = 1_000_000


class ResponseBodyError(RuntimeError):
    pass


class ResponseBodyLimitError(ResponseBodyError):
    pass


class ResponseState(Enum):
    OPEN = auto()
    RELEASED = auto()
    DISCARDED = auto()


class ReusableResponseAdapter(Protocol):
    def read(self, size: int, /) -> bytes: ...

    def prepare_next_cycle(self) -> None: ...

    def is_reusable(self) -> bool: ...

    def must_close(self) -> bool: ...


def _integer_limit(value: int, *, name: str, lower: int, upper: int) -> int:
    if isinstance(value, bool) or not isinstance(value, int):
        raise TypeError(f"{name} must be an integer")
    if not lower <= value <= upper:
        raise ValueError(f"{name} is outside the supported range")
    return value


class PooledResponseBody(Iterator[bytes]):
    def __init__(
        self,
        adapter: ReusableResponseAdapter,
        *,
        release: Callable[[], None],
        discard: Callable[[], None],
        chunk_size: int = 64 * 1024,
        max_bytes: int,
        max_chunks: int,
    ) -> None:
        for name, callback in (
            ("adapter.read", getattr(adapter, "read", None)),
            (
                "adapter.prepare_next_cycle",
                getattr(adapter, "prepare_next_cycle", None),
            ),
            ("adapter.is_reusable", getattr(adapter, "is_reusable", None)),
            ("adapter.must_close", getattr(adapter, "must_close", None)),
            ("release", release),
            ("discard", discard),
        ):
            if not callable(callback):
                raise TypeError(f"{name} must be callable")
        self._chunk_size = _integer_limit(
            chunk_size,
            name="chunk_size",
            lower=1,
            upper=_MAX_CHUNK_SIZE,
        )
        self._max_bytes = _integer_limit(
            max_bytes,
            name="max_bytes",
            lower=0,
            upper=_MAX_BODY_BYTES,
        )
        self._max_chunks = _integer_limit(
            max_chunks,
            name="max_chunks",
            lower=1,
            upper=_MAX_BODY_CHUNKS,
        )
        self._adapter = adapter
        self._release = release
        self._discard_callback = discard
        self._state = ResponseState.OPEN
        self._bytes_read = 0
        self._chunks_read = 0

    @property
    def state(self) -> ResponseState:
        return self._state

    def __iter__(self) -> Self:
        return self

    def _handoff(self, state: ResponseState, callback: Callable[[], None]) -> None:
        if self._state is not ResponseState.OPEN:
            return
        self._state = state
        callback()

    def _discard(self) -> None:
        self._handoff(ResponseState.DISCARDED, self._discard_callback)

    def __next__(self) -> bytes:
        if self._state is not ResponseState.OPEN:
            raise StopIteration

        try:
            chunk = self._adapter.read(self._chunk_size)
        except BaseException:
            self._discard()
            raise

        if not isinstance(chunk, bytes):
            self._discard()
            raise TypeError("response reads must return immutable bytes")

        if not chunk:
            try:
                self._adapter.prepare_next_cycle()
                reusable = self._adapter.is_reusable()
                must_close = self._adapter.must_close()
                if type(reusable) is not bool or type(must_close) is not bool:
                    raise TypeError("reuse decisions must be booleans")
                can_reuse = reusable and not must_close
            except BaseException:
                self._discard()
                raise
            if can_reuse:
                self._handoff(ResponseState.RELEASED, self._release)
            else:
                self._discard()
            raise StopIteration

        if (
            self._chunks_read >= self._max_chunks
            or len(chunk) > self._max_bytes - self._bytes_read
        ):
            self._discard()
            raise ResponseBodyLimitError("the response body limit was exceeded")
        self._chunks_read += 1
        self._bytes_read += len(chunk)
        return chunk

    def close(self) -> None:
        self._discard()

    def __enter__(self) -> Self:
        if self._state is not ResponseState.OPEN:
            raise ResponseBodyError("the response body is no longer open")
        return self

    def __exit__(
        self,
        exc_type: type[BaseException] | None,
        exc: BaseException | None,
        traceback: TracebackType | None,
    ) -> None:
        self.close()
```

## Example

```python
class FakeResponse:
    def __init__(
        self,
        chunks: list[bytes],
        *,
        reusable: bool = True,
        close_required: bool = False,
    ) -> None:
        self._chunks = chunks
        self._reusable = reusable
        self._close_required = close_required
        self.prepared = 0

    def read(self, size: int) -> bytes:
        assert size > 0
        return self._chunks.pop(0)

    def prepare_next_cycle(self) -> None:
        self.prepared += 1

    def is_reusable(self) -> bool:
        return self._reusable

    def must_close(self) -> bool:
        return self._close_required


events: list[str] = []
clean_adapter = FakeResponse([b"ab", b"cd", b""])
clean = PooledResponseBody(
    clean_adapter,
    release=lambda: events.append("released-clean"),
    discard=lambda: events.append("discarded-clean"),
    max_bytes=4,
    max_chunks=2,
)
clean_payload = b"".join(clean)
clean.close()

early = PooledResponseBody(
    FakeResponse([b"partial", b""]),
    release=lambda: events.append("released-early"),
    discard=lambda: events.append("discarded-early"),
    max_bytes=20,
    max_chunks=2,
)
with early:
    first_chunk = next(early)
early.close()

must_close = PooledResponseBody(
    FakeResponse([b"done", b""], close_required=True),
    release=lambda: events.append("released-must-close"),
    discard=lambda: events.append("discarded-must-close"),
    max_bytes=4,
    max_chunks=1,
)
must_close_payload = b"".join(must_close)

limited = PooledResponseBody(
    FakeResponse([b"large"]),
    release=lambda: events.append("released-limited"),
    discard=lambda: events.append("discarded-limited"),
    max_bytes=4,
    max_chunks=1,
)
try:
    next(limited)
except ResponseBodyLimitError:
    limit_raised = True
else:
    limit_raised = False

assert (
    clean_payload,
    clean.state,
    clean_adapter.prepared,
    first_chunk,
    early.state,
    must_close_payload,
    must_close.state,
    limit_raised,
    limited.state,
    events,
) == (
    b"abcd",
    ResponseState.RELEASED,
    1,
    b"partial",
    ResponseState.DISCARDED,
    b"done",
    ResponseState.DISCARDED,
    True,
    ResponseState.DISCARDED,
    [
        "released-clean",
        "discarded-early",
        "discarded-must-close",
        "discarded-limited",
    ],
)
```

## Trade-offs and Limitations

A short or final non-empty read does not prove completion; only the next
positive-size read returning `b""` does. Stopping immediately after the last
payload chunk therefore discards the connection. `close()` never drains the
body automatically, because draining can block or exceed limits. Close-delimited
responses cannot safely use this reusable-connection contract.

The byte cap applies to bytes returned by the injected reader, after any work
that adapter performs. This class does not parse framing, cap decompression,
set transport timeouts, retry reads, or implement a pool. It supports one
synchronous consumer and sequential protocol cycles only; it is not thread
safe, async, pipelined, or suitable for multiplexed HTTP/2 or HTTP/3 streams.
Terminal state is set before the ownership callback, preventing a second
handoff even if that callback violates its non-raising contract. There is no
finalizer, so callers must use the context manager or call `close()`.

## Related Snippets

<!-- catalog:related:start -->
- [Resume a Bounded HTTP Byte Stream with Validated Range Responses](resume-a-bounded-http-byte-stream-with-validated-range-responses.md)
- [Read and Write Size-Capped Varint Frames](read-and-write-size-capped-varint-frames.md)
- [Drain Bounded Deferred Writes Outside the Queue Lock](../concurrency-lifecycle/drain-bounded-deferred-writes-outside-the-queue-lock.md)
<!-- catalog:related:end -->
