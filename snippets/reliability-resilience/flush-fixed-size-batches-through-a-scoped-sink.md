---
title: "Flush Fixed-Size Batches Through a Scoped Sink"
snippet_type: pattern
use_cases:
  - data-transformation
  - resource-management
  - retry-recovery
tested_python:
  - "3.14"
dependencies: []
related:
  - commit-a-source-checkpoint-only-after-the-sink-accepts-a-batch.md
  - retry-only-eligible-items-in-a-bounded-batch.md
  - ../data-processing/batch-items-by-estimated-byte-size.md
---

# Flush Fixed-Size Batches Through a Scoped Sink

## Idea and Problem

Collect pushed items into fixed-size immutable batches and flush the final partial batch only when the surrounding operation exits successfully.

A one-use context manager makes the lifecycle explicit: items are accepted only
while its scope is active, full batches are sent immediately, and a clean exit
sends the tail. If the body fails, the unsent tail is abandoned. If the sink
raises, the attempted batch stays available for reconciliation and the writer
enters a terminal failed state instead of guessing whether a retry is safe.

## When to Use

Use this pattern when a synchronous operation naturally pushes items one at a
time, the sink accepts a complete tuple, and batching by item count is the
correct boundary. Treat a normal sink return as acknowledgement, and choose a
batch size that keeps both the tuple and the sink operation comfortably
bounded.

Use a transaction, durable queue, or idempotency protocol when acceptance must
survive process failure. Use byte-aware batching when item sizes vary enough
that a count limit does not bound the actual payload.

## Implementation

```python
from collections.abc import Callable
from enum import Enum, auto
from typing import Generic, Self, TypeVar


ItemT = TypeVar("ItemT")
_MAX_BATCH_SIZE = 10_000


class BatchWriterState(Enum):
    NEW = auto()
    ACTIVE = auto()
    CLOSED = auto()
    ABORTED = auto()
    FAILED = auto()


class BatchWriterStateError(RuntimeError):
    pass


class ReentrantSinkCall(RuntimeError):
    pass


class ScopedBatchWriter(Generic[ItemT]):
    def __init__(
        self,
        sink: Callable[
            [tuple[ItemT, ...]],
            None,
        ],
        *,
        batch_size: int,
    ) -> None:
        if not callable(sink):
            raise TypeError("sink must be callable")
        if isinstance(batch_size, bool) or not isinstance(batch_size, int):
            raise TypeError("batch_size must be an integer")
        if not 1 <= batch_size <= _MAX_BATCH_SIZE:
            raise ValueError("batch_size is outside the supported range")

        self._sink = sink
        self._batch_size = batch_size
        self._state = BatchWriterState.NEW
        self._buffer: list[ItemT] = []
        self._acknowledged_batches = 0
        self._acknowledged_items = 0
        self._failed_batch: tuple[ItemT, ...] | None = None
        self._abandoned_tail: tuple[ItemT, ...] = ()
        self._in_sink = False
        self._reentrant_attempted = False

    @property
    def state(self) -> BatchWriterState:
        return self._state

    @property
    def acknowledged_batches(self) -> int:
        return self._acknowledged_batches

    @property
    def acknowledged_items(self) -> int:
        return self._acknowledged_items

    @property
    def failed_batch(self) -> tuple[ItemT, ...] | None:
        return self._failed_batch

    @property
    def abandoned_tail(self) -> tuple[ItemT, ...]:
        return self._abandoned_tail

    def _reject_reentrant_call(self) -> None:
        if self._in_sink:
            self._reentrant_attempted = True
            raise ReentrantSinkCall("the sink must not re-enter the writer")

    def __enter__(self) -> Self:
        self._reject_reentrant_call()
        if self._state is not BatchWriterState.NEW:
            raise BatchWriterStateError("the writer can be entered only once")
        self._state = BatchWriterState.ACTIVE
        return self

    def add(self, item: ItemT) -> None:
        self._reject_reentrant_call()
        if self._state is not BatchWriterState.ACTIVE:
            raise BatchWriterStateError("items require an active writer")
        self._buffer.append(item)
        if len(self._buffer) == self._batch_size:
            self._flush()

    def _flush(self) -> None:
        batch = tuple(self._buffer)
        if not batch:
            return

        self._in_sink = True
        self._reentrant_attempted = False
        try:
            self._sink(batch)
            if self._reentrant_attempted:
                raise ReentrantSinkCall("the sink attempted to re-enter the writer")
        except BaseException:
            self._failed_batch = batch
            self._state = BatchWriterState.FAILED
            raise
        finally:
            self._in_sink = False

        self._buffer.clear()
        self._acknowledged_batches += 1
        self._acknowledged_items += len(batch)

    def __exit__(self, exc_type, exc_value, traceback) -> bool:
        self._reject_reentrant_call()
        if self._state is BatchWriterState.FAILED:
            if exc_type is None:
                raise BatchWriterStateError("the sink previously failed")
            return False
        if self._state is not BatchWriterState.ACTIVE:
            raise BatchWriterStateError("the writer is not active")

        if exc_type is not None:
            self._abandoned_tail = tuple(self._buffer)
            self._buffer.clear()
            self._state = BatchWriterState.ABORTED
            return False

        self._flush()
        self._state = BatchWriterState.CLOSED
        return False
```

## Example

```python
accepted: list[tuple[str, ...]] = []
with ScopedBatchWriter(accepted.append, batch_size=3) as writer:
    for item in ("a", "b", "c", "d"):
        writer.add(item)

assert (
    accepted,
    writer.state,
    writer.acknowledged_batches,
    writer.acknowledged_items,
) == (
    [("a", "b", "c"), ("d",)],
    BatchWriterState.CLOSED,
    2,
    4,
)

attempts: list[tuple[str, ...]] = []


def fail_second_batch(batch: tuple[str, ...]) -> None:
    attempts.append(batch)
    if batch == ("c", "d"):
        raise OSError("acknowledgement was not received")


failed = ScopedBatchWriter(fail_second_batch, batch_size=2)
try:
    with failed:
        for item in ("a", "b", "c", "d"):
            failed.add(item)
except OSError:
    sink_error_propagated = True
else:
    sink_error_propagated = False

aborted = ScopedBatchWriter(accepted.append, batch_size=3)
try:
    with aborted:
        aborted.add("unsent")
        raise LookupError("body failed")
except LookupError:
    body_error_propagated = True
else:
    body_error_propagated = False

holder = {}


def reentrant_sink(batch: tuple[str, ...]) -> None:
    try:
        holder["writer"].add("nested")
    except ReentrantSinkCall:
        pass


reentrant = ScopedBatchWriter(reentrant_sink, batch_size=1)
holder["writer"] = reentrant
try:
    with reentrant:
        reentrant.add("outer")
except ReentrantSinkCall:
    reentrant_call_rejected = True
else:
    reentrant_call_rejected = False

assert (
    attempts,
    sink_error_propagated,
    failed.state,
    failed.failed_batch,
    failed.acknowledged_batches,
    failed.acknowledged_items,
    body_error_propagated,
    aborted.state,
    aborted.abandoned_tail,
    reentrant_call_rejected,
    reentrant.state,
    reentrant.failed_batch,
) == (
    [("a", "b"), ("c", "d")],
    True,
    BatchWriterState.FAILED,
    ("c", "d"),
    1,
    2,
    True,
    BatchWriterState.ABORTED,
    ("unsent",),
    True,
    BatchWriterState.FAILED,
    ("outer",),
)
```

## Trade-offs and Limitations

The class is synchronous, deliberately one-use, and not thread-safe. The tuple
prevents the sink from changing batch membership or order, but it does not make
mutable items deeply immutable. A body exception abandons only the current
tail; earlier acknowledged batches remain external side effects and are not
rolled back.

A normal callback return is the only acknowledgement signal. If the callback
applies some or all of a batch and then raises, its outcome is ambiguous. The
writer retains that exact attempted tuple, counts only earlier acknowledged
batches, and never retries it. The caller must reconcile the retained batch or
use stable item identities and an idempotent sink before constructing a new
writer. Catching a sink exception inside the `with` body does not restore the
writer; a clean exit from that failed state raises another state error.

The item-count ceiling bounds only the buffer length. It does not bound item
sizes, callback duration, or external transaction size, and it provides no
asynchronous backpressure. A reentrant callback makes the entire attempt fail
even if the callback catches the immediate rejection.

## Related Snippets

<!-- catalog:related:start -->
- [Commit a Source Checkpoint Only After the Sink Accepts a Batch](commit-a-source-checkpoint-only-after-the-sink-accepts-a-batch.md)
- [Retry Only Eligible Items in a Bounded Batch](retry-only-eligible-items-in-a-bounded-batch.md)
- [Batch Items by Estimated Byte Size](../data-processing/batch-items-by-estimated-byte-size.md)
<!-- catalog:related:end -->
