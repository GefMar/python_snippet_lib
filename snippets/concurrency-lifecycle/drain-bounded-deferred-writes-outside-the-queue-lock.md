---
title: "Drain Bounded Deferred Writes Outside the Queue Lock"
snippet_type: pattern
use_cases:
  - concurrency-control
  - networking
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - ../networking-protocols/read-and-write-size-capped-varint-frames.md
  - ../observability-operations/process-log-records-in-a-background-thread-with-queuelistener.md
  - ../testing-tooling/wait-for-named-queue-conditions-under-one-monotonic-deadline.md
---

# Drain Bounded Deferred Writes Outside the Queue Lock

## Idea and Problem

Queue response bytes under a short lock, then perform potentially blocking writes without holding that queue lock.

Protocol parsing and other state transitions sometimes produce bytes that must
be written later. A bounded FIFO separates that bookkeeping from I/O, while a
second lock serializes drainers so frames keep their successful enqueue order.
The attempted frame stays counted as pending during its write, preventing a
slow sink from creating unaccounted capacity.

## When to Use

Use this pump when state changes synchronously produce complete immutable byte
frames, every outgoing frame follows the same FIFO, and the transport exposes a
`write_all(bytes)` operation. Call `defer()` while the owning parser or state
lock establishes output order, then call `drain()` only after releasing that
external lock. Treat any writer failure as terminal for the connection.

## Implementation

```python
from collections import deque
from collections.abc import Callable, Iterable
from dataclasses import dataclass
from threading import Lock


_MAX_MANAGED_FRAMES = 1024
_MAX_MANAGED_BYTES = 16 * 1024 * 1024


class PendingWriteCapacityError(BufferError):
    pass


class DeferredWriterFailed(RuntimeError):
    pass


@dataclass(slots=True)
class _PendingFrame:
    data: bytes
    attempted: bool = False


def _bounded_positive_int(value: int, *, name: str, maximum: int) -> int:
    if isinstance(value, bool) or not isinstance(value, int):
        raise TypeError(f"{name} must be an integer")
    if not 1 <= value <= maximum:
        raise ValueError(f"{name} is outside the supported range")
    return value


class DeferredByteWriter:
    def __init__(
        self,
        write_all: Callable[[bytes], None],
        *,
        max_pending_frames: int = 256,
        max_pending_bytes: int = 1024 * 1024,
    ) -> None:
        if not callable(write_all):
            raise TypeError("write_all must be callable")
        self._write_all = write_all
        self._max_pending_frames = _bounded_positive_int(
            max_pending_frames,
            name="max_pending_frames",
            maximum=_MAX_MANAGED_FRAMES,
        )
        self._max_pending_bytes = _bounded_positive_int(
            max_pending_bytes,
            name="max_pending_bytes",
            maximum=_MAX_MANAGED_BYTES,
        )
        self._pending: deque[_PendingFrame] = deque()
        self._failed = False
        self._queue_lock = Lock()
        self._drain_lock = Lock()

    @property
    def pending_count(self) -> int:
        with self._queue_lock:
            return len(self._pending)

    @property
    def pending_bytes(self) -> int:
        with self._queue_lock:
            return sum(len(frame.data) for frame in self._pending)

    @property
    def failed(self) -> bool:
        with self._queue_lock:
            return self._failed

    def defer(self, frames: Iterable[bytes]) -> None:
        if isinstance(frames, (bytes, bytearray, memoryview)):
            raise TypeError("frames must be an iterable of bytes objects")

        batch: list[bytes] = []
        batch_bytes = 0
        for frame in frames:
            if not isinstance(frame, bytes):
                raise TypeError("each frame must be immutable bytes")
            if len(batch) >= self._max_pending_frames:
                raise PendingWriteCapacityError("frame batch is too large")
            batch.append(frame)
            batch_bytes += len(frame)
            if batch_bytes > self._max_pending_bytes:
                raise PendingWriteCapacityError("frame batch is too large")

        with self._queue_lock:
            if self._failed:
                raise DeferredWriterFailed("writer is in a terminal state")
            if len(self._pending) + len(batch) > self._max_pending_frames:
                raise PendingWriteCapacityError("pending frame limit exceeded")
            pending_bytes = sum(len(frame.data) for frame in self._pending)
            if pending_bytes + batch_bytes > self._max_pending_bytes:
                raise PendingWriteCapacityError("pending byte limit exceeded")
            self._pending.extend(_PendingFrame(frame) for frame in batch)

    def drain(self, *, max_frames: int = 64) -> int:
        frame_budget = _bounded_positive_int(
            max_frames,
            name="max_frames",
            maximum=_MAX_MANAGED_FRAMES,
        )
        drained = 0
        with self._drain_lock:
            while drained < frame_budget:
                with self._queue_lock:
                    if self._failed:
                        raise DeferredWriterFailed(
                            "writer is in a terminal state"
                        )
                    if not self._pending:
                        return drained
                    pending_frame = self._pending[0]
                    if pending_frame.attempted:
                        self._pending.popleft()
                        self._failed = True
                        abandoned_attempt = True
                    else:
                        pending_frame.attempted = True
                        abandoned_attempt = False

                if abandoned_attempt:
                    raise DeferredWriterFailed(
                        "an interrupted write left an uncertain result"
                    )

                try:
                    self._write_all(pending_frame.data)
                    with self._queue_lock:
                        if (
                            self._pending
                            and self._pending[0] is pending_frame
                        ):
                            self._pending.popleft()
                except BaseException:
                    with self._queue_lock:
                        if (
                            self._pending
                            and self._pending[0] is pending_frame
                        ):
                            self._pending.popleft()
                        self._failed = True
                    raise
                drained += 1
        return drained
```

## Example

```python
state_lock = Lock()
written: list[bytes] = []


def write_all(frame: bytes) -> None:
    assert not state_lock.locked()
    written.append(frame)


writer = DeferredByteWriter(
    write_all,
    max_pending_frames=4,
    max_pending_bytes=64,
)

with state_lock:
    writer.defer((b"reply-one", b"reply-two"))

first_drain = writer.drain(max_frames=1)
second_drain = writer.drain(max_frames=1)

assert (
    first_drain,
    second_drain,
    written,
    writer.pending_count,
    writer.pending_bytes,
    writer.failed,
) == (1, 1, [b"reply-one", b"reply-two"], 0, 0, False)
```

## Trade-offs and Limitations

The FIFO order is the order in which `defer()` calls successfully append under
the internal lock. Callers must use one external state lock if protocol
transitions themselves require a stronger order, and all application and
control bytes must use the same pump. The count and byte limits provide
backpressure by raising; they do not wait for capacity.

Byte accounting scans the bounded queue, including the frame currently being
written, so an interrupted cleanup cannot leave a separate counter out of
sync. The queue is intentionally capped at 1,024 frames, which bounds that
scan.

`write_all` must either write the complete frame or raise. An attempted frame
is removed even when the callback raises, because retrying after an unknown
partial write could duplicate bytes. Later frames remain queued, but the pump
becomes terminal and will not send them. The caller owns transport timeout,
close, and reconnect policy. The per-call frame budget bounds one drain, not
the time spent inside an individual blocking write. The write callback may
enqueue later frames, but it must not re-enter `drain()`.

## Related Snippets

<!-- catalog:related:start -->
- [Read and Write Size-Capped Varint Frames](../networking-protocols/read-and-write-size-capped-varint-frames.md)
- [Process Log Records in a Background Thread with QueueListener](../observability-operations/process-log-records-in-a-background-thread-with-queuelistener.md)
- [Wait for Named Queue Conditions Under One Monotonic Deadline](../testing-tooling/wait-for-named-queue-conditions-under-one-monotonic-deadline.md)
<!-- catalog:related:end -->
