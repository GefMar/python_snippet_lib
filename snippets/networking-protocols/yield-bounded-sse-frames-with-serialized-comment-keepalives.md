---
title: "Yield Bounded SSE Frames with Serialized Comment Keepalives"
snippet_type: integration
use_cases:
  - concurrency-control
  - networking
  - resource-management
  - serialization
tested_python:
  - "3.14"
dependencies: []
related:
  - encode-a-bounded-http-1-1-chunked-body-for-protocol-tests.md
  - ../reliability-resilience/propagate-a-monotonic-deadline-with-contextvar.md
  - ../data-processing/limit-text-lines-across-arbitrary-chunks.md
---

# Yield Bounded SSE Frames with Serialized Comment Keepalives

## Idea and Problem

Serialize bounded JSON events and idle comment keepalives through one async byte iterator so two tasks never write overlapping Server-Sent Event frames.

One pending source read survives repeated idle timeouts. A completed event
becomes one compact `data:` frame, while a timeout yields the protocol comment
`: keepalive`. Event, keepalive, and aggregate byte budgets stop a stalled or
unbounded source, and an explicit early close cancels the pending read before
the source is closed.

## When to Use

Use this integration helper behind a framework response that accepts an async
byte iterator and already sets the required event-stream headers. Events must
be small flat JSON objects, and one idle interval plus the total keepalive
budget must fit the connection policy. The source iterator must propagate
`CancelledError` promptly from a pending `__anext__`; cooperative cancellation
is part of this helper's lifecycle contract. A consumer that may stop early
must call `aclose()` in `finally` or use `contextlib.aclosing`; `break` alone is
not a lifecycle guarantee while the iterator remains referenced.

Use a maintained framework primitive when it owns disconnect detection,
backpressure, proxy behavior, or reconnection IDs. Use WebSockets or another
bidirectional protocol when the client must send messages over the same
channel. Application success and failure events belong to the producer rather
than this transport formatter.

## Implementation

```python
import asyncio
import json
import math
import re
from collections.abc import AsyncGenerator, AsyncIterator, Mapping
from typing import TypeAlias


JsonScalar: TypeAlias = str | int | float | bool | None
_MAX_EVENTS = 10_000
_MAX_KEEPALIVES = 1_000
_MAX_EVENT_FIELDS = 32
_MAX_EVENT_BYTES = 64 * 1024
_MAX_TOTAL_BYTES = 8 * 1024 * 1024
_MAX_TEXT_CHARACTERS = 4_096
_FIELD_NAME = re.compile(r"[A-Za-z][A-Za-z0-9._-]{0,63}", re.ASCII)
_KEEPALIVE_FRAME = b": keepalive\n\n"


def _positive_integer(value: object, *, name: str, upper: int) -> int:
    if isinstance(value, bool) or not isinstance(value, int):
        raise TypeError(f"{name} must be an integer")
    if not 1 <= value <= upper:
        raise ValueError(f"{name} is outside the supported range")
    return value


def _idle_interval(value: object) -> float:
    if (
        isinstance(value, bool)
        or not isinstance(value, (int, float))
        or not math.isfinite(value)
        or not 0.001 <= value <= 300.0
    ):
        raise ValueError("idle_seconds is outside the supported range")
    return float(value)


def _event_frame(value: object, *, maximum_bytes: int) -> bytes:
    if not isinstance(value, Mapping):
        raise TypeError("SSE events must be mappings")
    if not 1 <= len(value) <= _MAX_EVENT_FIELDS:
        raise ValueError("SSE event field count is outside the supported range")

    event: dict[str, JsonScalar] = {}
    for key, item in value.items():
        if not isinstance(key, str) or _FIELD_NAME.fullmatch(key) is None:
            raise ValueError("an SSE event field name is not canonical")
        if isinstance(item, str):
            if len(item) > _MAX_TEXT_CHARACTERS:
                raise ValueError("an SSE event text value is too large")
        elif isinstance(item, bool) or item is None:
            pass
        elif isinstance(item, int):
            if not -(2**53) <= item <= 2**53:
                raise ValueError("an SSE event integer is outside the supported range")
        elif isinstance(item, float):
            if not math.isfinite(item):
                raise ValueError("an SSE event number must be finite")
        else:
            raise TypeError("SSE event values must be flat JSON scalars")
        event[key] = item

    payload = json.dumps(
        event,
        ensure_ascii=False,
        allow_nan=False,
        sort_keys=True,
        separators=(",", ":"),
    ).encode("utf-8")
    if len(payload) > maximum_bytes:
        raise ValueError("an SSE event exceeds max_event_bytes")
    return b"data: " + payload + b"\n\n"


async def yield_sse_frames(
    events: AsyncIterator[Mapping[str, JsonScalar]],
    *,
    idle_seconds: float = 15.0,
    max_events: int = 1_000,
    max_keepalives: int = 100,
    max_event_bytes: int = 16 * 1024,
    max_total_bytes: int = 1024 * 1024,
) -> AsyncGenerator[bytes, None]:
    if not isinstance(events, AsyncIterator):
        raise TypeError("events must be an async iterator")
    idle_seconds = _idle_interval(idle_seconds)
    max_events = _positive_integer(max_events, name="max_events", upper=_MAX_EVENTS)
    max_keepalives = _positive_integer(
        max_keepalives,
        name="max_keepalives",
        upper=_MAX_KEEPALIVES,
    )
    max_event_bytes = _positive_integer(
        max_event_bytes,
        name="max_event_bytes",
        upper=_MAX_EVENT_BYTES,
    )
    max_total_bytes = _positive_integer(
        max_total_bytes,
        name="max_total_bytes",
        upper=_MAX_TOTAL_BYTES,
    )
    close_source = getattr(events, "aclose", None)
    if close_source is not None and not callable(close_source):
        raise TypeError("events.aclose must be callable when present")

    pending: asyncio.Task[Mapping[str, JsonScalar]] | None = None
    event_count = 0
    keepalive_count = 0
    emitted_bytes = 0
    operation_error: BaseException | None = None
    operation_traceback = None

    async def next_event() -> Mapping[str, JsonScalar]:
        return await anext(events)

    try:
        while True:
            if pending is None:
                pending = asyncio.create_task(next_event())
            done, _ = await asyncio.wait((pending,), timeout=idle_seconds)
            if pending not in done:
                keepalive_count += 1
                if keepalive_count > max_keepalives:
                    raise TimeoutError("SSE keepalive budget was exhausted")
                frame = _KEEPALIVE_FRAME
            else:
                completed = pending
                pending = None
                try:
                    event = completed.result()
                except StopAsyncIteration:
                    break
                event_count += 1
                if event_count > max_events:
                    raise ValueError("SSE event count exceeds max_events")
                frame = _event_frame(event, maximum_bytes=max_event_bytes)

            emitted_bytes += len(frame)
            if emitted_bytes > max_total_bytes:
                raise ValueError("SSE output exceeds max_total_bytes")
            yield frame
    except BaseException as error:
        operation_error = error
        operation_traceback = error.__traceback__

    cleanup_errors: list[BaseException] = []
    if pending is not None:
        if not pending.done():
            pending.cancel()
        try:
            await pending
        except (asyncio.CancelledError, StopAsyncIteration):
            pass
        except BaseException as error:
            cleanup_errors.append(error)
    if close_source is not None:
        try:
            await close_source()
        except BaseException as error:
            cleanup_errors.append(error)

    if operation_error is not None:
        if cleanup_errors:
            raise BaseExceptionGroup(
                "SSE iteration and cleanup failed",
                (operation_error, *cleanup_errors),
            ) from None
        raise operation_error.with_traceback(operation_traceback)
    if cleanup_errors:
        raise BaseExceptionGroup("SSE source cleanup failed", cleanup_errors)
```

## Example

```python
class StalledEvents(AsyncIterator[Mapping[str, JsonScalar]]):
    def __init__(self) -> None:
        self.index = 0
        self.gate = asyncio.Event()
        self.cancelled = False
        self.closed = False

    def __aiter__(self) -> "StalledEvents":
        return self

    async def __anext__(self) -> Mapping[str, JsonScalar]:
        if self.index == 0:
            self.index += 1
            return {"progress": 25, "status": "running"}
        try:
            await self.gate.wait()
        except asyncio.CancelledError:
            self.cancelled = True
            raise
        raise StopAsyncIteration

    async def aclose(self) -> None:
        self.closed = True
        self.gate.set()


async def collect_example() -> tuple[list[bytes], bool, bool]:
    source = StalledEvents()
    stream = yield_sse_frames(
        source,
        idle_seconds=0.001,
        max_events=2,
        max_keepalives=2,
        max_total_bytes=256,
    )
    frames = [await anext(stream), await anext(stream)]
    await stream.aclose()
    return frames, source.cancelled, source.closed


frames, pending_cancelled, source_closed = asyncio.run(collect_example())

assert (frames, pending_cancelled, source_closed) == (
    [b'data: {"progress":25,"status":"running"}\n\n', b": keepalive\n\n"],
    True,
    True,
)
```

## Trade-offs and Limitations

The keepalive budget bounds emitted idle frames and initiates cancellation,
while the event count and byte budget bound a busy source. Completion is not
wall-clock bounded if the source suppresses `CancelledError`; use only a source
that honors the cooperative-cancellation contract. A slow consumer still
applies backpressure because the generator does not request another frame until
iteration resumes. The source's `aclose()` method and cancellation cleanup
should be idempotent; if iteration and cleanup both fail, both causes are
preserved in an exception group.

This helper formats only event-stream bytes. It does not set `Content-Type`,
disable caching, flush a framework response, detect a disconnected peer,
control proxy buffering, implement `Last-Event-ID`, or schedule application
completion events. Comment keepalives consume bandwidth but do not guarantee
that every intermediary or client will keep a connection open.

## Related Snippets

<!-- catalog:related:start -->
- [Encode a Bounded HTTP/1.1 Chunked Body for Protocol Tests](encode-a-bounded-http-1-1-chunked-body-for-protocol-tests.md)
- [Propagate a Monotonic Deadline with ContextVar](../reliability-resilience/propagate-a-monotonic-deadline-with-contextvar.md)
- [Limit Text Lines Across Arbitrary Chunks](../data-processing/limit-text-lines-across-arbitrary-chunks.md)
<!-- catalog:related:end -->
